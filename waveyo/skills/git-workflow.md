# Git 工作流规范（WaveYo 风格）

> 触发时机：创建分支、编写 commit、创建 PR、执行 Code Review 时必读。
> 来源：多个仓库的完整 PR 生命周期复盘，已做通用化处理。
> 与常见 Git 约定的差异见 §8；**具体项目另有规定时，以项目规范与用户指令为准**。

---

## 1. Commit 规范

### 1.1 格式

```
<type>(<scope>): <中文描述>
```

默认**不添加**额外的「共同撰写人」trailer；项目有明确要求时按项目要求执行。

### 1.2 Type 枚举

| Type | 场景 | 示例 |
|---|---|---|
| `feat` | 新功能 | `feat(upload): 实现 STS 前缀锁定凭证签发` |
| `fix` | 缺陷 / 安全修复 | `fix(security): 移除对象存储调试日志中的凭证` |
| `refactor` | 重构（无功能变更） | `refactor(model): 将 FileInfo 下沉到 model 层` |
| `docs` | 文档变更 | `docs(readme): 补充 OSS 配置说明` |
| `chore` | 清理 / 构建 / 工具 / 格式 | `chore(cleanup): 清理临时文件并加固 .gitignore` |
| `test` | 测试相关 | `test(handler): 补充上传接口单元测试` |
| `perf` | 性能优化 | `perf(cache): 优化 LRU 淘汰策略` |

> 与常见 6 类约定的差异：**新增 `docs`**（文档量大时值得独立成类，便于回溯文档变更）。

### 1.3 Scope 规范

Scope 按**领域 / 模块**命名，与目录、领域包或页面路由保持一致。以下是中性示例，
**按项目实际取名**，不要照搬：

```
auth        — 认证 / 会话
account     — 账号与资料
catalog     — 资源目录与检索
upload      — 上传与文件处理
order       — 订单与流程状态
notify      — 通知 / 消息
storage     — 对象存储适配
web         — 前端
config      — 配置管理
router      — 路由注册
ci          — CI/CD 配置
security    — 安全相关（多模块通用）
defects     — 缺陷修复（多模块通用）
cleanup     — 代码清理（多模块通用）
build       — 构建相关
delivery    — 部署交付
```

判定方法：一个 scope 应该能对应到 reviewer 脑中一个明确的目录或领域；
如果某个改动横跨多个领域，就用 `security` / `defects` / `cleanup` 这类跨模块 scope。

### 1.4 描述规则

- **使用中文**，简洁说明变更内容
- 多项同类修复用**顿号**分隔：
  `fix(defects): 修复静默错误吞掉、panic 改 error、goroutine recover`
- 一个 commit 对应一个逻辑单元，**不混合**不同性质的变更
- 不加句号，首字母不强求小写（中文描述直接书写）

### 1.5 破坏性变更

在 type 后加 `!`，footer 写迁移说明：

```
feat(auth)!: 变更令牌载荷结构

BREAKING CHANGE: JWT 载荷字段 uid 更名为 user_id，客户端需同步调整解析逻辑。
```

---

## 2. 原子提交拆分原则

当单个 PR 包含多类修复时，按严重度拆分为独立 commit：

```
commit 1: fix(security): ...          ← 最高优先级（安全漏洞）
commit 2: fix(defects): ...           ← 缺陷修复（运行时问题）
commit 3: chore(cleanup): ...         ← 风格 / 规范清理
```

**优先级排序**：`安全 > 缺陷 > 重构 > 风格 > 文档 > 清理`

**拆分粒度判断**：

| 情形 | 处理 |
|---|---|
| 修改同一函数的不同行 | 合并在一个 commit |
| 修改不同模块 / 不同性质的代码 | 拆分为多个 commit |
| 重构与行为修复混在一起 | **必须拆开**（重构不得改变语义） |

多个 commit 共享同一 PR，由人工审核后以**保留原始提交与合并节点的 Merge PR** 合入集成分支。
**squash / rebase merge 不是默认策略**；只有当用户或项目规范明确要求时才使用，
并且必须在 PR 描述或交付说明里写明采用了该方式及原因。

---

## 3. PR 标准

### 3.1 分支命名

```
feature/<描述>     — 功能开发 / 审查修复 / 清理
fix/<描述>         — 紧急修复
```

描述用 **kebab-case**。示例：

- `feature/list-filter-restore`
- `feature/order-detail-page`
- `feature/upstream-timeout-handling`
- `fix/broken-relative-links`

> 禁止个人化命名（`test123`、`tmp`、`johns-branch`）。

### 3.2 PR 标题

遵循 commit 规范格式，概括 PR 目的：

```
feat(web): 新增订单详情页 — 独立路由 + 吸顶操作区 + 返回状态复原
fix(storage): 修正上传超时被误判为失败 — 保留错误链 + 结果未知处理
```

### 3.3 PR 正文模板

**正文必须用真实换行**：不要写出 `\n` 字面量，也不要把整段正文压成一行。
长正文先写成仓库外或仓库内（`.git` 忽略）的 UTF-8 文件，再用 body 文件提交
（如 `gh pr create --body-file pr.md`），**提交后回读一次**确认换行、表格、代码块渲染正常。

````markdown
## 变更概要
<一句话说明 PR 目的和变更范围>

## 修复清单

### 🔴 安全漏洞
| # | 文件:行 | 问题 | 修复 |
|---|---|---|---|
| S1 | `path/file.go:42` | <问题描述> | <修复说明> |

### 🟠 缺陷修复
| # | 文件:行 | 问题 | 修复 |
|---|---|---|---|

### 🟡 风格/规范
| # | 文件 | 问题 | 修复 |
|---|---|---|---|

## 测试验证
- [ ] `go build ./...` 通过
- [ ] `go vet ./...` 无警告
- [ ] 单元测试通过（`go test ./... -short`）
- [ ] 集成测试通过（`go test ./... -tags=integration`）
- [ ] 前端构建通过（如涉及）

## 验收证据

- 界面/交互：<录屏或连续帧链接或路径；键盘走查结果；200% 缩放下是否裁切>
- 上游错误：<失败分类清单 + 状态码/业务码 + 请求 ID + 耗时；结果未知如何收敛>
- 未验证项：<未跑的用例、未覆盖的环境>

## 风险说明
- 破坏性变更：<无 / 列出具体变更>
- 依赖变更：<无 / 列出新增或升级的依赖>
- 演进触发条件（如引入新组件）：<压测数据 / 错误日志 / 实例数证据>

## 审核者
@WaveYo
````

### 3.4 提交数量建议

| 变更规模 | 建议 commit 数 | 示例 |
|---|---|---|
| 单文件小修 | 1 | 修正一个配置项 |
| 多文件同类别 | 1–3 | 模块级重构 |
| 跨模块多类别 | 3–5 | 安全 + 缺陷 + 风格综合修复 |
| 大型功能 | 5+ | 按功能拆分为多个 PR |

---

## 4. PR Review Checklist

人类审核 PR 时逐项确认：

```
[ ] CI 全部通过（build + vet + unit + integration + frontend）
[ ] PR 正文包含结构化审查报告（Security / Defect / Style 三级分类）
[ ] 每个问题标注了文件路径和行号
[ ] 修复方案与问题一一对应
[ ] 无破坏性变更（或已明确列出并说明影响范围）
[ ] 无依赖变更（或已明确列出变更原因）
[ ] Commit 信息符合规范（<type>(<scope>): <中文描述>）
[ ] 分支命名符合 feature/xxx 或 fix/xxx
[ ] 无敏感信息泄露（凭证 / 密钥 / 内网地址）
[ ] PR 正文使用真实换行（无 `\n` 字面量），表格与代码块渲染正常
[ ] 界面改动附动态验收证据（录屏/连续帧），未只给静态截图
[ ] 上游依赖改动附失败分类与错误链验证结果
[ ] 未验证项已显式列出
[ ] 命名符合 Entity-First / Noun-First
[ ] 注释全中文，新文件有行头，公开函数有 GoDoc
[ ] 日志级别正确，未输出 password/token/secret/key
[ ] 未提前引入 Redis / 分布式组件（最小原型先行）
```

---

## 5. 合并后清理步骤

PR 被 merge 后执行：

```bash
# 1. 切换到集成分支并拉取最新
git checkout <integration-branch> && git pull origin <integration-branch>

# 2. 删除本地 feature 分支
git branch -D feature/<分支名>

# 3. 删除远端 feature 分支
git push origin --delete feature/<分支名>

# 4. 清理本地跟踪引用
git remote prune origin
```

若项目启用自动清理（如托管平台的 "delete branch after merge"），以平台行为为准，无需重复操作。

---

## 6. 集成与发布原则

功能 PR **保留原始提交及合并节点**，禁止以 squash / rebase merge 合入集成分支；
集成分支 → 发布分支通过人工审核的发布 PR 合入，同样保留合并节点，
便于版本流程读取完整语义变更：

```
  feature/* ──Merge PR──► 集成分支(dev) ──发布 PR──► 发布分支(main) ──► tag ──► Release
```

分支名与发布方式以项目约定为准（有的项目只有 `main`，有的用 `dev` + `main`）。
`main → dev` 之类的同步不创建正式 tag；版本号与发布操作遵循项目自身的发布文档，
版本流程不依赖固定的 merge 标题格式。

---

## 7. 反模式（避免的行为）

- ❌ 直接在发布分支（`main` 等）上 commit 或 push
- ❌ 把 `dev → main` 的语义历史压成无语义标题，且不设置 semver 标签
- ❌ 未经用户或项目规范要求就默认 squash，导致单个改动无法追溯
- ❌ 一个 PR 包含不相关的修复（混合安全漏洞修复与功能开发）
- ❌ PR 正文只有 "fixes #123" 没有结构化说明
- ❌ PR 正文里出现 `\n` 字面量（未使用真实换行）
- ❌ Commit 描述为 "fix bug" / "update code"（不具体）
- ❌ 分支名使用个人化命名
- ❌ 测试失败后修改断言以求「通过」

---

## 8. 与常见 Git 约定的差异

本 skill 的默认选择与部分通用约定不同（**项目另有规定时以项目为准**）：

| 冲突点 | 常见默认 | **本 skill 默认**                      |
|---|---|-------------------------------------|
| 主线分支 | `main` + `dev` 双线 | **以项目现状为准：功能 PR 进集成分支，发布 PR 进发布分支**           |
| 个人分支命名 | `<initials>/<feature>` | **`feature/<描述>` / `fix/<描述>`** |
| 合并方式 | `dev → main` 用 `--no-ff` | **保留原始提交与合并节点的 Merge PR；squash 需明确要求** |
| commit type | 6 类（无 `docs`） | **7 类**（含 `docs`）               |
| subject 语言 | 跟随交互语言 | **中文（与交付语言一致）**                        |
| 自动提交 | 完成后自动 commit、不询问 | **走 PR；AI 不直推发布分支**     |
| 分支名风格 | kebab-case | `feature/` 后用 kebab-case（一致）  |

**沿用的通用做法**：

- 「一个 commit 只做一件事」
- 破坏性变更用 `!` 标注 + `BREAKING CHANGE` footer
- 版本号用 Git tag `v<major>.<minor>.<patch>`
- Changelog 从语义提交生成：`git log $(git describe --tags --abbrev=0)..HEAD --pretty=format:"%s"`
- 需要线性历史时用 rebase 变基后再合并；**但「线性」不等于「必须 squash」**——
  两者按项目要求分别确定，不要用一个策略冒充另一个
