# WaveYo Skill — 开发者技能蒸馏

> 将 WavesMan 在 25 个公开仓库中体现的开发模式、技术习惯和工程判断，蒸馏为可被 AI 加载的 DeepSeek Skill。

## 快速使用

加载 `waveyo/SKILL.md` 后，AI 将获得 WaveYo 的工程能力：

- 写 Go 后端代码 → 遵循 cmd/internal/api/configs 布局 + 分层调用
- 设计缓存方案 → 本地 LRU → Redis → 源站，每层加锁保护
- 部署服务 → Dockerfile + docker-compose + systemd + 健康检查
- 写 Python 运维工具 → audit/fix 模式 + ThreadPoolExecutor
- Code Review → 关注 N+1、事务、并发、缓存竞争

子 Skill 可独立加载：`/waveyo-skills/go-service-scaffold` 等。

## 目录

```
my_skill/
├── waveyo/                 # 蒸馏产物
│   ├── SKILL.md            # 主入口
│   ├── work.md             # 技术工作技能
│   ├── persona.md          # 工程人格
│   ├── meta.json           # 元数据
│   └── skills/             # 6 个领域专项 Skill
├── evidence/               # 证据索引
│   └── evidence-index.md
├── reference/              # 方法论参考
│   └── colleague-skill-5layer-structure.md
└── README.md               # 本文件
```

## 蒸馏方法

参考 colleague-skill 的 5 层 Persona 框架，适配为代码证据驱动模式：

- **数据源**：代码仓库目录结构 + 源码模式 + 画像分析文件（替代聊天记录）
- **Persona**：5 层工程人格（替代人际人格）
- **标签库**：工程模式标签（替代职场性格标签）
- **边界约束**：从简历表达边界文件自动注入，防止夸大

## 证据来源

- 画像文件：`resume/resume-data/` 下 20 个分析文件
- 源码仓库：25 个完整仓库（`github-backend-analysis/` + `github-frontend-analysis/`）
- 硬约束：3 个边界文件（禁止夸大、fork 标注、上游声明）

## 版本

- **1.0.0** (2026-05-15)：初始蒸馏版本，覆盖 25 个仓库 + 20 个画像文件
