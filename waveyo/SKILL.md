---
name: waveyo
description: >
  WaveYo 工程技能：跨项目复用的工程纪律与交付习惯，不绑定任何单一业务项目。
  当需要编写或评审后端服务、前端页面与交互、命名与注释、日志与可观测性、
  上游依赖（SDK / HTTP / RPC / 数据库驱动 / 对象存储）的错误处理、
  或处理分支提交 PR 与 AI 协作交付时使用此技能。
  也应在以下隐含场景主动触发：新增文件/函数/类型、决定包名与表名、选择日志级别、
  判断是否该引入缓存或分布式组件、做界面层级与交互反馈设计、
  处理「上游报错但不知原因」「结果未知」「HTTP 200 但业务失败」、
  编写结构化审查报告、跨会话接续开发。
  即使用户没提「规范」，只要任务涉及以上工程判断，都应加载并按需读取对应专题。
version: "2.1.0"
user-invocable: true
---

# WaveYo — 工程技能

> **定位**：一套可跨项目使用的工程纪律。管**判断方式与交付底线**（命名、注释、日志、
> 最小闭环、界面品质、上游错误、协作与 Git），不管某个业务领域的表结构与接口细节。
>
> **它不是什么**：不是某个项目的专属规范，不预设特定框架、特定数据库、特定目录结构；
> 具体项目的约定（框架、目录、配置路径、公开错误契约）永远优先于本 skill 的默认建议。

---

## 1. 优先级

```
人类明确指令  >  所在仓库的显式规范（AGENTS.md / CONTRIBUTING / 设计系统）  >  本 skill  >  通用最佳实践
```

- 用户说「这次按 X 做」→ 按 X，不必援引本 skill。
- 本 skill 与仓库规范冲突 → **仓库规范优先**，除非用户明确要求覆盖。
- 本 skill 未覆盖 → 用通用最佳实践，并在 PR 或交付说明中写明判断依据。
- 两条**不可被项目豁免**的底线：敏感信息不落日志、不把未验证的内容说成已完成。

---

## 2. 触发说明

| 场景 | 必读专题 |
|---|---|
| 前端页面 / 导航 / 表单 / 列表 / Dialog / 主题 / 动效 / 响应式 / 视觉验收 | `skills/frontend-ui-ux.md` |
| 接入 SDK、HTTP/RPC、数据库驱动、对象存储、消息队列；排查超时/取消/结果未知 | `skills/upstream-error-handling.md` |
| 新增标识符（包/类型/函数/表/路由/参数） | `skills/naming.md` |
| 写文件头、函数注释、接口契约 | `skills/comment-style.md` |
| 加日志、调级别、配置输出 | `skills/logging.md` |
| 接新需求、考虑引入缓存/队列/分布式组件 | `skills/minimal-prototype.md` |
| 与 AI 协作编码、审查、交付 | `skills/ai-collaboration.md` |
| 建分支、写 commit、开 PR、Code Review | `skills/git-workflow.md` |
| 写 CI 配置、调 CI runner、排 CI 网络/TLS 失败 | `skills/cicd-actions.md` |
| 改路由/字段名/类型，生成或维护 OpenAPI 文档，接口弃用 | `skills/openapi-doc.md` |

**按需读取**：主题详细规则只在对应专题维护，本入口只做索引与底线，不复制正文——
避免同一规则两处维护、日后互相矛盾。

---

## 3. 十条工程底线

| # | 底线 | 展开位置 |
|---|---|---|
| B1 | 命名实体优先：类型/表/包用纯名词，函数用动词 + 名词，不用 `Manager`/`Helper`/`Util` 泛词 | `skills/naming.md` |
| B2 | 注释解释「为什么」，公开函数写清契约（前提/副作用/错误） | `skills/comment-style.md` |
| B3 | 日志分级统一（TRACE/DEBUG/INFO/WARN/ERROR），敏感字段任何级别都不输出 | `skills/logging.md` |
| B4 | 最小闭环先行：先用项目已有的基础设施跑通真实链路，再按证据增加组件 | `skills/minimal-prototype.md` |
| B5 | 界面状态诚实：加载/空/错/成功可区分，刷新不清空，返回能复原 | `skills/frontend-ui-ux.md` |
| B6 | 上游错误保留原始错误链，提取结构化诊断，对外只暴露项目定义的公开语义 | `skills/upstream-error-handling.md` |
| B7 | 变更走分支 + PR；合并与发布由人类决策（无明确授权时 AI 不自行合并） | `skills/ai-collaboration.md` |
| B8 | 提交保持原子：安全 > 缺陷 > 重构 > 风格 > 文档 > 清理，按类别拆分 | `skills/git-workflow.md` |
| B9 | 合并保留原始提交与合并节点（Merge PR）；squash 不是默认策略，除非用户或项目明确要求 | `skills/git-workflow.md` |
| B10 | 不把未验证、未执行、计划中的内容写成已完成 | `persona.md` Layer 5 |

---

## 4. 交付前自检（通用最小集）

- [ ] 命名符合实体优先，未引入语义空洞的泛词
- [ ] 新文件有文件头说明；变更过的公开接口写清契约
- [ ] 日志级别正确，未输出密码/令牌/密钥/验证码/完整联系方式
- [ ] 错误保留了原始链（`%w` 或等价机制），对外文案不含上游原文
- [ ] 界面（如涉及）四态可区分，键盘可用，提供动态验收证据
- [ ] 变更已按性质拆分提交，commit 描述具体到做了什么
- [ ] 构建/静态检查/相关测试已实际执行，命令与结果可复述
- [ ] 未验证的部分已明确列为「待验证」，未伪装成已完成

---

## 5. 专题索引

| 专题 | 何时必读 |
|---|---|
| [`skills/frontend-ui-ux.md`](skills/frontend-ui-ux.md) | 任何页面结构、视觉、交互、状态反馈、可访问性改动 |
| [`skills/upstream-error-handling.md`](skills/upstream-error-handling.md) | 任何外部依赖的错误映射、诊断、日志、重试设计 |
| [`skills/naming.md`](skills/naming.md) | 新增任何标识符之前 |
| [`skills/comment-style.md`](skills/comment-style.md) | 编写或修改文件头、函数、接口注释 |
| [`skills/logging.md`](skills/logging.md) | 添加日志语句、调整日志级别 |
| [`skills/minimal-prototype.md`](skills/minimal-prototype.md) | 接到新需求、考虑引入新组件时 |
| [`skills/ai-collaboration.md`](skills/ai-collaboration.md) | AI 参与编码/审查/交付时 |
| [`skills/git-workflow.md`](skills/git-workflow.md) | 创建分支、写 commit、建 PR、Code Review |
| [`skills/cicd-actions.md`](skills/cicd-actions.md) | 写 CI 配置、调 runner、排 CI 网络与 TLS 问题 |
| [`skills/openapi-doc.md`](skills/openapi-doc.md) | 改路由/结构体字段、生成或增量更新 OpenAPI 文档 |

配套文档：

| 文件 | 内容 |
|---|---|
| [`persona.md`](persona.md) | 工程原则、技术定位、表达习惯、决策模式、技术红线（5 层框架） |
| [`work.md`](work.md) | 技术面：后端 / 前端 / 自动化 / 运维交付的规范与工作流程 |
| [`collaboration.md`](collaboration.md) | 人机协作协议：任务范围、角色边界、审查报告、跨会话延续 |
| [`versions/`](versions/) | 版本演进记录（当前版本见本文件 frontmatter） |
| [`archive/`](archive/) | 历史快照，**只作溯源，不具当前执行效力** |

---

## 6. 目录结构

```
waveyo/
├── SKILL.md                          # 本文件：定位、优先级、底线、索引
├── persona.md                        # 工程人格（5 层框架）
├── work.md                           # 技术工作技能与工作流程
├── collaboration.md                  # 人机协作规范
├── skills/                           # 专题（详细规则在此维护）
│   ├── frontend-ui-ux.md             # 前端 UI/UX 设计要求
│   ├── upstream-error-handling.md    # 上游错误正确处理
│   ├── naming.md
│   ├── comment-style.md
│   ├── logging.md
│   ├── minimal-prototype.md
│   ├── ai-collaboration.md
│   ├── git-workflow.md
│   ├── cicd-actions.md
│   └── openapi-doc.md                # OpenAPI 接口文档规范
├── versions/                         # 版本记录
└── archive/                          # 历史快照（无当前效力）
```

---

## 7. 维护约定

1. **一条规则只写一处**：详细规则进 `skills/*.md`，其他文档只引用，不复述。
2. **新增专题**需同时更新：本文件 §2 触发表、§5 索引、§6 目录、`work.md` 对应工作入口。
3. **改默认策略**（如合并方式）必须同步：`skills/git-workflow.md`、`collaboration.md`、
   `work.md` 与相关流程图，避免文字与图示打架。
4. **不写入具体项目私有信息**：私有地址、内部路径、凭据、单项目表结构、未脱敏日志样例。
   示例使用中性名称与占位符。
5. **历史不改写**：`archive/` 保留原日期与原始结论，只在必要时做匿名化与链接修复，
   并标注「归档，不具当前效力」。

---

> 方法论溯源：本 skill 采用 colleague-skill 的 5 层 Persona 结构 + Work Skill 双轨架构，
> 内容来自公开代码与协作记录的归纳，不引用私有数据。框架参考见
> [`../reference/colleague-skill-5layer-structure.md`](../reference/colleague-skill-5layer-structure.md)。
