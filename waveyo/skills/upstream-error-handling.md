# 上游错误处理

> 触发时机：接入或改造 SDK、HTTP/RPC 调用、数据库驱动、对象存储、消息队列、第三方支付/短信等
> 外部依赖，或排查「上游没报错但结果不对」「报错了却不知道是谁的错」时必读。
>
> 主要示例使用 Go，**语言无关的处理原则同时适用于 Python / TypeScript / Rust**。

---

## 1. 用途与触发条件

### 用途

把「上游怎么失败的」这件事**完整、可诊断、且不泄露**地带到该到的地方：

- 原始原因保留在错误链里，供上层用 `errors.Is` / `errors.As`（或等价机制）判断；
- 结构化诊断字段（阶段、业务码、请求 ID、耗时）供日志与排障；
- 对外只暴露项目定义的安全语义，不透传上游原文。

一句话：**保留原始错误链，提取结构化诊断，对外映射安全错误。**

### 触发条件

- 新增或更换外部依赖适配层（adapter / client / repository）
- 处理「HTTP 200 但业务失败」这类结果
- 编写错误分类、错误码、错误中间件
- 排查超时、取消、重试、结果未知（幂等性）问题
- 用户反馈「提示不明确」「不知道要不要重试」「后台看不到记录」

### 不触发

纯内存计算、参数校验（本地即可判定）、UI 文案润色。

---

## 2. 适用边界

| 属于本专题 | 不属于本专题 |
|---|---|
| 错误链保留与包装 | 前端提示的排版与交互 → `frontend-ui-ux.md` |
| 失败类型分类（传输 / 业务 / 结构 / 未知） | 数据库表设计与事务边界 → 数据层文档 |
| 结构化诊断字段选择 | 日志级别选择细则 → `logging.md` |
| 取消与截止的来源判定 | 分支与 PR → `git-workflow.md` |
| 对外错误映射原则 | 具体项目的 HTTP 状态码约定 → 项目规范 |

**两条边界**：

1. 「保留原始错误」**不等于**「把原始错误全文返回给调用方」。
2. 本 skill 不强制所有服务使用同一套 HTTP 状态码或固定四字段响应体；对外契约由**各项目自己定义**，
   本专题只约束「分类清楚、原因保留、公开内容安全」。

---

## 3. 核心原则

```
原始错误（类型 + 身份 + 消息）
   └─ 每层只加有用的上下文（阶段），用 %w / Unwrap 链接
        └─ 适配层提取结构化诊断（provider / operation / code / request_id / 耗时）
             └─ 业务层映射为稳定语义（可重试？可对外说明？）
                  ├─ 日志：写诊断字段，不写敏感原文
                  └─ 对外：写安全文案，不透传上游 message
```

---

## 4. 核心规则

### 4.1 保留错误链

- **不要用固定错误替换原始原因**：`return ErrUploadFailed` 会让排障信息归零。
- **不要用 `errors.New(err.Error())` 重建错误**：类型、身份、`errors.Is` 全部丢失，
  这是最常见的「日志里有、代码里判不出来」的根因。
- **使用 `%w` 包装**，或自定义类型实现 `Unwrap()` / `Cause()`。
- **保证上层仍能识别底层原因**：`errors.Is(err, context.DeadlineExceeded)`、
  `errors.As(err, &apiErr)` 必须依旧成立。
- **每层只加有用上下文**：阶段名（`upload.put`）、关键标识（对象键前缀、订单号）。
  **不机械重复包装同一段文本**——`fmt.Errorf("failed: %w", fmt.Errorf("failed: %w", err))`
  只会把日志变成回声。

```go
// 只加阶段与可用标识，保留原始错误身份
return fmt.Errorf("storage.put: key=%s: %w", keyPrefix, err)

// 反例：原因被吞掉，类型消失
// return errors.New("上传失败")
// return fmt.Errorf("上传失败: %v", err)   // %v 同样截断错误链
```

### 4.2 区分失败类型

失败至少有五类，处理方式完全不同：

| 类型 | 典型表现 | 是否可重试 | 对外语义 |
|---|---|---|---|
| **传输失败** | 连接被拒、DNS 失败、连接重置 | 一般可（有界） | 依赖暂不可用 |
| **超时 / 截止** | `DeadlineExceeded`、客户端超时 | 谨慎：结果可能未知 | 处理超时 |
| **取消** | `Canceled`（用户离开、父请求结束） | 不重试 | 请求已取消 |
| **业务拒绝** | HTTP 200 + 业务码非成功；RPC 返回业务错误码 | 不重试（除非语义明确） | 业务规则拒绝 |
| **响应缺失 / 结构异常** | 空响应体、字段缺失、类型不符、解析失败 | 谨慎：结果未知 | 依赖响应异常 |
| **结果未知** | 超时且未拿到回执、写操作断连 | 需幂等才可重试 | 状态待确认 |

- **HTTP 200 不能直接当作业务成功**：需要检查业务码 / 响应体结构 / 关键字段。
- **不确定结果不能计为成功**，也**不能自动认定绝对未执行**。
  「没收到响应」≠「没有发生」，写操作尤其要按「未知」处理，交给幂等键或对账解决。

### 4.3 结构化诊断

按实际依赖提取，只取**真正存在**的字段：

| 字段 | 来源 | 用途 |
|---|---|---|
| `provider` | 配置/常量 | 区分阿里云 / AWS / 自建 |
| `operation` / `stage` | 调用点 | 定位是哪一次调用 |
| `biz_code` | SDK 错误字段或响应体 | 供应商业务语义 |
| `request_id` | 供应商返回头或错误对象 | 找供应商查单的凭证 |
| `http_status` / `protocol_code` | 传输层 | 区分 4xx/5xx/超时 |
| `elapsed` | 本地计时 | 判断是否触达超时阈值 |
| `retryable` | 由上述字段推导 | 只在**有证据**时置真 |

```go
// 适配层：把上游错误翻译成结构化诊断，同时保留原始错误链
type UpstreamError struct {
	Provider  string
	Operation string
	BizCode   string
	RequestID string
	HTTPCode  int
	Err       error
}

func (e *UpstreamError) Error() string {
	return fmt.Sprintf("upstream %s.%s failed (code=%s, http=%d)", e.Provider, e.Operation, e.BizCode, e.HTTPCode)
}

// Unwrap 让上层仍可用 errors.Is / errors.As 判断底层原因（超时、取消、连接错误）
func (e *UpstreamError) Unwrap() error { return e.Err }
```

**优先用已知类型、字段与 getter，避免字符串匹配**：

```go
var apiErr *oss.ServiceError          // 已知 SDK 错误类型
if errors.As(err, &apiErr) {
	diag.BizCode = apiErr.Code
	diag.RequestID = apiErr.RequestID
	diag.HTTPCode = apiErr.StatusCode
}

var urlErr *url.Error                 // 传输层
if errors.As(err, &urlErr) { /* 连接 / 超时 / TLS */ }

var netErr net.Error
if errors.As(err, &netErr) && netErr.Timeout() { /* 传输超时 */ }
```

- 兼容 SDK 的包装层与不同错误类型时，可写**薄适配函数**逐个类型尝试；
- **不滥用反射**，也不为兼容而造「万能错误解析框架」——那会变成没人敢改的黑盒。

### 4.4 取消与截止

- 判定来源要结合**父请求 context 与依赖调用 context**：
  - 父 context 已取消 → 归因「调用方取消」，不是上游故障；
  - 仅本次调用 context 到达 deadline → 才可能是上游慢/超时。
- **不能把 `context canceled` 一律归为上游超时**。
- **不能仅凭时长或堆栈推断**网络、权限、配置故障。缺证据就写「待验证」。

```go
// 区分取消来源：父请求已结束 vs 本次调用超时
select {
case <-parentCtx.Done():
	return fmt.Errorf("caller canceled: %w", parentCtx.Err())
default:
}
if errors.Is(err, context.DeadlineExceeded) {
	// 本次调用截止，视为上游超时/慢
}
```

### 4.5 业务层映射

- 提供**稳定的业务语义**，同时**继续保留内部原因**：对外是 `依赖暂不可用`，
  内部错误链里仍是原始 SDK 错误。
- **错误分类与公开消息分别管理**：分类用于分支与告警；公开消息用于调用方阅读，
  两者可以多对一，但不要混成一个字符串。
- **对外契约按具体项目定义**，不强制固定 HTTP 状态码或四字段格式。
  若项目已定义公开错误契约（例如固定 `code` / `message` / `data` / `request_id`），以项目契约为准并保持稳定。
- **不直接将供应商 `Message` 或 `err.Error()` 透传给用户**：其中可能含端点、账号片段、
  内部资源名甚至堆栈。

```go
// 分类 + 公开消息分离
type Kind string

const (
	KindUnavailable Kind = "DEPENDENCY_UNAVAILABLE"
	KindTimeout     Kind = "DEPENDENCY_TIMEOUT"
	KindRejected    Kind = "BUSINESS_REJECTED"
	KindUnknown     Kind = "RESULT_UNKNOWN"
)

// 对外文案表（与分类解耦，可独立演进）
var publicMessages = map[Kind]string{
	KindUnavailable: "依赖服务暂时不可用，请稍后重试",
	KindTimeout:     "处理超时，请稍后重试",
	KindRejected:    "请求未通过业务校验",
	KindUnknown:     "处理结果待确认，请勿重复提交",
}
```

### 4.6 日志与追踪

- **记录**：失败阶段、结构化分类、请求关联 ID（`request_id` / trace id）、依赖耗时。
- **请求结束日志里的栈 ≠ 原始失败栈**：框架在请求收尾时打的堆栈只是收尾位置，
  要定位根因必须靠错误链或 `%+v` 保留的原始栈。
- **原始错误保留在错误链，不意味着可以全文写入普通日志**：普通日志应记录诊断字段，
  需要原文时走受限的调试日志或受控工单通道。
- **不记录**：密钥、令牌、验证码、完整联系方式、请求正文（含身份证号、地址、手机号）。
- **原始消息可能含敏感内容**，记录前必须遵守项目脱敏与访问控制规则
  （例如只记长度、前缀、哈希）。
- **避免层层重复记录同一错误**：只在「处理边界」（HTTP handler、消息消费者入口、
  定时任务入口）记录一次；中间层只包装不打印。

```go
logging.Error("上游调用失败",
	zap.String("stage", diag.Operation),
	zap.String("provider", diag.Provider),
	zap.String("biz_code", diag.BizCode),
	zap.String("upstream_request_id", diag.RequestID),
	zap.Duration("elapsed", elapsed),
	zap.String("error", classify(err)), // 分类，不塞完整原文
)
```

### 4.7 诚实诊断

- **没有业务码时明确「信息不足」**，不能直接断言「SDK 有问题」「配置错了」「供应商挂了」。
- **供应商业务后台无记录 ≠ 请求从未到达**：权限、签名、配额、网关层的拒绝
  可能根本不形成业务记录。
- 结论要分三层写清：
  1. **已知事实**（状态码、错误字段、时间点）
  2. **推断**（最可能的原因，附依据）
  3. **待验证项**（下一步需要什么证据）

### 4.8 重试与副作用

- **只对明确允许重试的情况做有界重试**：连接失败、明确的 5xx/限流码，且有退避与上限。
- **写操作、扣费、发码、创建资源必须考虑幂等与结果未知**：没有幂等键就不要盲重试。
- **不通过无限重试或吞掉错误掩盖故障**：重试只会把故障拖长，并放大上游压力。
- 本节保持简洁：**不扩建无需求的重试系统**，没有证据就先不重试。

### 4.9 验证

必须覆盖的用例（可用替身，不依赖真实收费调用）：

1. `errors.Is` / `errors.As` 仍能识别原始原因（超时、取消、SDK 错误类型）。
2. 五类失败各一例：传输失败、业务拒绝（HTTP 200 + 失败码）、取消、超时、结构异常。
3. 公开响应与普通日志**不含**原始敏感消息（端点、账号、令牌、完整堆栈）。
4. 结果未知路径：超时后不误报成功，也不宣称绝对未执行。
5. 写操作重试：有幂等键时可重试，无幂等键时禁止重试。

**关于本地修复的可信度**：

- 若用本地 `replace` / 本地副本修复依赖，修复通过后**必须同步到可远程获取的依赖版本**，并更新版本锁
  （`go.mod` / `package-lock.json` / `pyproject` 约束）。
- **在无本地替换的干净环境再验证一次**（CI 或新容器），否则本地 SDK 会掩盖 CI 失败。
- **后端与前端共用同一 SDK 时**，检查两侧解析到的版本与产物是否一致（同版本号不同产物同样会出问题）。

---

## 5. 常见错误

| 反模式 | 后果 | 正确做法 |
|---|---|---|
| `errors.New(err.Error())` | 类型与身份丢失，无法 `Is/As` | `%w` 包装 |
| `fmt.Errorf("...: %v", err)` | 错误链截断 | `%w` |
| 每层都 `log.Error(err)` | 日志重复，噪声掩盖真问题 | 边界记录一次 |
| HTTP 200 即成功 | 业务失败被记为成功 | 检查业务码与响应结构 |
| 超时即判「未执行」 | 写操作重复、金额错乱 | 按「结果未知」处理，靠幂等/对账 |
| 把上游 message 直接返回前端 | 泄露端点/账号/内部资源名 | 分类映射 + 安全文案 |
| 把完整错误写普通日志 | 敏感信息落盘 | 只写诊断字段 |
| 无码也断言「供应商故障」 | 误导排障方向 | 写「信息不足 + 待验证项」 |
| 无幂等键无限重试 | 上游雪崩、重复扣费 | 有界重试 + 幂等键 |
| 本地 replace 修好就宣布完成 | CI/线上仍失败 | 同步依赖版本 + 干净环境复验 |

---

## 6. 验收清单

**错误链**

- [ ] 每层包装使用 `%w` 或实现 `Unwrap()` 的类型
- [ ] 无 `errors.New(err.Error())`、无 `%v` 包装丢链
- [ ] 上层 `errors.Is` / `errors.As` 能识别超时、取消、SDK 错误类型
- [ ] 每层只加阶段/标识，无重复文本套娃

**分类**

- [ ] 传输、超时、取消、业务拒绝、结构异常、结果未知均有明确判定分支
- [ ] HTTP 200 的业务码已校验
- [ ] 结果未知未被计为成功，也未宣称绝对未执行

**诊断**

- [ ] 结构化字段来自已知类型/getter，不靠字符串匹配
- [ ] 取消来源已区分（父请求取消 vs 本次超时）
- [ ] 无码场景写明「信息不足」与待验证项

**对外与日志**

- [ ] 对外只返回项目定义的公开字段，无上游原文
- [ ] 普通日志无令牌、密钥、验证码、完整联系方式、请求正文
- [ ] 同一错误只在边界记录一次
- [ ] 日志含 stage / provider / 分类 / request_id / 耗时

**重试与验证**

- [ ] 重试有上限与退避，且仅限明确可重试的类型
- [ ] 写操作有幂等键或明确不重试
- [ ] 五类失败用例已用替身覆盖
- [ ] 本地 replace 修复已同步依赖版本锁，并在干净环境复验
- [ ] 前后端共用 SDK 时版本与产物一致
- [ ] 未验证项已如实列出

---

## 7. 语言无关要点

| 语言 | 保留链的机制 | 身份判断 | 注意 |
|---|---|---|---|
| Go | `%w` / `Unwrap` | `errors.Is` / `errors.As` | 避免 `%v`、避免重建错误 |
| Python | `raise NewError(...) from err` | `except SpecificError` / `err.__cause__` | 不要 `raise X(str(e))` 无 `from` |
| TypeScript | `new Error(msg, { cause })` | `instanceof` / `err.cause` | 不要字符串化后重抛 |
| Rust | `thiserror` / `anyhow::Context` | `downcast_ref::<T>()` / 错误枚举 | 不要 `map_err(|_| MyErr)` 丢源 |

---

## 8. 参考

- Go `errors` 包（`Is` / `As` / `Unwrap`）：<https://pkg.go.dev/errors>
- Go `fmt` 中 `%w`：<https://pkg.go.dev/fmt#hdr-Printing>
- Go `context`（取消与截止）：<https://pkg.go.dev/context>
- Python `raise ... from`：<https://docs.python.org/3/reference/simple_stmts.html#the-raise-statement>
- `Error.cause`（TC39）：<https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Error/cause>
- Rust `thiserror` / `anyhow`：<https://docs.rs/thiserror> · <https://docs.rs/anyhow>

---

> 关联：`logging.md` 定义级别与敏感字段；`work.md`「错误处理」给出分类与集成位置；
> `persona.md` Layer 0 的「错误必须分类，日志必须分级」是本专题的上位原则。
