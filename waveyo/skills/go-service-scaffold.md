# Go Service Scaffold — WaveYo 风格

> 触发条件：新建 Go 后端项目时加载。
> 来源证据：YoBFF, Koishi-Mirror, YoOSF-API 的目录结构和启动模式。

---

## 项目布局

```
project/
├── main.go              # 入口（配置加载 + 依赖注入 + 启动）
├── cmd/                 # 子命令（可选）
├── config/              # 配置结构体与默认值
│   └── config.go
├── internal/            # 核心业务（不可外部导入）
│   ├── handler/         # HTTP handler（薄层，只做参数绑定和响应）
│   ├── service/         # 业务逻辑
│   ├── repository/      # 数据访问（DB / Redis / S3）
│   ├── middleware/      # 中间件（auth / logging / recovery）
│   └── model/           # 数据模型
├── api/                 # 路由注册
│   └── router.go
├── contracts/           # OpenAPI 契约（可选）
├── web/                 # 内嵌前端（可选）
├── Dockerfile
├── docker-compose.yml
├── systemctl-files/     # systemd unit 文件
├── .env.example
├── go.mod
└── README.md
```

---

## 启动流程（main.go）

```go
package main

import (
    "context"
    "os"
    "os/signal"
    "syscall"
    "time"

    "go.uber.org/zap"
)

func main() {
    // 1. 加载配置
    cfg := config.Load()  // Viper + env

    // 2. 初始化日志
    logger, _ := zap.NewProduction()
    defer logger.Sync()

    // 3. 初始化依赖
    db, _ := initDB(cfg)
    redis := initRedis(cfg)
    s3 := initS3(cfg)

    // 4. 初始化仓库/服务/handler 层
    repo := repository.New(db, redis, s3)
    svc := service.New(repo, logger)
    handler := handler.New(svc, logger)

    // 5. 注册路由
    router := api.Setup(handler, cfg)

    // 6. 启动服务器（带优雅关闭）
    srv := &http.Server{Addr: cfg.Addr, Handler: router}
    go func() {
        logger.Info("server started", zap.String("addr", cfg.Addr))
        if err := srv.ListenAndServe(); err != http.ErrServerClosed {
            logger.Fatal("server error", zap.Error(err))
        }
    }()

    // 7. 等待信号
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit

    // 8. 优雅关闭
    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()
    srv.Shutdown(ctx)
    logger.Info("server stopped")
}
```

---

## 配置管理

```go
// config/config.go
type Config struct {
    Addr      string `mapstructure:"ADDR"`
    DB        DBConfig
    Redis     RedisConfig
    S3        S3Config
}

func Load() *Config {
    viper.SetDefault("ADDR", ":8080")
    viper.SetEnvKeyReplacer(strings.NewReplacer(".", "_"))
    viper.AutomaticEnv()

    viper.SetConfigName("config")
    viper.SetConfigType("yaml")
    viper.AddConfigPath(".")
    viper.ReadInConfig()

    var cfg Config
    viper.Unmarshal(&cfg)
    return &cfg
}
```

.env.example：
```
ADDR=:8080
DB_HOST=localhost
DB_PORT=5432
DB_USER=app
DB_PASSWORD=change_me
DB_NAME=myapp
REDIS_ADDR=localhost:6379
S3_ENDPOINT=s3.example.com
S3_BUCKET=my-bucket
```

---

## 健康检查（必须）

```go
// GET /health
func HealthHandler(db *sql.DB, redis *redis.Client) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        deps := map[string]string{}
        if err := db.Ping(); err != nil {
            deps["db"] = "error: " + err.Error()
        } else {
            deps["db"] = "ok"
        }
        if err := redis.Ping().Err(); err != nil {
            deps["redis"] = "error: " + err.Error()
        } else {
            deps["redis"] = "ok"
        }
        status := "ok"
        for _, v := range deps {
            if v != "ok" {
                status = "degraded"
            }
        }
        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode(map[string]interface{}{
            "status": status,
            "deps":   deps,
        })
    }
}
```

---

## 日志初始化（Zap — 5 级体系）

```go
import (
    "strings"
    "go.uber.org/zap"
    "go.uber.org/zap/zapcore"
)

// 自定义 TRACE 级别（Zap 原生最低为 Debug=-1，TRACE 注册为 -2）
const TraceLevel = zapcore.Level(-2)

func initLogger(levelStr string) *zap.Logger {
    cfg := zap.NewProductionConfig()
    cfg.EncoderConfig.TimeKey = "timestamp"
    cfg.EncoderConfig.EncodeTime = zapcore.ISO8601TimeEncoder

    switch strings.ToLower(levelStr) {
    case "trace":
        cfg.Level = zap.NewAtomicLevelAt(TraceLevel)
    case "debug":
        cfg.Level = zap.NewAtomicLevelAt(zap.DebugLevel)
    case "warn":
        cfg.Level = zap.NewAtomicLevelAt(zap.WarnLevel)
    case "error":
        cfg.Level = zap.NewAtomicLevelAt(zap.ErrorLevel)
    default:
        cfg.Level = zap.NewAtomicLevelAt(zap.InfoLevel)
    }

    logger, _ := cfg.Build()
    return logger
}

// initLoggerFromEnv 从环境变量读取日志级别，默认 info
func initLoggerFromEnv() *zap.Logger {
    level := os.Getenv("LOG_LEVEL")
    if level == "" {
        level = "info"
    }
    return initLogger(level)
}

// logTrace TRACE 级别日志辅助函数（Zap 无原生 Trace，用 Check 实现）
func logTrace(logger *zap.Logger, msg string, fields ...zap.Field) {
    if logger.Core().Enabled(TraceLevel) {
        logger.Check(TraceLevel, msg).Write(fields...)
    }
}
```

使用示例：

```go
// INFO — 请求完成
logger.Info("request completed",
    zap.String("method", r.Method),
    zap.String("path", r.URL.Path),
    zap.Int("status", status),
    zap.Duration("latency", latency),
    zap.String("request_id", requestID),
)

// TRACE — 缓存逐 key 命中（仅开发环境输出）
logTrace(logger, "cache lookup",
    zap.String("key", key),
    zap.Bool("hit", ok),
    zap.String("layer", "l1"),
)

// WARN — 重试
logger.Warn("retrying request",
    zap.String("url", url),
    zap.Int("attempt", attempt),
    zap.Error(err),
)

// ERROR — 依赖故障
logger.Error("redis connection failed",
    zap.String("addr", cfg.Redis.Addr),
    zap.Error(err),
)
```

---

## 错误处理

### AppError 自定义错误类型

```go
type AppError struct {
    Code    string      `json:"code"`     // VALIDATION, NOT_FOUND, AUTH, TIMEOUT, INTERNAL
    Message string      `json:"message"`  // 人类可读消息
    Details interface{} `json:"details,omitempty"`
    Err     error       `json:"-"`        // 原始错误（不序列化）
}

func (e *AppError) Error() string {
    if e.Err != nil {
        return fmt.Sprintf("[%s] %s: %v", e.Code, e.Message, e.Err)
    }
    return fmt.Sprintf("[%s] %s", e.Code, e.Message)
}

func (e *AppError) Unwrap() error { return e.Err }

// 错误构造函数
func NewValidationError(msg string, err error) *AppError {
    return &AppError{Code: "VALIDATION", Message: msg, Err: err}
}
func NewNotFoundError(msg string, err error) *AppError {
    return &AppError{Code: "NOT_FOUND", Message: msg, Err: err}
}
func NewAuthError(msg string, err error) *AppError {
    return &AppError{Code: "AUTH", Message: msg, Err: err}
}
func NewTimeoutError(msg string, err error) *AppError {
    return &AppError{Code: "TIMEOUT", Message: msg, Err: err}
}
func NewInternalError(msg string, err error) *AppError {
    return &AppError{Code: "INTERNAL", Message: msg, Err: err}
}
```

### 错误-HTTP 状态码映射

```go
var codeToStatus = map[string]int{
    "VALIDATION": 400,
    "NOT_FOUND":  404,
    "AUTH":       401,
    "TIMEOUT":    408,
    "INTERNAL":   500,
}
```

### 错误中间件（Fiber 示例）

```go
func ErrorMiddleware(logger *zap.Logger) fiber.Handler {
    return func(c *fiber.Ctx) error {
        err := c.Next()
        if err == nil {
            return nil
        }

        var appErr *AppError
        if errors.As(err, &appErr) {
            status := codeToStatus[appErr.Code]
            // 按错误码映射日志级别
            switch appErr.Code {
            case "VALIDATION":
                logger.Debug("validation error", zap.String("path", c.Path()), zap.Error(appErr))
            case "NOT_FOUND":
                logger.Warn("resource not found", zap.String("path", c.Path()), zap.Error(appErr))
            case "AUTH":
                logger.Warn("auth failed", zap.String("path", c.Path()), zap.Error(appErr))
            case "TIMEOUT", "INTERNAL":
                logger.Error("server error", zap.String("path", c.Path()), zap.Error(appErr))
            }
            return c.Status(status).JSON(fiber.Map{
                "code":       appErr.Code,
                "message":    appErr.Message,
                "details":    appErr.Details,
                "request_id": c.Locals("request_id"),
            })
        }

        // 未知错误 → 500
        logger.Error("unhandled error", zap.String("path", c.Path()), zap.Error(err))
        return c.Status(500).JSON(fiber.Map{
            "code":       "INTERNAL",
            "message":    "internal server error",
            "request_id": c.Locals("request_id"),
        })
    }
}
```

### Recovery 中间件

```go
func RecoveryMiddleware(logger *zap.Logger) fiber.Handler {
    return func(c *fiber.Ctx) error {
        defer func() {
            if r := recover(); r != nil {
                logger.Error("panic recovered",
                    zap.String("path", c.Path()),
                    zap.Any("panic", r),
                    zap.String("stack", string(debug.Stack())),
                )
                c.Status(500).JSON(fiber.Map{
                    "code":    "INTERNAL",
                    "message": "internal server error",
                })
            }
        }()
        return c.Next()
    }
}
```

---

## 注意事项

- 不要在 `handler` 层写业务逻辑，handler 只做参数绑定、调用 service、返回响应
- 错误必须向上传递（wrap + return），使用 AppError 分类，不使用 panic（仅 middleware 层 recover）
- 所有 DB/Redis/S3 连接在 main 中初始化，通过依赖注入传递
- 单元测试放在对应包的 `*_test.go` 中，覆盖核心路径
- `.env.example` 在首次提交时必须存在
- README 必须包含：架构说明 + 接口列表 + 部署步骤 + 压测数据（如有）

> 来源：WaveYo / YoBFF, Koishi-Mirror, YoOSF-API 的项目结构。
