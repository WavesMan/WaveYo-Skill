# Git Collaboration — WaveYo 风格

> 触发条件：需要创建分支、编写 commit、创建 PR 或执行 Code Review 时加载。
> 来源证据：2026-05-17 YoMirrorSite 全流程会话（5 个 PR 完整生命周期）。
> 可独立加载：是。

---

## Commit 规范

### 格式

```
<type>(<scope>): <中文描述>
```

### Type 枚举

| Type | 场景 | 示例 |
|---|---|---|
| `feat` | 新功能 | `feat(s3): 添加分片上传支持` |
| `fix` | 缺陷/安全修复 | `fix(security): 移除 S3 AccessKey 调试日志` |
| `refactor` | 重构（无功能变更） | `refactor(model): 将 FileInfo 下沉到 model 层` |
| `docs` | 文档变更 | `docs(readme): 补充部署步骤和配置说明` |
| `chore` | 清理/构建/工具/格式 | `chore(cleanup): 清理临时文件、删除构建脚本` |
| `test` | 测试相关 | `test(handler): 补充 handler 层单元测试` |
| `perf` | 性能优化 | `perf(cache): 优化 LRU 淘汰策略` |

### Scope 规范

Scope 按项目模块命名，以下为 YoMirrorSite 项目的 scope 示例：

```
s3        — 对象存储模块
config    — 配置管理
handler   — HTTP handler 层
syncer    — 数据同步模块
service   — 业务逻辑层
router    — 路由注册
ci        — CI/CD 配置
util      — 工具函数
model     — 数据模型
postgres  — PostgreSQL 相关
github    — GitHub API 集成
security  — 安全相关
defects   — 缺陷修复（多模块通用）
cleanup   — 代码清理（多模块通用）
build     — 构建相关
delivery  — 部署交付
```

不同项目的 scope 应按其模块结构对应调整。

### 描述规则

- 使用中文，简洁说明变更内容
- 多项同类修复用顿号分隔：`修复静默错误吞掉、panic 改 error、goroutine recover`
- 一个 commit 对应一个逻辑单元，不混合不同性质的变更

---

## 原子提交拆分原则

当单个 PR 包含多类修复时，按严重程度和性质拆分为独立 commit：

```
commit 1: fix(security): ...          ← 最高优先级（安全漏洞）
commit 2: fix(defects): ...           ← 缺陷修复（运行时问题）
commit 3: chore(style): ...           ← 风格/规范清理
```

**优先级排序**：安全 > 缺陷 > 重构 > 风格 > 文档 > 清理

**拆分粒度判断**：
- 修改了同一个函数的不同行 → 合并在一个 commit
- 修改了不同模块/不同性质的代码 → 拆分为多个 commit
- 多 commit 共享同一个 PR，由人类通过 squash merge 合并为一条干净历史

---

## PR 标准

### 分支命名

```
feature/<描述>     — 功能开发 / 审查修复 / 清理
fix/<描述>         — 紧急修复
```

命名示例：
- `feature/security-review-fix`
- `feature/cache-cleanup`
- `feature/deadcode-style-cleanup`
- `fix/go-compile-errors`

### PR 标题

遵循 commit 规范格式，概括 PR 目的：

```
fix(backend): 代码审查修复 — 安全漏洞 + 运行时缺陷 + 配置加固
chore(cleanup): 清理临时文件、删除 release-build.ps1、修正 .gitignore
```

### PR 正文模板

```markdown
## 变更概要
<一句话说明 PR 目的和变更范围>

## 修复清单

###  安全漏洞
| # | 文件:行 | 问题 | 修复 |
|---|---|---|---|
| S1 | `path/file.go:42` | <问题描述> | <修复说明> |

###  缺陷修复
| # | 文件:行 | 问题 | 修复 |
|---|---|---|---|

###  风格/规范
| # | 文件 | 问题 | 修复 |
|---|---|---|---|

## 测试验证
- [ ] `go test ./...` 通过
- [ ] `go vet ./...` 无警告
- [ ] 集成测试通过
- [ ] 前端构建通过（如涉及）

## 风险说明
- 破坏性变更：<无 / 列出具体变更>
- 依赖变更：<无 / 列出新增或升级的依赖>

## 审核者
@WavesMan
```

### 提交数量建议

| 变更规模 | 建议 commit 数 | 示例 |
|---|---|---|
| 单文件小修 | 1 | 修正一个配置项 |
| 多文件同类别 | 1–3 | 模块级重构 |
| 跨模块多类别 | 3–5 | 安全+缺陷+风格综合修复 |
| 大型功能 | 5+ | 按功能拆分为多个 PR |

---

## PR Review Checklist

人类审核 PR 时逐项确认：

```
[ ] CI 全部通过（unit + integration + vet + frontend）
[ ] PR 正文包含结构化审查报告（Security / Defect / Style 三级分类）
[ ] 每个问题标注了文件路径和行号
[ ] 修复方案与问题一一对应
[ ] 无破坏性变更（或已明确列出并说明影响范围）
[ ] 无依赖变更（或已明确列出变更原因）
[ ] Commit 信息符合 Angular 规范（<type>(<scope>): <中文描述>）
[ ] 分支命名符合 feature/xxx 或 fix/xxx
[ ] 无敏感信息泄露（凭证/TLS 密钥/内网地址）
```

---

## 合并后清理步骤

PR 被 squash merge 后，执行以下清理：

```bash
# 1. 切换到 main 并拉取最新
git checkout main && git pull origin main

# 2. 删除本地 feature 分支
git branch -D feature/<分支名>

# 3. 删除远端 feature 分支（GitHub PR 合并后通常自动删除，可跳过）
git push origin --delete feature/<分支名>

# 4. 清理本地跟踪引用
git remote prune origin
```

---

## 线性历史原则

```
所有 PR 通过 squash merge 进入 main，保证 main 历史为一条直线：

  main:  A ─── B ─── C ─── D ─── E
          ↑      ↑      ↑      ↑
         PR#1   PR#3   PR#4   PR#5
```

每个 squash merge 的 commit 包含 PR 编号和标题，便于回溯。

---

## 反模式（避免的行为）

- ❌ 直接在 main 上 commit 或 push
- ❌ merge commit 进入 main（必须 squash）
- ❌ 一个 PR 包含不相关的修复（混合安全漏洞修复和功能开发）
- ❌ PR 正文只有 "fixes #123" 没有结构化说明
- ❌ Commit 描述为 "fix bug" 或 "update code"（不具体）
- ❌ 分支名使用个人化命名（如 `johns-branch`、`tmp`、`test123`）
