# WaveYo Skill

> *将一位开发者 27 个公开仓库中反复出现的工程模式、技术习惯和架构判断，蒸馏为一套可被 AI 加载的复用技能。*

[![Skill Type](https://img.shields.io/badge/Skill-Engineering-blue)](./waveyo/SKILL.md) [![Version](https://img.shields.io/badge/version-1.3.0-green)](./waveyo/meta.json) [![DeepSeek](https://img.shields.io/badge/DeepSeek_TUI-AgentSkills-purple)](https://github.com/deepseek) [![Claude](https://img.shields.io/badge/Claude_Code-AgentSkills-0e6eb8)](https://www.anthropic.com/) [![Cursor](https://img.shields.io/badge/Cursor-AgentSkills-00b4ff)](https://www.cursor.so/) [![Codex](https://img.shields.io/badge/Codex-AgentSkills-8a2be2)](https://openai.com/blog/codex)

---

## 这是什么

**WaveYo** 是对 GitHub 开发者 WavesMan 的工程技能蒸馏产物。它不是一份简历，也不是一个静态的代码模板库——而是一个加载后能让 AI 以 WaveYo 的工程思维完成工作的 **DeepSeek Skill**。

加载后，AI 会获得：
- **11 条硬工程原则**：缓存必加锁、服务必带 health、README 先于代码、配置即数据、API 同构、密码学安全、连接生命周期管理……
- **6 个技术领域的完整能力**：Go 后端 / Python 自动化 / 前端工程 / Rust 系统工具 / 多语言 Monorepo / 运维交付
- **7 个独立可复用的子 Skill**：从 Go 脚手架到多级缓存到项目交付，新增 Git 协作规范
- **5 层技术决策框架**：正确性 > 可观测性 > 性能 > 部署 > 扩展
- **人机协作规范**：角色边界、信任梯度、跨会话状态传递、PR 审查报告自动生成

---

## 安装

### 人工安装

#### 从 GitHub 安装（推荐）

```bash
/skill install github:WavesMan/WaveYo-Skill
```

#### 更新已安装的 Skill

```bash
/skill update waveyo
```

#### 直接 clone 到全局 skills 目录

```bash
git clone https://github.com/WavesMan/WaveYo-Skill.git ~/.deepseek/skills/waveyo
```

#### 手动复制

将 `waveyo/` 目录复制到 `~/.deepseek/skills/` 下。

---

### AI Agent 安装

以下方式面向 AI Agent（Claude Code、Cursor、Codex、DeepSeek 等）的自动化安装场景。

工作前需要根据 AI Agent 所属系统环境，检查工具依赖情况： `git` `curl` `wget`，根据环境选择下述安装方式

#### 方式 1：Git Clone（推荐）

适用于有 shell 执行能力的 Agent。克隆后仅需 `waveyo/` 目录。

```bash
# 克隆到临时目录
git clone --depth 1 https://github.com/WavesMan/WaveYo-Skill.git /tmp/waveyo-skill

# 按平台复制到 skills 目录
# Claude Code
cp -r /tmp/waveyo-skill/waveyo ~/.claude/skills/waveyo

# DeepSeek TUI
cp -r /tmp/waveyo-skill/waveyo ~/.deepseek/skills/waveyo

# Cursor（项目级）
cp -r /tmp/waveyo-skill/waveyo .cursor/skills/waveyo

# 通用（自定义路径）
cp -r /tmp/waveyo-skill/waveyo <YOUR_SKILLS_DIR>/waveyo

# 清理
rm -rf /tmp/waveyo-skill
```

#### 方式 2：GitHub API（无需 git）

适用于无 git 但可发 HTTP 请求的 Agent。通过 GitHub API 下载 tarball 并解压。

```bash
# 下载 main 分支 tarball
curl -sL https://api.github.com/repos/WavesMan/WaveYo-Skill/tarball/main -o /tmp/waveyo-skill.tar.gz

# 解压（自动创建顶层目录）
mkdir -p /tmp/waveyo-skill
tar -xzf /tmp/waveyo-skill.tar.gz -C /tmp/waveyo-skill --strip-components=1

# 复制 waveyo/ 到目标目录
cp -r /tmp/waveyo-skill/waveyo ~/.claude/skills/waveyo

# 清理
rm -rf /tmp/waveyo-skill /tmp/waveyo-skill.tar.gz
```

#### 方式 3：Curl / Wget 直接下载

适用于仅支持基础 HTTP 的受限环境。下载 zipball 并解压。

```bash
# Curl 方式
curl -sL https://github.com/WavesMan/WaveYo-Skill/archive/refs/heads/main.zip -o /tmp/waveyo-skill.zip
unzip -q /tmp/waveyo-skill.zip -d /tmp/waveyo-skill
cp -r /tmp/waveyo-skill/WaveYo-Skill-main/waveyo ~/.claude/skills/waveyo
rm -rf /tmp/waveyo-skill /tmp/waveyo-skill.zip

# Wget 方式
wget -q https://github.com/WavesMan/WaveYo-Skill/archive/refs/heads/main.zip -O /tmp/waveyo-skill.zip
unzip -q /tmp/waveyo-skill.zip -d /tmp/waveyo-skill
cp -r /tmp/waveyo-skill/WaveYo-Skill-main/waveyo ~/.claude/skills/waveyo
rm -rf /tmp/waveyo-skill /tmp/waveyo-skill.zip
```

#### 方式 4：GitHub API 单文件获取

适用于只需加载部分文件的 Agent（按需拉取，不下载整个仓库）。

```bash
# 获取文件列表
curl -sL https://api.github.com/repos/WavesMan/WaveYo-Skill/contents/waveyo | jq -r '.[].name'

# 下载单个文件（以 SKILL.md 为例）
curl -sL https://raw.githubusercontent.com/WavesMan/WaveYo-Skill/main/waveyo/SKILL.md

# 下载单个文件并保存
curl -sL https://raw.githubusercontent.com/WavesMan/WaveYo-Skill/main/waveyo/work.md -o ~/.claude/skills/waveyo/work.md
```

#### 安装验证

安装完成后，Agent 可通过以下方式验证：

```bash
# 检查 SKILL.md 存在且可读
cat ~/.claude/skills/waveyo/SKILL.md | head -5

# 检查版本号
grep "version:" ~/.claude/skills/waveyo/SKILL.md

# 检查子 Skill 完整性
ls ~/.claude/skills/waveyo/skills/
# 预期输出：7 个 .md 文件
```

#### 平台路径速查

| Agent 平台 | Skills 目录 |
|---|---|
| Claude Code | `~/.claude/skills/` |
| DeepSeek TUI | `~/.deepseek/skills/` |
| Cursor | `.cursor/skills/`（项目级）或 `~/.cursor/skills/`（全局） |
| Codex | `~/.codex/skills/` |
| 通用 | 自定义，确保 Agent 启动时可读取 |

---

## 使用

```
# 激活完整 WaveYo 工程能力
/waveyo

# 独立加载子 Skill
/skill waveyo/skills/go-service-scaffold
/skill waveyo/skills/multi-level-cache
/skill waveyo/skills/git-collaboration
```

---

## 子 Skill 概览

| 子 Skill | 触发场景 | 核心内容 |
|---|---|---|
| `go-service-scaffold` | 新建 Go 后端项目 | 标准布局、Viper 配置、Zap 日志、健康检查、优雅关闭 |
| `multi-level-cache` | 引入缓存优化 | 本地 LRU → Redis → 源站，分片锁保护，异步刷新，Prometheus 指标 |
| `object-storage-integration` | 接入 S3/COS/OSS | 预签名 URL、CDN 优先下载、游标分页、镜像同步编排 |
| `gateway-bff-pattern` | 设计网关或 BFF | 控制面/数据面分离、Host 路由、负载均衡、审计日志、配置回滚 |
| `python-ssh-audit` | Linux 批量巡检/修复 | audit/fix 模式分离、ThreadPoolExecutor、多格式报告、进度心跳 |
| `project-delivery` | 服务打包交付 | Dockerfile 多阶段构建、docker-compose、systemd unit、构建脚本 |
| `git-collaboration` | 分支/提交/PR/Code Review | Angular Commit 规范、PR 模板、Review Checklist、合并清理流程 |

---

## 目录结构

```
waveyo-skill/
├── waveyo/
│   ├── SKILL.md                          # 主入口（AgentSkills 标准）
│   ├── work.md                           # 技术工作技能
│   ├── persona.md                        # 5 层工程人格
│   ├── collaboration.md                  # 人机协作规范（v1.1.0 新增）
│   ├── meta.json                         # 元数据
│   ├── skills/
│   │   ├── go-service-scaffold.md        # Go 服务脚手架
│   │   ├── multi-level-cache.md          # 多级缓存模式
│   │   ├── object-storage-integration.md # 对象存储集成
│   │   ├── gateway-bff-pattern.md        # 网关/BFF 模式
│   │   ├── python-ssh-audit.md           # Python SSH 巡检
│   │   ├── project-delivery.md           # 项目交付部署
│   │   └── git-collaboration.md          # Git 协作规范（v1.1.0 新增）
│   └── versions/
│       └── v1.0.0.md                     # v1.0.0 版本快照
├── evidence/
│   └── evidence-index.md                 # 证据交叉索引表
├── reference/
│   └── colleague-skill-5layer-structure.md  # 方法论参考
└── README.md
```

---

## 人机协作规范（v1.1.0 新增）

WaveYo 作为 AI 助手与人类协作时，遵循 `collaboration.md` 中定义的协议：

- **协作生命周期**：人类发起 → AI 执行 + 审查报告 → 人类审核 → squash merge
- **角色边界**：架构决策 / 合并决策由人类独占，AI 负责执行、审查、文档
- **信任梯度**：T0（逐行审查）→ T1（审查报告+CI）→ T2（快速扫视），安全 PR 始终 T0
- **跨会话延续**：每场会话结束时输出延续提示词（HEAD commit + 已完成清单 + 待讨论方向）
- **PR 审查报告**：Security / Defect / Style 三级分类，每项含文件路径+行号

---

## 蒸馏方法论

### 数据源

不同于 [colleague-skill](https://github.com/titanwings/colleague-skill) 依赖聊天记录，本次蒸馏以**代码证据驱动**：

| 数据源 | 数量 | 内容 |
|---|---|---|
| 源码仓库 | 27 个（10 后端 + 16 前端 + 1 本地） | 实际代码、目录结构、部署配置 |
| 画像分析文件 | 20 个 | 技术能力矩阵、职责边界、综合画像 |
| 边界约束文件 | 3 个 | 表达边界、fork 声明规则、禁用词清单 |
| 协作会话记录 | 1 场全流程 | 83 行 reflog + 5 个 PR 完整生命周期 |

### 框架适配

借用了 colleague-skill 的 **5 层 Persona 结构**，但做了完整的内容重写：

```
colleague-skill                 →    WaveYo 适配
─────────────────────────────────────────────────
Layer 0: 性格标签 → 行为规则     →    工程原则（代码证据推断）
Layer 1: 公司/职级               →    技术定位（Go 主线 + 4 辅助线 + 跨语言线）
Layer 2: 口头禅/说话方式         →    工程表达习惯（README 风格等）
Layer 3: 人际决策               →    技术决策模式（优先级排序）
Layer 4: 人际行为               →    协作行为（fork 标注 / 文档交接 / 人机协作）
Layer 5: 雷区/边界              →    技术红线（禁止夸大 / 边界约束）
```

### 质量控制

- 所有技术声明必须有对应的证据索引条目可回溯
- 所有"精通""架构师""企业级"等禁用词通过 grep 验证已排除
- fork/协作项目标注"基于上游优化"或"PR 贡献"
- 性能数据标注"压测记录 / README 记录"
- 协作规范从实操会话中提取，每条规则可追溯至具体 PR

---

## 版本

- **1.3.0** (2026-06-25)：新增 2 个仓库蒸馏（cn-area + freessh）+ 声明优化。新增 Rust/Tauri 桌面应用开发、多语言 Monorepo + 代码生成、解耦规则统一归档。扩展全函数中文注释规范（GoDoc/Rust///Python docstring/JSDoc 四语言统一）。新增文件行数限制（前端 600 / 后端 800）。覆盖仓库数 25 → 27。
- **1.2.0** (2026-05-22)：新增 5 级日志体系（TRACE < DEBUG < INFO < WARN < ERROR）+ 统一错误处理规范（AppError/AppException 分类、错误码体系、错误中间件、错误-日志级别映射、缓存 miss/error 区分）。
- **1.1.0** (2026-05-17)：新增人机协作规范体系 — `collaboration.md` + `git-collaboration` 子 Skill。persona.md Layer 4 扩展人机协作小节，work.md 扩展 Git 协作流程和 PR 审查报告生成。
- **1.0.0** (2026-05-15)：初始蒸馏，覆盖 25 个仓库 + 20 个画像文件

---

## 许可

[MIT License](LICENSE)
