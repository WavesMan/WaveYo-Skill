# WaveYo — Work Skill

> 蒸馏自 25 个公开仓库 + 13 画像文件，基于代码证据驱动。
> 生成时间：2026-05-15 | 版本 1.2.0

---

## 职责范围

WaveYo 是一名以 Go 后端为主线，具备 Python 自动化工具、前端工程和运维交付能力的工程型开发者。负责以下技术领域：

- **Go 后端服务**：API 服务、对象存储镜像站、网关/BFF、系统网络工具
- **Python 自动化**：Linux 批量巡检、Windows 系统工具、本地 WebUI、CLI 工具
- **前端工程**：个人主页/博客、简历模板、管理面板、静态站、数据展示
- **运维交付**：服务部署、健康检查、日志/指标、批量运维自动化

---

## 技术规范

### Go 后端

**技术栈**：
- 框架：Fiber（首选）、Gin、标准库 net/http、Go-Zero（协作项目）
- 数据库：PostgreSQL、SQLite、Redis
- 存储：S3 / OSS / COS 对象存储
- 日志：Zap 结构化日志
- 配置：Viper + .env.example 环境变量覆盖
- 指标：Prometheus（Counter / Histogram）
- 认证：Bearer Token、Basic Auth、签名验证

**项目布局规范**：
```
project/
├── main.go              # 入口（配置加载 + 依赖注入 + 启动）
├── cmd/                 # 子命令入口
├── config/              # 配置结构体与默认值
├── internal/            # 核心业务（不对外暴露）
│   ├── handler/         # HTTP handler
│   ├── service/         # 业务逻辑
│   ├── repository/      # 数据访问
│   └── middleware/      # 中间件
├── api/                 # 路由注册
├── contracts/           # OpenAPI 契约
├── web/                 # 内嵌前端资源
├── Dockerfile
├── docker-compose.yml
├── .env.example
└── README.md
```

**缓存模式（核心特征）**：
- 多级缓存：本地 LRU/LFU → Redis → 源站
- 分片锁保护：按 key hash 分片，减少竞争
- 异步刷新：后台 goroutine + semaphore 控制并发
- 每层独立命中率指标
- 示例仓库：YoOSF-API（AcquireLock 模式）、Koishi-Mirror（Redis semaphore）

**代码风格**：
- 函数单一职责，避免超过 80 行的处理函数
- 错误必须向上传递（不使用 panic），每层 wrap context
- Handler → Service → Repository 分层调用，不跨层
- 使用 `context.Context` 传递请求级元数据
- 单元测试覆盖核心路径

### Python 自动化

**技术栈**：
- Python 3.11+，uv 包管理
- SSH：paramiko 或 subprocess
- 并发：ThreadPoolExecutor + Queue
- WebUI：Flask
- 报告：JSON / CSV / Markdown 多格式输出
- 打包：PyInstaller / Nuitka

**工作流程**：
```
配置解析 → 状态采集 → 风险分析 → 修复执行 → 报告输出
```

**核心模式**：
- audit 模式：只读检查，不修改目标
- fix 模式：执行修复操作，需显式 `--fix` 标志
- 进度心跳：长时间任务输出进度指示
- 失败分类：按主机/阶段分组错误并生成摘要

### 前端工程

**技术栈**：
- Vue 生态：Vue 3 + Vite + Pinia + Vue Router + Tailwind CSS
- React 生态：Next.js App Router + TypeScript + CSS Modules
- 数据：Axios、ECharts
- 内容：Markdown / frontmatter / GFM / 代码高亮（Shiki）
- 部署：静态导出 / Vercel / Cloudflare Pages

**组件拆分原则**：
- 业务组件与展示组件分离
- 模板与数据分离（如简历系统：`registry/` + `templates/` + `data/`）
- 可复用组件放 `components/`，页面级组件放对应路由目录

**来源边界声明**：
- `YoSpace`：README 声明 UI/UX 灵感源于 `KumaKorin/react-homepage`，个人化改造
- `Koishi-Registry`：基于上游项目自定义与优化
- fork 项目保留上游声明，不写成原创

### 运维交付

**部署规范**：
- Dockerfile（多阶段构建，alpine 基础镜像）
- docker-compose.yml（服务编排 + 依赖管理）
- systemd unit 文件（`systemctl-files/` 目录）
- 构建/发布脚本（`release-build.sh` / `release-build.ps1`）
- `.env.example` 提供配置模板

**健康检查模板**：
```go
// GET /health
// 返回 200 + JSON: {"status":"ok","deps":{"redis":"ok","db":"ok"}}
```

**日志规范**：

5 级日志体系（TRACE < DEBUG < INFO < WARN < ERROR）：

| 级别 | Zap 值 | 用途 | 环境 |
|------|--------|------|------|
| TRACE | -2（自定义） | 函数入口/出口、变量 dump、缓存逐 key 命中 | 开发环境 |
| DEBUG | -1 | 请求/响应 payload（脱敏）、SQL、Redis 命令、配置 dump | 开发环境 |
| INFO | 0 | 请求完成、服务启停、健康检查、缓存预热完成 | 生产最低 |
| WARN | 1 | 重试、降级触发、限流预警、非安全认证失败 | 全环境 |
| ERROR | 2 | 未处理错误、依赖故障、数据不一致、panic 恢复 | 全环境 |

- 结构化日志：Zap JSON 格式（Go）/ logging JSON 格式（Python）
- 标准字段：request_id、latency、status、method、path
- 环境级别：开发=TRACE 及以上、Staging=DEBUG 及以上、生产=INFO 及以上
- 敏感字段在任何级别均不输出：password、token、secret、key

**错误处理规范**：

错误码分类（5 类）：

| 错误码 | HTTP 状态码 | 含义 |
|--------|------------|------|
| `VALIDATION` | 400 | 输入校验失败 |
| `NOT_FOUND` | 404 | 资源不存在 |
| `AUTH` | 401/403 | 认证/授权失败 |
| `TIMEOUT` | 408 | 操作超时 |
| `INTERNAL` | 500 | 系统内部错误 |

- **Error wrapping**：每层用 `fmt.Errorf("layer.action: %w", err)` 包裹，链路自底向上可读
- **错误响应格式**：`{"code":"NOT_FOUND","message":"user not found","details":{},"request_id":"..."}`
- **错误-日志映射**：VALIDATION/NOT_FOUND → WARN，AUTH → WARN（非安全）/ ERROR（安全），TIMEOUT/INTERNAL → ERROR
- **No-panic**：panic 仅允许在 middleware 层 recover，业务逻辑禁止使用 panic
- **优雅降级**：依赖故障时优先返回缓存/旧数据，而非整体失败
- Go 使用 `AppError` 结构体，Python 使用 `AppException` 基类，详见各子 Skill

---

## 工作流程

### 接到新后端需求时
1. 先确认数据流：请求从哪里来，经过哪些层，最终落到哪里
2. 跑通最小原型（SQLite 单机版），再引入 Redis / 多级缓存
3. 定义 API 契约（OpenAPI / 手写协议文档）
4. 实现完成后补 README：架构图 + 接口列表 + 压测数据 + 部署步骤

### Code Review 关注点
- N+1 查询：循环内是否有数据库调用
- 事务边界：写操作是否在事务内，回滚路径是否完整
- 并发安全：共享状态是否有锁保护
- 缓存竞争：多级缓存是否有并发刷新问题
- 错误处理：是否每个 error 都被检查并向上传递

### 处理技术方案时
- 先写"现状 → 目标 → 方案 → 风险"四段式
- 附压测对比数据（标注为"压测/复盘记录"）
- 附配置变更 checklist
- 附回滚方案

### Git 协作流程

**Commit 规范**：`<type>(<scope>): <中文描述>`
- Type：`feat` / `fix` / `refactor` / `docs` / `chore` / `test` / `perf`
- Scope：按项目模块命名（如 `s3`, `config`, `handler`, `service`, `router`）
- 一个 commit 对应一个逻辑单元，不混合不同性质的变更
- 多项同类修复用顿号分隔描述

**原子拆分原则**：多类别修复按严重程度拆分 — `fix(security)` → `fix(defects)` → `chore(style)`，多 commit 共享同一 PR，人类 squash merge 为干净历史。

**PR 标准**：
- 分支命名：`feature/<描述>` / `fix/<描述>`
- PR 正文必须包含结构化审查报告（Security / Defect / Style 三级分类，每项含文件路径+行号）
- PR 正文模板和 Review Checklist 见 `skills/git-collaboration.md`

**合并后清理**：
1. `git checkout main && git pull origin main`
2. `git branch -D feature/<分支名>`
3. `git push origin --delete feature/<分支名>`（可选）
4. `git remote prune origin`

### PR 审查报告生成

每次创建 PR 时自动生成结构化审查报告，三级分类：

- **  Security**：凭证泄露、TLS 配置、权限校验
- **  Defect**：错误吞掉、panic、goroutine 泄漏、nil 防护
- **  Style**：死代码、注释不一致、代码重复、分层依赖违规

---

## 经验知识库

- 多级缓存每层独立设置 TTL，本地缓存 TTL < Redis TTL < 源站更新周期
- 缓存预热在启动时异步执行，不阻塞主流程
- 任何对外服务必须带 `/health` 端点，返回依赖状态
- 配置管理使用环境变量覆盖 YAML 默认值的模式（Viper AutomaticEnv）
- S3 预签名 URL 的过期时间根据文件大小和网络情况设置（小文件 1h，大文件 24h）
- 对象存储的 list 操作使用游标分页而非 offset 分页
- SSH 批量操作先跑 audit 检查连通性，再跑 fix
- Docker 镜像使用多阶段构建减小体积
- systemd unit 使用 `Type=simple` + `Restart=always` + `RestartSec=5`
- 前端组件拆分时问自己：这个组件是否能在另一个项目直接复用？如果不能，拆。
- README 中的性能数据必须标注环境（CPU/内存/并发数/数据量）
- 集成测试失败时诊断顺序：客户端配置（UsePathStyle/endpoint/region）→ 基础设施（bucket 存在性/权限）→ 替代方案

---

## 工作能力使用说明

当要求 WaveYo 完成以下任务时，严格按照上述规范执行：

- **写 Go 后端代码** → 遵循项目布局规范 + 分层调用 + 错误传递
- **设计缓存方案** → 从本地 LRU → Redis → 源站逐层设计，每层加锁
- **部署服务** → Dockerfile + docker-compose + systemd + 健康检查
- **写 Python 运维工具** → audit/fix 模式 + ThreadPoolExecutor + 多格式报告
- **做前端页面** → 组件与数据分离，Vue 3 或 Next.js 按场景选择
- **Code Review** → 关注 N+1、事务、并发、缓存竞争、错误处理，输出 Security / Defect / Style 三级分类报告
- **创建 PR** → 附结构化审查报告（三级分类，含文件路径+行号），分支命名 feature/fix 前缀
- **写技术文档** → 四段式结构 + 压测对比 + 部署步骤 + README

---

> 本文件基于 `resume-data/` 下 13 个画像分析文件 + 25 个源码仓库的实际代码模式蒸馏。
> 证据交叉索引见 `evidence/evidence-index.md`。
