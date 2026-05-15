# WaveYo 技能蒸馏 — 证据交叉索引

> 生成时间：2026-05-15
> 证据来源：resume-data/ 画像 13 文件 + 源码仓库 25 个

## Go 后端工程

| 技能维度 | 证据强度 | 画像来源 | 源码验证仓库 | 边界约束 |
|---|---|---|---|---|
| Fiber/Gin/标准库多框架 | 高 | backend-01 | YoOSF-API (Fiber), Koishi-Mirror (Gin), ip-api (标准库) | — |
| cmd/internal/api/configs 布局 | 高 | backend-01 | YoBFF, Koishi-Mirror, YoOSF-API | — |
| Redis + LRU/LFU 多级缓存 | 高 | backend-01/04, 00-综合 | YoOSF-API (AcquireLock), Koishi-Mirror (semaphore) | 压测数据标注"README记录" |
| 健康检查端点 | 高 | backend-01/05, 00-综合 | YoOSF-API, Koishi-Mirror, YoBFF, ip-api | — |
| 结构化日志 (Zap) | 高 | backend-04 | YoBFF, YoOSF-API, Koishi-Mirror | — |
| OpenAPI 契约 | 中 | backend-01 | YoBFF (contracts/openapi/) | — |
| S3/OSS/COS 对象存储 | 高 | backend-01, 00-综合 | YoOSF-API, Koishi-Mirror | — |
| 控制面/数据面分离 | 中 | backend-01 | YoBFF (internal/admin vs internal/gateway) | 不写 enterprise |
| Docker + systemd 部署 | 高 | backend-05 | YoOSF-API, Koishi-Mirror, YoBFF, ip-api | — |
| Prometheus 指标 | 中 | backend-01 | ip-api | — |

## Python 自动化

| 技能维度 | 证据强度 | 画像来源 | 源码验证仓库 | 边界约束 |
|---|---|---|---|---|
| SSH 批量执行 | 高 | backend-01, 00-综合 | cve-2026-31431-fleet-remediator | 简历泛化CVE编号 |
| audit/fix 模式分离 | 高 | backend-01 | cve-2026-31431-fleet-remediator | — |
| 多格式报告 (JSON/CSV/MD) | 高 | backend-01 | cve-2026-31431-fleet-remediator | — |
| Flask WebUI | 中 | backend-01 | DeepSeek-R1-Local-WebUI | 不写模型训练 |
| ThreadPoolExecutor 并发 | 中 | backend-01 | cve-2026-31431-fleet-remediator | — |
| Windows 系统工具 | 高 | backend-01, 00-综合 | Disable-automatic-Windows-update | — |

## 前端工程

| 技能维度 | 证据强度 | 画像来源 | 源码验证仓库 | 边界约束 |
|---|---|---|---|---|
| Vue 3 + Vite + Pinia | 高 | frontend-01 | YoResume, captcha-index | — |
| Vue Router + Tailwind | 高 | frontend-01 | YoResume, captcha-index | — |
| 模板/数据分离 | 中 | frontend-01 | YoResume (src/registry/, src/resumes/templates/, src/resumes/data/) | — |
| Next.js App Router | 高 | frontend-01 | YoSpace | UI/UX 参考上游声明 |
| Markdown/frontmatter 内容系统 | 高 | frontend-01 | YoSpace | — |
| TypeScript | 高 | frontend-01 | YoSpace, Koishi-Registry, Koishi-WaveYo-AI | — |
| Koishi 生态 | 中 | frontend-01 | Koishi-Registry, koishi-plugin-vrchat-status | 标注"基于上游优化" |

## 运维交付

| 技能维度 | 证据强度 | 画像来源 | 源码验证仓库 | 边界约束 |
|---|---|---|---|---|
| Dockerfile 编写 | 高 | backend-05 | YoOSF-API, Koishi-Mirror, ip-api | — |
| docker-compose.yml | 高 | backend-05 | Koishi-Mirror, YoOSF-API | — |
| systemd unit | 高 | backend-05 | YoOSF-API, Koishi-Mirror, n2n-go | — |
| 构建/发布脚本 | 高 | backend-05 | Koishi-Mirror, YoOSF-API | — |
| .env.example 模式 | 高 | backend-01 | 全部 Go 项目 | — |
| 健康检查模板 | 中 | backend-05 | 4+ 仓库 | — |

## 硬约束（从边界文件提取）

| 约束 | 来源 |
|---|---|
| 禁止使用：精通、架构师、企业级成熟、生产级大规模 | 02-简历表达边界.md |
| xi_cloud_disk、Xi_Oopz 只能写"参与/PR 贡献" | 06-职责边界.md, 06-fork边界.md |
| Koishi-Registry 标注"基于上游项目自定义优化" | 06-fork边界.md |
| YoSpace 保留 UI/UX 来源声明 | 06-fork边界.md, 02-简历表达边界.md |
| YoCWAF 不写成"成熟 WAF" | 06-职责边界.md |
| QPS/压测数据标注"README记录 / 压测结果" | 02-简历表达边界.md |
| fork 项目不写成原创 | 06-职责边界.md |
