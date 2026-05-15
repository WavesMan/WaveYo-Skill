# WaveYo Skill

> *将一位开发者 25 个公开仓库中反复出现的工程模式、技术习惯和架构判断，蒸馏为一套可被 AI 加载的复用技能。*

[![Skill Type](https://img.shields.io/badge/Skill-Engineering-blue)](./waveyo/SKILL.md)
[![Version](https://img.shields.io/badge/version-1.0.0-green)](./waveyo/meta.json)
[![DeepSeek](https://img.shields.io/badge/DeepSeek-TUI%20%7C%20AgentSkills-purple)](https://github.com/deepseek)

---

## 这是什么

**WaveYo** 是对 GitHub 开发者 WavesMan 的工程技能蒸馏产物。它不是一份简历，也不是一个静态的代码模板库——而是一个加载后能让 AI 以 WaveYo 的工程思维完成工作的 **DeepSeek Skill**。

加载后，AI 会获得：
- **7 条硬工程原则**：缓存必加锁、服务必带 health、README 先于代码……
- **4 个技术领域的完整能力**：Go 后端 / Python 自动化 / 前端工程 / 运维交付
- **6 个独立可复用的子 Skill**：从 Go 脚手架到多级缓存到项目交付
- **5 层技术决策框架**：正确性 > 可观测性 > 性能 > 部署 > 扩展

---

## 安装

### 从 GitHub 安装（推荐）

```bash
/skill install github:<your-username>/waveyo-skill
```

### 直接 clone 到全局 skills 目录

```bash
git clone https://github.com/<your-username>/waveyo-skill.git ~/.deepseek/skills/waveyo
```

### 手动复制

将 `waveyo/` 目录复制到 `~/.deepseek/skills/` 下。

---

## 使用

```
# 激活完整 WaveYo 工程能力
/waveyo

# 独立加载子 Skill
/skill waveyo/skills/go-service-scaffold
/skill waveyo/skills/multi-level-cache
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

---

## 目录结构

```
waveyo-skill/
├── waveyo/
│   ├── SKILL.md                          # 主入口（AgentSkills 标准）
│   ├── work.md                           # 技术工作技能
│   ├── persona.md                        # 5 层工程人格
│   ├── meta.json                         # 元数据
│   └── skills/
│       ├── go-service-scaffold.md        # Go 服务脚手架
│       ├── multi-level-cache.md          # 多级缓存模式
│       ├── object-storage-integration.md # 对象存储集成
│       ├── gateway-bff-pattern.md        # 网关/BFF 模式
│       ├── python-ssh-audit.md           # Python SSH 巡检
│       └── project-delivery.md           # 项目交付部署
├── evidence/
│   └── evidence-index.md                 # 证据交叉索引表
├── reference/
│   └── colleague-skill-5layer-structure.md  # 方法论参考
└── README.md
```

---

## 蒸馏方法论

### 数据源

不同于 [colleague-skill](https://github.com/titanwings/colleague-skill) 依赖聊天记录，本次蒸馏以**代码证据驱动**：

| 数据源 | 数量 | 内容 |
|---|---|---|
| 源码仓库 | 25 个（9 后端 + 16 前端） | 实际代码、目录结构、部署配置 |
| 画像分析文件 | 20 个 | 技术能力矩阵、职责边界、综合画像 |
| 边界约束文件 | 3 个 | 表达边界、fork 声明规则、禁用词清单 |

### 框架适配

借用了 colleague-skill 的 **5 层 Persona 结构**，但做了完整的内容重写：

```
colleague-skill                 →    WaveYo 适配
─────────────────────────────────────────────────
Layer 0: 性格标签 → 行为规则     →    工程原则（代码证据推断）
Layer 1: 公司/职级               →    技术定位（Go 主线 + 3 辅助线）
Layer 2: 口头禅/说话方式         →    工程表达习惯（README 风格等）
Layer 3: 人际决策               →    技术决策模式（优先级排序）
Layer 4: 人际行为               →    协作行为（fork 标注 / 文档交接）
Layer 5: 雷区/边界              →    技术红线（禁止夸大 / 边界约束）
```

### 质量控制

- 所有技术声明必须有对应的证据索引条目可回溯
- 所有"精通""架构师""企业级"等禁用词通过 grep 验证已排除
- fork/协作项目标注"基于上游优化"或"PR 贡献"
- 性能数据标注"压测记录 / README 记录"

---

## 版本

- **1.0.0** (2026-05-15)：初始蒸馏，覆盖 25 个仓库 + 20 个画像文件

---

## 许可

MIT
