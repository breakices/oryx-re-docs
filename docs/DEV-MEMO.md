# Oryx-RE Dev Memo

> 开发纪律。不进 CLAUDE.md(那里是全局工作空间约定);这里专门收 Oryx 项目内部的 wiki 化条款。
>
> 索引:
>   - **Q4** · KV cache prefix discipline(byte stability 准则)
>   - **Q5** · Feature submission checklist
>   - **Q6** · API versioning policy
>   - **Q7** · Backend feature package + release gate
>   - **Q8** · Mode / workflow / skill / tool boundary
>   - **Q9** · Finance provider and data tool discipline

---

## Q4 · KV cache prefix discipline

### 哪些字符串是 cache prefix

每个 turn 的请求 prefix 由这些拼起来,顺序固定。**任何 byte 变更都会让所有现存 session 的 KV cache 失效一次**(重建后稳定)。

| 锚 | 文件 | 长度量级 |
|---|---|---|
| SYSTEM_PROMPT | `backend/app/features/chat/constants.py` | ~2-3 KB |
| `COMPACT_PROMPT` | 同上 | ~1 KB |
| `SUB_SYSTEM_PROMPT` | `backend/app/features/sub_agent/service.py` | ~1.5 KB |
| 工具 schema(`spawn_sub_agent` / `run_sandbox` / ...) | 各 `features/*/tool.py` + `features/sub_agent/tool.py` | 序列化后 ~10-20 KB |
| `REASONING_COMPACTED` placeholder | `features/chat/constants.py` | 短 |
| compact summary 模板 | `features/compact/__init__.py` | 中 |

### 什么变更安全

- **加新工具 schema**:append 到 schema list 尾部 — prefix 前缀稳;新工具只影响之后那个 turn 的部分,**已有 session 不破 cache**。
- **改某 tool description 一个字**:破自该工具被序列化到 schema list 的位置之后的所有字节 — 破 cache。
- **改 SYSTEM_PROMPT 一个字**:破整个 prefix(系统 prompt 是最前面的锚)— 全破。

### 工程纪律(commit message convention)

任何对**上面 6 个锚**之一的字节变更,commit 标记必须包含:

```
prompt-cache-break: <reason>
```

例:
```
prompt-cache-break: tighten suggest_options narration rule
prompt-cache-break: add new optional field to spawn_sub_agent schema
```

这个标签**不阻塞**(我们要的是显式而非禁止),用于:
- 上线后看 KV cache miss spike 时,git blame 能直接对上
- 评估"这次破 cache 值不值得",future-proof 决策记录

### 工程化检查(可选,未做)

后续若想自动化,可加 `scripts/prefix_hash_check.py`:计算当前 6 个锚的 hash 串,跟 `.cache-hashes.json`(上次 commit 的)比较,变化时 `print` warning(不 fail PR)。**未做,等真出现"莫名其妙 cache miss"问题再做**。

### 不该做的

- 不要在 prompt 里塞 per-turn variable 数据(时间戳、user_id),那会让每个 turn 的 prefix 都不同,完全杀掉 cache 价值。所有 per-turn 信息应该在 user message body 里,不在 system / 工具 schema 里。

---

## Q5 · Feature submission checklist

新 feature 提交(`features/<x>/`)前,**作者自查**:

```
☐ router 每个 endpoint 都有 response_model(D 之后是硬要求;CI 闸已捕获 schema drift 但漏掉的 endpoint 会以 `dict` 出现在 schema.gen.ts)
☐ 关键 path 用 log.info / log.warning(structlog),key 字段含:
    - trace_id(从 ContextVar 自动接;不用手写)
    - session_id(若是 session-scoped)
    - turn_id / bg_task_id(若适用)
☐ 有 bg 活动的 emit bg_event(drawer 可见);非 silent 的标 urgency=notify
☐ settings 新字段写注释 + 默认值合理 + dev / prod 默认意图明确
☐ alembic 迁移:upgrade + downgrade 都写;PR diff 包含 `0XXX_*.py`
☐ 错误响应统一走 HTTPException(detail=str | dict)— FastAPI 全局 handler 会包装成 ErrorEnvelope
☐ 若新增 wire 字段:跑 `npm run gen:api:offline` + 提交 schema.gen.ts diff
☐ 若改 prompt prefix(SYSTEM_PROMPT / 工具 schema description):commit message 带 `prompt-cache-break:` 标签(见 Q4)
☐ 触摸到 Session / Turn / Memory 写规则:走 domain entity 方法,不直接抓 ORM column
```

### Reviewer 视角

- PR diff 第一眼看 alembic 是否单 head:`uv run alembic heads`
- 看 schema.gen.ts 有没有 drift:CI 已闸
- 看 feature 的 `schemas.py` 是否新增了视图模型(D 之后是常态)

---

## Q6 · API versioning policy

### 当前

所有 endpoint 在 `/api/v1/` 前缀下(2026-05-21 起)。前端 `client.ts` 通过 `API_BASE` 拼;nginx 反代同步;vite dev proxy `/api/v1/`。

### 破坏性变更 vs 兼容变更

| 变更类型 | 是否破坏 | 怎么走 |
|---|---|---|
| **加字段**(wire 响应新增 nullable / optional field) | 兼容 | 直接改 v1 |
| **删字段** | 破 | 必须开 v2 |
| **改字段语义**(同字段名换含义) | 破 | 必须开 v2;不允许在 v1 上 silent change |
| **改字段类型**(`string` → `number`) | 破 | 必须开 v2 |
| **加 endpoint** | 兼容 | 直接加 v1 |
| **删 endpoint** | 破 | v1 上 410 Gone,v2 干净开 |
| **改 endpoint 路径** | 破 | v2;v1 保留旧路径 |
| **改错误响应格式** | 兼容(ErrorEnvelope 加字段)/ 破(改 envelope shape) | 加字段不算破;改 shape 必须 v2 |

### v2 触发条件

不主动上 v2。**只有出现以下之一时才考虑**:
- 累积破坏性变更已经 ≥ 3 个,在 v1 上发不了
- 某个核心数据流(如 SSE chunk 协议)需要 redesign

### v2 上线流程

1. 新建 `/api/v2/` 前缀 router
2. 各 endpoint 重新挂在 v2 下
3. 前端切到 v2(同 PR / 同 release)
4. v1 endpoint 标 `deprecated=True` 但保留 6 个月
5. 6 个月后真正删除 v1

### 当前不维护多版本并存

只有 v1 一个版本。半年内不会主动上 v2(写入策略本身就是为了避免乱开)。

---

## Q7 · Backend feature package + release gate

### 目标

新后端能力默认以 `backend/app/features/<name>/` 为边界提交,避免再把逻辑塞回
`main.py` / `chat/turn.py` / `sub_agent/service.py` / 共享 tool 巨型文件。

### 标准目录

```
backend/app/features/<name>/
├── __init__.py          # import-time registration only;不要放业务逻辑
├── router.py            # HTTP edge: auth / parse / response_model / delegate
├── schemas.py           # Pydantic wire model;前端类型来源
├── service.py           # feature 对外公开 API;跨 feature 只 import 这里
├── tool.py              # LLM Tool / FormTool 注册(如需要)
├── events.py            # event_bus subscriber / publisher(如需要)
├── jobs.py              # Job / bg worker(如需要)
└── README.md            # 功能开关 / 接入点 / 验收步骤
```

小 feature 可以省略不存在的文件,但不要把多个 feature 混进同一个 `tool.py` /
`router.py`。跨 feature 调用只能走:

- 对方 `service.py` 里明确公开的函数
- `kernel.event_bus`
- `kernel.pipeline`
- `infra.state`
- DB repo + domain entity

禁止直接 import 对方的 live runner dict、私有 `_do_*`、内部 prompt builder。

### Release gate

任何可隐藏发布的新能力必须同时带:

```
☐ settings feature flag(默认 false 或 dev-only true,注释写明 dev/prod 意图)
☐ router / tool 在关闭时返回稳定错误或 disabled payload
☐ 前端入口可不做,但后端 API 必须能在 flag off 下安全存在
☐ OpenAPI schema 已生成/校验
☐ 手动验收路径写入 feature README 或 PR 描述
```

配置命名:

- `FEATURE_<NAME>_ENABLED` 用于纯功能开关
- `<DOMAIN>_ENABLED` 用于已有 domain 级能力,例如 `SANDBOX_ENABLED`
- 关闭原因用 `<FLAG>_DISABLED_REASON`,让 agent 能如实告知用户

### 敏捷切片

后端新功能按 vertical slice 合入主干:

1. schema + flag + 空 service(可导入,默认 disabled)
2. router/tool 最小接线
3. service 真实逻辑
4. events/jobs/observability
5. flag 打开与发布记录

每片都必须可启动、可回滚;不要攒一个长期悬挂的大分支。

---

## Q8 · Mode / workflow / skill / tool boundary

### 全局模式

工作区模式是 workspace 级 profile,不是单条消息开关。它可以影响:

- 可见 workflow / skill catalog
- 默认 memory source
- 右侧 workspace 工作台
- provider 状态面板
- skill gate 过滤
- 历史缓存命中预期

切换模式时,前端必须提示"可能导致当前工作区对话无法命中历史缓存,产生额外费用"。

### 边界

| 概念 | 作用 | 是否给模型 |
|---|---|---|
| `workflow.md` | 用户可选的高层任务编排入口 | 主 agent 只看 public/main 摘要;executor 细节按需给 sub-agent |
| `skill.md` | 可执行步骤 / provider 指南 / 模式约束 | 可进入模型上下文 |
| `ToolMeta` | UI / 审计 / usage / 后续权限治理 | 不进入模型上下文 |
| tool schema | 模型可调用的稳定工具契约 | 进入 OpenAI `tools[]` |

### 纪律

- 用户显式选择 workflow 时,后端跳过 skill gate,直接预加载对应 workflow。
- workflow 的内部能力用 `internal_skill_ids` 指向 skill;不要让主 agent 再二次 search skill。
- `visibility: workflow_internal` 的 skill 不进入普通 skill gate。
- provider skill 用 `visibility: provider`,只说明如何使用稳定 Oryx data tools,不暴露 provider 凭据。
- `ToolMeta` 只能附在 SSE / 审计 / usage 面;不得写入 prompt、tool result、skill body 或 OpenAI schema。
- 新增 workflow/skill 后必须跑 `cd backend && uv run python scripts/check_skill_registry.py`。

---

## Q9 · Finance provider and data tool discipline

### 稳定接口

LLM 只调用稳定 Oryx finance tools:

- `finance_provider_status`
- `finance_quote`
- `finance_news`
- `finance_search_data`
- `finance_statement`

Provider 选择、鉴权、fallback、usage 记录都在 `features/finance_data/` 内部处理。

### 新 provider checklist

```
☐ settings/env example 已补,默认 disabled 或无 key 不 ready
☐ auth resolver 独立,不要把 token 交换写在 tool.py
☐ provider adapter 返回统一 ProviderResult
☐ health/status 能说明 disabled / missing_key / ready / failed
☐ 每次请求记录 finance_provider_usage
☐ provider skill 只写使用指南,不写密钥、不写临时 token
☐ 无真 key 联调的 adapter 保留 TODO,不得伪装 ready
```

### 失败语义

- provider 未开启:返回稳定 `disabled` payload。
- provider key 缺失:返回 `missing_key`,让 agent 改用可用 provider。
- provider 请求失败:返回 `failed` 并记录 usage/error,不得吞错后输出空成功。
- 前端 provider 面板只展示状态和 usage 摘要,不展示密钥或 token。

---

## Q10 · Finance memory / graph discipline

Phase 4 金融记忆不复制 Knevo 的"每用户一个 SQLite"形态。Oryx 的默认规则:

- `features/finance_memory/` 是唯一业务入口;不要把金融记忆 CRUD 写回 `finance/` 或通用 `memory_store/`。
- Oryx DB 是 `finance_memory_records`、entities、edges、facts、gaps、ingestion jobs 的事实源。
- 文件系统只放原始文件、解析 chunk、导出包、可重建索引;REST/tool 不暴露物理路径。
- 未来 Neo4j/Kuzu/Graphiti 只能做 derived index。canonical id、source refs、version、deleted_at 必须留在 Oryx DB。
- 模型不得拿全量节点表。写入事实前用 `finance_entity_resolve` 取 top-k 候选,再通过 `finance_memory_write` 写入 entity refs。
- `finance_graph_context` 先走 SQL graph,空结果写 `finance_memory_gaps`,前端 Gaps 面板展示这些缺口。
