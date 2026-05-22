# Gateway / BFF Pattern — WaveYo 风格

> 触发条件：需要设计网关或 BFF 层时加载。
> 来源证据：YoBFF 的控制面/数据面分离 + Host 路由 + 审计日志模式。

---

## 架构：控制面 / 数据面分离

```
                    ┌─────────────────────┐
                    │     管理控制面       │
                    │  (Admin API)         │
                    │  - 配置管理          │
                    │  - 证书管理          │
                    │  - 审计日志          │
                    │  - 用户管理          │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │     数据面          │
                    │  (Gateway / BFF)    │
                    │  - Host 路由         │
                    │  - 负载均衡          │
                    │  - Bearer Token     │
                    │  - 限流 / 验证码     │
                    └─────────────────────┘
```

---

## 数据面核心：Host 路由

```go
type Gateway struct {
    routes   map[string]*Route  // host → Route
    mu       sync.RWMutex
    logger   *zap.Logger
}

type Route struct {
    Host        string
    Backends    []string        // 后端服务地址列表
    lbIndex     uint32          // 轮询索引（原子操作）
    TLS         *TLSConfig
    Auth        AuthMethod      // none / bearer / basic
    Middlewares []Middleware
}

func (g *Gateway) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    host := r.Host

    g.mu.RLock()
    route, ok := g.routes[host]
    g.mu.RUnlock()

    if !ok {
        http.Error(w, "host not found", 404)
        return
    }

    // 认证
    if err := route.authenticate(r); err != nil {
        g.logger.Warn("auth failed", zap.String("host", host), zap.Error(err))
        http.Error(w, "unauthorized", 401)
        return
    }

    // 选择后端（轮询）
    backend := route.nextBackend()

    // 反向代理
    proxy := httputil.NewSingleHostReverseProxy(backend)
    proxy.ServeHTTP(w, r)

    // 记录审计
    g.auditLog(host, backend.String(), r)
}
```

---

## 负载均衡（轮询 + 原子操作）

```go
func (r *Route) nextBackend() *url.URL {
    idx := atomic.AddUint32(&r.lbIndex, 1)
    backend := r.Backends[idx%uint32(len(r.Backends))]
    u, _ := url.Parse(backend)
    return u
}
```

---

## 配置管理与回滚

```go
type ConfigManager struct {
    current   *Config
    history   []*ConfigVersion  // 配置历史，支持回滚
    mu        sync.RWMutex
}

type ConfigVersion struct {
    Version   int
    Config    *Config
    CreatedAt time.Time
    Comment   string
}

func (cm *ConfigManager) Apply(newCfg *Config, comment string) error {
    cm.mu.Lock()
    defer cm.mu.Unlock()

    version := ConfigVersion{
        Version:   cm.current.Version + 1,
        Config:    newCfg,
        CreatedAt: time.Now(),
        Comment:   comment,
    }
    cm.history = append(cm.history, &version)
    cm.current = newCfg
    return nil
}

func (cm *ConfigManager) Rollback(version int) error {
    cm.mu.Lock()
    defer cm.mu.Unlock()

    for _, v := range cm.history {
        if v.Version == version {
            cm.current = v.Config
            return nil
        }
    }
    return fmt.Errorf("version %d not found", version)
}
```

---

## 审计日志

```go
type AuditEntry struct {
    Timestamp  time.Time     `json:"timestamp"`
    Level      string        `json:"level"`               // info / warn / error
    Host       string        `json:"host"`
    Backend    string        `json:"backend"`
    Method     string        `json:"method"`
    Path       string        `json:"path"`
    Status     int           `json:"status"`
    Latency    time.Duration `json:"latency"`
    ClientIP   string        `json:"client_ip"`
    AuthUser   string        `json:"auth_user,omitempty"`
}

func (g *Gateway) auditLog(host, backend string, r *http.Request, status int) {
    // 按响应状态码映射日志级别
    level := "info"
    if status >= 500 {
        level = "error"
    } else if status >= 400 {
        level = "warn"
    }

    entry := AuditEntry{
        Timestamp: time.Now(),
        Level:     level,
        Host:      host,
        Backend:   backend,
        Method:    r.Method,
        Path:      r.URL.Path,
        Status:    status,
        ClientIP:  r.RemoteAddr,
    }
    g.logger.Info("gateway_request",
        zap.String("level", entry.Level),
        zap.String("host", entry.Host),
        zap.String("backend", entry.Backend),
        zap.String("method", entry.Method),
        zap.String("path", entry.Path),
        zap.Int("status", status),
    )
}
```

---

## 错误响应标准

```go
type ErrorResponse struct {
    Code      string      `json:"code"`
    Message   string      `json:"message"`
    Details   interface{} `json:"details,omitempty"`
    RequestID string      `json:"request_id"`
}

// 网关错误码
const (
    ErrUpstreamTimeout    = "UPSTREAM_TIMEOUT"
    ErrUpstreamUnavail    = "UPSTREAM_UNAVAILABLE"
    ErrAuthFailed         = "AUTH_FAILED"
    ErrRouteNotFound      = "ROUTE_NOT_FOUND"
)

// 网关错误-HTTP 状态码映射
var gatewayCodeToStatus = map[string]int{
    ErrUpstreamTimeout: 504,
    ErrUpstreamUnavail: 502,
    ErrAuthFailed:      401,
    ErrRouteNotFound:   404,
}
```

网关错误-日志级别映射：

| 错误码 | 日志级别 | 原因 |
|--------|---------|------|
| `ROUTE_NOT_FOUND` | DEBUG | 多为机器人扫描 |
| `AUTH_FAILED` | WARN | 认证失败 |
| `UPSTREAM_TIMEOUT` | ERROR | 上游超时 |
| `UPSTREAM_UNAVAILABLE` | ERROR | 上游不可达 |

---

## 验证码防爆破

```go
type CaptchaGuard struct {
    attempts  map[string]*attemptCounter
    mu        sync.Mutex
    threshold int
    window    time.Duration
}

func (g *CaptchaGuard) Check(identifier string) bool {
    g.mu.Lock()
    defer g.mu.Unlock()

    counter, ok := g.attempts[identifier]
    if !ok || time.Since(counter.windowStart) > g.window {
        g.attempts[identifier] = &attemptCounter{count: 1, windowStart: time.Now()}
        return false
    }
    counter.count++
    return counter.count >= g.threshold  // 触发验证码
}
```

---

## OpenAPI 契约

管理面 API 使用 OpenAPI 3.0 契约定义：

```yaml
# contracts/openapi/admin.yaml
openapi: 3.0.0
info:
  title: Admin API
  version: 1.0.0
paths:
  /api/v1/routes:
    get:
      summary: 获取路由列表
      responses:
        '200':
          description: 路由列表
  /api/v1/routes/{host}:
    put:
      summary: 更新路由配置
      parameters:
        - name: host
          in: path
          required: true
```

---

## 注意事项

- 控制面和数据面必须物理分离（不同端口或不同进程）
- 配置变更记录版本，支持回滚
- 审计日志记录所有数据面请求，但不记录敏感数据（如 Token 正文）
- Bearer Token 在请求头中传递，不在 URL 中
- 不把 Kubernetes / Consul / Etcd 服务发现写成已实现（当前为静态配置 + 文件 reload）
- README 中写已实现的能力列表，不写 planned 功能

> 来源：WaveYo / YoBFF 网关实现模式。只写 README 和源码中已实现能力。
