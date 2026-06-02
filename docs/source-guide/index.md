# 源码导读：总览与阅读路线

逐文件的源码阅读地图，覆盖约 **275 个源文件**、整理成 **109 张文件卡**，分 10 个模块。每张卡含「作用 / 关键内容 / 优先级 / 依赖」。

!!! tip "如何在阅读时做笔记"
    每张文件卡下方都留了一行 HTML 注释占位 `<!-- ✍️ 我的笔记（手动补充）： -->`。
    你读完对应核心代码后，把它替换成自己的理解即可（比如调用链、坑、TODO）。
    注释不会在网页上渲染，但编辑 `.md` 时一眼可见。改完 push 到 `main`，站点自动更新。

优先级图例：🔴 核心（必读）· 🟡 重要 · ⚪ 次要（随用随查）。

## 🗺️ 总阅读路线（先骨架 → 数据流 → 外围）

| 阶段 | 内容 | 模块 |
|---|---|---|
| **0** | **准备**：先读项目自带文档 [架构路线](../ROADMAP.md) / [开发纪律](../DEV-MEMO.md)，建立「内核 + features 六区」心智模型 | — |
| **1** | **后端地基与内核**：config / trace / main 启动编排；工具协议、注册表、流式 LLM 单 iter、pipeline 钩子、runner、SSE 协议 | [01 内核与启动骨架](01-g1.md) |
| **2** | **数据层**：domain 纯实体 + 状态机（Turn/BgTask FSM）与 db ORM/repo/mapping | [02 数据层](02-g2.md) |
| **3** | **横切关注与基础设施**：capabilities（ContextVar 身份/权限/审计）+ infra（state 六子协议、LLM 通道、sandbox、trace sink、websearch） | [03 横切与基础设施](03-g3.md) |
| **4** | **核心对话（应用心脏）**：chat/turn.py 主循环、SYSTEM_PROMPT 锚点、消息装配、compact 压缩 | [04 核心对话编排](04-g4.md) |
| **5** | **Agent 与工具执行**：sub_agent 派生/流式、sandbox docker 执行、shell 九层护栏、web 工具、FormTool 与 pre_inject | [05 Agent 与工具执行](05-g5.md) |
| **6** | **通用功能与平台**：workspace 文件、memory、skills/workflows、四个 lite 模型特性、admin/auth/impersonation | [06 通用功能 / Lite / 平台](06-g6.md) |
| **7** | **金融子系统**：finance 模式、finance_data（provider 适配器 + auth 解析器）、finance_memory（实体图谱）、finance_portfolio | [07 金融子系统](07-g7.md) |
| **8** | **前端核心**：types / zustand store / api 层 / 两条 SSE 流 hooks | [08 前端核心](08-g8.md) |
| **9** | **前端组件与 lib**：AppShell/ChatPane/Message/Workspace 四大件 + merge/markdown 两个核心 lib | [09 前端组件与 lib](09-g9.md) |
| **10** | **内容与样式**：skills/workflows 的能力卡、样式系统、根配置、docs 文档 | [10 内容 / 样式 / 配置](10-g10.md) |

## 模块清单

- [01 · 内核与启动骨架](01-g1.md) — 16 卡 · `backend/app` kernel / 入口 / lib / jobs
- [02 · 数据层：领域实体 + 持久化](02-g2.md) — 15 卡 · domain / db
- [03 · 横切关注 + 基础设施](03-g3.md) — 11 卡 · capabilities / infra
- [04 · 核心对话编排（应用心脏）](04-g4.md) — 9 卡 · chat / compact
- [05 · Agent 派生与工具执行](05-g5.md) — 10 卡 · sub_agent / sandbox / shell / web / pre_inject
- [06 · 通用功能 / Lite 特性 / 平台](06-g6.md) — 10 卡 · workspace / memory / skills / lite / auth
- [07 · 金融子系统](07-g7.md) — 9 卡 · finance / finance_data / finance_memory / finance_portfolio
- [08 · 前端核心：状态 / API / 两条 SSE 流](08-g8.md) — 12 卡 · store / api / hooks
- [09 · 前端组件与 lib 工具](09-g9.md) — 12 卡 · components / lib
- [10 · 内容 / 样式 / 配置 / 文档](10-g10.md) — 5 卡 · skills / workflows / styles / docs

!!! warning "阅读时留意的一个坑"
    `domain/memory.py`（领域层仍含 `session` scope + `summary/raw_content` 字段）与 `db/models.py` 的 Memory v2（仅 `workspace|user` scope，改用 `confidence/entity_refs`）存在字段错位 —— 渐进式重构的遗留不一致点。详见 [02 数据层](02-g2.md)。
