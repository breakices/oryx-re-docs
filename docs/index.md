# Oryx-RE 开发文档

> 长程科研任务 Agent。前后端单仓：React + zustand 前端，FastAPI + SQLAlchemy 2.0 (async) 后端，DeepSeek 主/light 模型双通道，Tavily 检索，Docker 沙箱跑代码。内置通用研究模式与金融投研模式。

本站汇总 Oryx-RE 的架构说明、开发纪律与源码导读，方便随时查阅（含手机端）。

## 从这里开始

<div class="grid cards" markdown>

- :material-map: **[架构路线](ROADMAP.md)**

    已落地 / 计划中 / 待定的唯一事实源。

- :material-clipboard-check: **[开发纪律](DEV-MEMO.md)**

    KV cache 字节稳定性、feature 提交清单、API 版本策略等硬约束。

- :material-rocket-launch: **[部署指南](DEPLOY.md)**

    单机 docker-compose：FastAPI + Postgres + ClickHouse + nginx + sandbox。

- :material-book-open-page-variant: **[源码导读](source-reading-guide.html)**

    逐文件阅读路线，覆盖 ~275 个源文件，支持筛选与「已读」标记。

</div>

## 技术栈速览

| 层 | 选型 |
|---|---|
| **前端** | React 19、zustand、Vite 8、TypeScript 6、react-markdown + KaTeX + highlight.js |
| **后端** | FastAPI、SQLAlchemy 2.0 async、Alembic、asyncpg / aiosqlite |
| **类型契约** | `openapi-typescript` 把 FastAPI `/openapi.json` 直转 `src/api/schema.gen.ts` |
| **观测** | ClickHouse turn trace（prod）/ FileSink（本地） |
| **沙箱** | docker CLI；`network=none` + cap-drop + uv 缓存离线装包 |
| **鉴权** | PBKDF2 + session cookie + 邀请码注册；admin 可借身份 |

## 后端六区结构

内核 / domain / capabilities / infra / features / jobs。每个 feature 在 `features/<name>/` 自治，通过 `main.lifespan` 显式 import 触发自注册。详见 [架构路线](ROADMAP.md) 与 [v1 重构记录](REFACTOR-260520.md)。

---

!!! tip "本站如何维护"
    所有页面源文件在本仓库的 `docs/` 目录下，是纯 Markdown。改完 push 到 `main` 分支，GitHub Actions 会自动构建并发布。本地预览：`mkdocs serve`。
