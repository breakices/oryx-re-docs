# Oryx-RE 重构计划 — 2026-05-20

> 一次性梳理：当前组织结构的问题、目标结构、分阶段落地路径。
> 与 `ROADMAP.md` 关系：ROADMAP 管"做什么 feature"，本文管"代码长什么样"。
> 两份文档独立维护；feature 落地后**先看本文是否需要补 hook / channel / event 抽象，再开工**。

---

## 第一部分 · 问题发现

### 1. 后端组织的结构性问题

#### 1.1 单一维度分层，feature 没有"家"

当前组织是「按技术层」单维度：

```
routers/  →  services/  →  tools/  →  db/repo/  →  trace_sink/
```

适合早期主线 ≤5K LoC。但**每个新 feature 都横切多个层**：

- 加一个 lite-model 检查（next_options / 新触发器）→ 同时碰 `routers/chat.py` + `services/lite/` + `services/chat/turn.py` 加分支 + `tools/registry.py`
- 加一个工具 → `tools/registry.py` 加 schema + dispatch 分支 + 可能改 `services/chat/turn.py` 接入

**结果**：feature 实现散落 3-5 个文件，回头追代码要跨整个 tree。

这是 ROADMAP 增长到当前规模后必然结构混乱的根因。

#### 1.2 四个"上帝文件"

| 文件 | 行数 | 主要职责 |
|---|---|---|
| `tools/registry.py` | 877 | TOOL_SCHEMAS 列表 + 全部工具的 if-elif dispatch |
| `bg_runner.py` | 967 | sub-agent 启动、LLM 流式循环、emit 处理、resume |
| `services/chat/turn.py` | 850 | run_turn 主循环 + deferred 派发 + compact 链 + trace |
| `process_runner.py` | 564 | docker sandbox 启动 + 流式 stdout 解析 + emit |

合计 3258 行。每个都过了"一次性读完才敢改"的认知阈值。

`turn.py::run_turn` 单函数从 line 273 到 824——一个函数 550 行做：lite gate 加载、SOFT compact、history rebuild、pre-inject、4 类 hint、流式 LLM、tool dispatch、deferred 派发、HARD compact、pending B chain、trace emit。

#### 1.3 两套相近但不同的 LLM 流式循环

`run_turn` 和 `_run_sub_turn` 是平行的两套实现，各自重写：

- retry + transient 检测
- stream parse（usage / reasoning_content / content / tool_calls）
- cancel check 安排
- 工具 dispatch 后的 DB 落盘
- trace emit

注释中写了"试图共享会留死代码"——这个判断在 V1 阶段成立，但**当前两边都已稳定，应抽公共 helper**。任何流处理的 bug 都要修两遍。

#### 1.4 工具系统的两个表达性盲区

(a) **dispatch 是手写 if-elif 链**（registry.py line 725-871）。每加一个工具改两处（SCHEMAS list + dispatch 分支）。改成 `dict[str, Callable]` 是机械重构。

(b) **没有"结构化输出"概念**。当前只有两类 LLM 产出语义：`content`（散文）和 `tool_calls`（有副作用、要回灌）。但 `next_options` 这类「模型给 UI 的结构化提示，不回灌」无处安放——硬塞工具就要白白走一轮 LLM 调用。

#### 1.5 进程内单例状态散布

- `runner._runners` (turn runners)
- `bg_runner._runners` (sub-agent runners)
- `bg_hub` 的 waiters / signal 槽
- `_digest_locks` per session lock
- `TurnRunner.deferred_calls` per-turn

全部 module-level dict / lock，单进程 OK，**水平扩容前所有这些都要换成外部存储**。ROADMAP 已点明"Phase B 换 Redis"。

#### 1.6 模块边界已经开始模糊

- `bg_emit.py` 调 `services.chat.maybe_spawn_digest_turn`（函数内 import 避免循环）
- `services/chat/turn.py` 调 `bg_emit`
- `bootstrap.py` 调 `app.skills_loader` 调 `app.services.lite.skill`
- `tools/bg.py` 调 `bg_runner.spawn` 调 `services.chat.maybe_spawn_digest_turn`

能跑，但已经出现"靠 lazy import 解循环"的征兆。说明 chat / sub-agent / process 三方实际是同级 feature，不应该有谁是"主"。

#### 1.7 缺自动化测试

仓库无 `tests/`。ROADMAP 的「测过/未测」表用 session id 列出来——是手工集成测。

对一个 9.6K LoC、改一处 KV cache 策略可能影响整个 agent 行为的后端，这是最大的工程债务。

---

### 2. 前端组织的问题

#### 2.1 大文件

| 文件 | 行数 | 备注 |
|---|---|---|
| `Sidebar.tsx` | 807 | session 列表 + bg badge + workspace + auth 全部一份 |
| `Workspace.tsx` | 790 | tree / 文件预览 / 上传 / 拖拽全部一份 |
| `Composer.tsx` | 705 | mention / slash / cite chip / 附件 / 队列 / 键盘事件 |
| `api/client.ts` | 575 | REST + SSE + Auth + mock 切换 |
| `hooks/useChat.ts` | 407 | consumeStream 130+ 行 switch |

#### 2.2 单 mega zustand store

`store.ts` 一个文件管 messages / bg / memory / workspace / queue / pagination / cite / tz。300 行还撑得住，再加 1-2 个 domain 必须拆。

#### 2.3 SSE 消费的大 switch

`useChat.ts::consumeStream` 一个内联 switch 处理 15+ 种 chunk type。加新 chunk → 多一个 case。没有 dispatch table、没有 exhaustiveness 检查。

#### 2.4 没有路由

纯 store-driven。深链需求出现时（分享某个 turn / bg_task）改造成本高。

#### 2.5 流式结束后全量 listMessages 重拉

`useChat.ts` 在每次 turn 流结束后重新拉一遍当前窗口消息再 `setMessages(mergeMessages(...))`。注释承认这有 ordering / pre-inject 时序的坑。**双渲染**：流式过程是一种状态，重拉是另一种，肉眼可能看到瞬间跳变。

#### 2.6 Mock 路径写死个人目录

`api/client.ts` 里 `MOCK_ROOT = '/Users/flint/Projects/Oryx-RE/.workspace-mock'`——dev-only 但不干净。

---

### 3. 跨前后端的类型契约漂移

| 类型 | 当前来源 | 漂移风险 | 后端 schema |
|---|---|---|---|
| REST 请求/响应 | 前端手写 in `client.ts` / `types.ts` | 中 | 有 Pydantic |
| **SSE chunk** (15+ 种) | 前端手写联合类型 | **高** | **无** — 全是裸 dict |
| 域对象 (Message / FsNode / BgTask 等) | 前端手写 in `types.ts` | 中 | 部分有 |

**SSE 才是真正的雷**：后端 `runner.publish({"type": ..., ...})` 散落十几处，零校验。前端 `Chunk` 联合类型靠记忆同步。加新 chunk type 而前端漏 case 是静默 bug。

---

### 4. Agent 上下文管理的次级问题

主要逻辑已经做得很好（见审视报告），次级遗留：

- `MAX_LOOPS = 12` 偏紧，撞顶无优雅降级
- compact 是单次 LLM 调用，summary 质量 + 恢复种子选择都无评分门控
- memory ranker 没有向量索引也没用 hit_count
- `inherit` 模式 sub-agent 复制父历史，token 成本可叠加
- 每个新用户消息都跑 4 个 lite call，即使 "好的"（已经识别 simple_reply，但没 fast-path）
- 几乎全部强绑 DeepSeek（reasoning_content 协议、cache_hit/miss 字段、1M context 假设）

这些不在本文重构范围，但应在 ROADMAP 跟进。

---

## 第二部分 · 目标结构

### 2.1 三维正交组织

```
                ┌─ kernel       (内核：流式 LLM 循环 + 扩展点)
垂直能力 ───────┼─ domain       (长期数据模型)
(feature)       └─ infra        (与外部世界的契约)

横切能力 ──── capabilities      (context-bound: auth / audit / rate-limit)
异步反应 ──── events / jobs     (解耦的反应链)
```

**核心原则**：

1. **kernel 不知道 feature**——内核只暴露扩展点（Stage / FormTool / ContentChannel / EventBus）
2. **feature 不知道彼此内部**——feature 之间只通过对方的顶层 service 函数 / event 通信
3. **infra 可换实现**——LLM / Sandbox / Trace sink 全走 Protocol
4. **capabilities 用 ContextVar 透传**——不污染 feature 签名

### 2.2 完整目录树

```
backend/app/
│
├── main.py                          仅 FastAPI app + lifespan
├── config.py
│
├── kernel/                          内核，>6 个月不动
│   ├── llm_loop.py                  StreamingLLMLoop（公共流式循环）
│   ├── pipeline.py                  Stage 类型 + 注册中心 (PreTurn/PreIter/PostIter/PostTurn)
│   ├── tool_protocol.py             Tool / FormTool 抽象基类
│   ├── tool_registry.py             dict[str, Tool|FormTool] + register 装饰器
│   ├── content_channels.py          (Phase 6+) fenced block 流式解析
│   ├── sse_protocol.py              所有 SSE Chunk 的 Pydantic 联合（discriminator）
│   ├── event_bus.py                 in-process pub/sub
│   ├── runner_registry.py           TurnRunner / SubAgentRunner 注册（合并）
│   └── sse.py                       SSE helper
│
├── domain/                          长期数据模型
│   ├── models.py                    SQLAlchemy ORM（现 db/models.py）
│   └── types.py                     跨模块共享的 TypedDict / Enum
│
├── infra/                           外部世界的契约
│   ├── db/
│   │   ├── engine.py
│   │   └── repo/                    现 db/repo/
│   ├── llm/
│   │   ├── deepseek.py              现 llm.py
│   │   └── light.py                 现 services/lite/_client.py
│   ├── websearch/                   现 services/websearch/
│   ├── sandbox/
│   │   ├── protocol.py              SandboxBackend Protocol
│   │   └── local_docker.py          LocalDockerBackend 实现
│   ├── trace_sink/                  现 trace_sink/
│   └── object_store/                (将来) 大附件
│
├── features/                        每个 feature 一个垂直目录
│   │
│   │  ─── 现有主线 ───
│   ├── chat/                        主对话编排
│   │   ├── router.py                现 routers/chat.py
│   │   ├── service.py               现 services/chat/turn.py 瘦身后的 100 行
│   │   ├── messages.py              现 services/chat/messages.py
│   │   ├── constants.py             SYSTEM_PROMPT
│   │   └── schemas.py
│   │
│   ├── compact/
│   │   ├── service.py               现 services/chat/compact.py
│   │   ├── hooks.py                 PreTurn(SOFT) + PostIter(HARD) hook 注册
│   │   └── prompts.py               COMPACT_PROMPT
│   │
│   ├── pre_inject/
│   │   ├── service.py               现 bootstrap.py 里的 persist_*_cycle
│   │   └── hooks.py                 PreTurn hook
│   │
│   ├── triage_guard/                现 services/lite/triage.py
│   ├── memory_ranker/               现 services/lite/memory.py
│   ├── skill_gate/                  现 services/lite/skill.py
│   ├── spawn_advisor/               现 services/lite/insight.py
│   │
│   ├── sub_agent/                   现 bg_runner.py（拆分）
│   │   ├── service.py               run_sub_turn（~200 行）
│   │   ├── router.py                routers/bg.py 中 sub-agent 部分
│   │   ├── hub.py                   现 bg_hub.py
│   │   ├── emit.py                  现 bg_emit.py
│   │   ├── prompts.py               SUB_SYSTEM_PROMPT
│   │   └── tool.py                  spawn_sub_agent / get_bg_task / wait_for_*
│   │
│   ├── sandbox_runner/              现 process_runner.py（拆分）
│   │   ├── service.py
│   │   ├── router.py
│   │   └── tool.py                  run_sandbox
│   │
│   ├── workspace/
│   │   ├── router.py
│   │   ├── service.py
│   │   ├── tool.py                  现 tools/fs.py
│   │   └── hooks.py                 (Phase 8) FolderUploaded event 发布
│   │
│   ├── memory_store/
│   │   ├── router.py
│   │   └── tool.py
│   │
│   ├── skills/
│   │   ├── router.py
│   │   ├── loader.py                现 skills_loader.py
│   │   └── tool.py
│   │
│   ├── shell_tool/                  现 tools/shell.py
│   ├── web_tool/                    现 tools/web.py
│   │
│   ├── auth/                        登录 / 注册 / invite
│   ├── admin/                       admin 面板 endpoints
│   │
│   │  ─── 以下是 bonus，0 行新文件碰旧文件 ───
│   ├── next_options/                bonus 1
│   │   ├── form_tool.py             SuggestOptionsTool(FormTool)
│   │   └── __init__.py              register() 加 SYSTEM 提示
│   │
│   ├── auto_research/               bonus 2
│   │   ├── router.py                upload endpoint
│   │   ├── service.py
│   │   ├── events.py                订阅 FolderUploaded
│   │   ├── jobs.py                  FolderIndexerJob
│   │   └── prompts.py
│   │
│   └── impersonation/               bonus 3
│       ├── router.py                admin 借身份 endpoint
│       └── service.py
│
├── capabilities/                    横切（context-bound）
│   ├── auth_context.py              AuthContext + dep（替换 current_user）
│   ├── permission.py
│   ├── audit.py                     impersonation 期自动 hook DB write
│   └── rate_limit.py
│
├── jobs/                            跨 turn 的长期 worker
│   └── runtime.py                   Job ABC + 调度
│
├── api/
│   ├── __init__.py                  FastAPI app 构造 + 路由聚合
│   └── middleware.py                trace_middleware
│
└── lib/                             保留
    ├── retry.py
    └── sanitize.py
```

### 2.3 前端目标结构

```
src/
├── api/
│   ├── client.ts                    瘦身：仅 authFetch + sse helper
│   ├── schema.gen.ts                ★ openapi-typescript 生成（含 Chunk 联合）
│   ├── rest.ts                      所有 REST 调用，类型从 schema.gen 引入
│   └── sse.ts                       SSE generator + dev-mode Zod 校验
│
├── stores/                          ★ 拆 store
│   ├── chat.ts                      messages / turn / queue / activeTurnId
│   ├── bg.ts                        bgTasks / bgEvents
│   ├── workspace.ts                 fs tree / mode
│   ├── memory.ts
│   └── ui.ts                        rewindFrom / pendingCite / pagination
│
├── features/                        ★ 与后端 features 对应
│   ├── chat/
│   │   ├── ChatPane.tsx
│   │   ├── Composer.tsx             拆分后
│   │   ├── Message.tsx
│   │   ├── ToolStream.tsx
│   │   ├── SelectionBubble.tsx
│   │   └── useChat.ts
│   ├── sidebar/
│   │   ├── Sidebar.tsx
│   │   ├── SessionList.tsx
│   │   └── WorkspaceSwitch.tsx
│   ├── workspace/
│   │   ├── Workspace.tsx            拆分后
│   │   ├── FileTree.tsx
│   │   └── FilePreview.tsx
│   ├── bg/
│   │   ├── BgDrawer.tsx
│   │   ├── BgHeaderStrip.tsx
│   │   └── useSessionEvents.ts
│   ├── auth/
│   │   ├── LoginPanel.tsx
│   │   └── AuthGate.tsx
│   └── compact/
│       └── ...
│
├── kernel/                          ★ 共用
│   ├── markdown.ts
│   ├── merge.ts                     mergeMessages
│   ├── extractRich.ts
│   ├── time.ts
│   └── dialog.ts
│
└── App.tsx / main.tsx / styles.css
```

---

## 第三部分 · 文件迁移对照表

| 当前位置 | 新位置 | 备注 |
|---|---|---|
| `runner.py` | `kernel/runner_registry.py` | 与 `bg_runner._runners` 合并 |
| `sse.py` | `kernel/sse.py` | |
| `bg_hub.py` | `features/sub_agent/hub.py` | |
| `bg_emit.py` | `features/sub_agent/emit.py` | |
| `bg_runner.py` ★ | `features/sub_agent/service.py` + `kernel/llm_loop.py` | 拆分 |
| `process_runner.py` ★ | `features/sandbox_runner/service.py` | |
| `sandbox.py` | `infra/sandbox/{protocol,local_docker}.py` | 拆 |
| `llm.py` | `infra/llm/deepseek.py` | |
| `skills_loader.py` | `features/skills/loader.py` | |
| `workspace.py` | `features/workspace/service.py` | |
| `auth.py` | `capabilities/auth_context.py` + `features/auth/` | 拆 |
| `services/chat/turn.py` ★ | `features/chat/service.py`（~100行）+ `kernel/llm_loop.py`（~200行）+ 各 feature hook | **核心拆分** |
| `services/chat/compact.py` | `features/compact/service.py` + `hooks.py` | |
| `services/chat/bootstrap.py` | `features/pre_inject/` + 4 个 lite gate feature | 拆 |
| `services/chat/messages.py` | `features/chat/messages.py` | |
| `services/chat/constants.py` | `features/chat/constants.py` + 各 feature prompt | 拆 |
| `services/lite/triage.py` | `features/triage_guard/` | |
| `services/lite/memory.py` | `features/memory_ranker/` | |
| `services/lite/skill.py` | `features/skill_gate/` | |
| `services/lite/insight.py` | `features/spawn_advisor/` | |
| `services/lite/_client.py` | `infra/llm/light.py` | |
| `services/lite/shell.py` | `features/shell_tool/lite.py` | |
| `services/websearch/` | `infra/websearch/` | |
| `services/vision.py` | `infra/llm/vision.py` | |
| `tools/registry.py` ★ | **完全消失** | TOOL_SCHEMAS 散到各 feature；dispatch 改 `kernel/tool_registry.py` |
| `tools/fs.py` | `features/workspace/tool.py` | |
| `tools/shell.py` | `features/shell_tool/tool.py` | |
| `tools/web.py` | `features/web_tool/tool.py` | |
| `tools/bg.py` | 拆 `features/sub_agent/tool.py` + `features/sandbox_runner/tool.py` | |
| `tools/memory.py` | `features/memory_store/tool.py` | |
| `tools/skill.py` | `features/skills/tool.py` | |
| `trace_sink/` | `infra/trace_sink/` | |
| `db/engine.py` | `infra/db/engine.py` | |
| `db/models.py` | `domain/models.py` | |
| `db/repo/` | `infra/db/repo/` | |
| `routers/chat.py` | `features/chat/router.py` | |
| `routers/auth.py` + `routers/invites.py` | `features/auth/` | |
| `routers/admin.py` | `features/admin/router.py` | |
| `routers/bg.py` | 拆 `features/sub_agent/router.py` + `features/sandbox_runner/router.py` | |
| `routers/memory.py` | `features/memory_store/router.py` | |
| `routers/workspace.py` | `features/workspace/router.py` | |
| `routers/skills.py` | `features/skills/router.py` | |
| `main.py` | `main.py`（瘦） + `api/__init__.py` | lifespan 留 main，路由聚合移走 |

---

## 第四部分 · 分阶段落地路径

每个阶段独立可上线，可在任何节点停下不影响主线。

### Phase 0 · 基线测试（开始前 + 持续）

> 没有这一步，后续每个阶段都是冒进。

- [ ] 把 ROADMAP「测过/未测」表里的 7 个场景写成 pytest fixtures（不必跑 LLM，用 mock client）
- [ ] 至少覆盖：基础 chat / rewind / compact 触发 / sub-agent 派发 / sandbox 启动 / pending A 注入 / pending B chain
- [ ] CI 跑得起来（GitHub Actions / 本地 makefile）

**工作量**：1.5 天  
**前置**：无  
**收益**：之后每个 phase 上线前能跑一遍兜底

---

### Phase 1 · 内核抽取（零功能改动）

> 最高杠杆的一步。完成后两个巨型文件各砍 ~500 行。

- [ ] 建 `kernel/` 目录
- [ ] `kernel/tool_protocol.py`：定义 `Tool` ABC（含 `dispatch`） + `FormTool` ABC（仅 `to_event` / `persist_key`）
- [ ] `kernel/tool_registry.py`：`_REGISTRY: dict[str, Tool|FormTool]` + `register()` 装饰器
- [ ] 把 `tools/registry.py::TOOL_SCHEMAS` 拆到各 `tools/*.py`，每个工具用 `@register` 自注册
- [ ] `kernel/tool_registry.py::dispatch(name, args)` 替换 `tools/registry.py::dispatch` 的 if 链
- [ ] `kernel/llm_loop.py::StreamingLLMLoop`：抽 `run_turn` 和 `_run_sub_turn` 共有部分（retry / stream parse / cancel check / usage 累积）
- [ ] `services/chat/turn.py::run_turn` 改用 `StreamingLLMLoop`，瘦身到 ~200 行
- [ ] `bg_runner._run_sub_turn` 改用 `StreamingLLMLoop`，瘦身到 ~250 行

**工作量**：2 天  
**前置**：Phase 0  
**收益**：
- `tools/registry.py` 877 行 → 消失
- `turn.py` 850 行 → ~200 行
- `bg_runner.py` 967 行 → ~250 行
- 立刻不再有四个巨型文件

---

### Phase 2 · REST 类型契约（前端）

- [ ] `npm i -D openapi-typescript`
- [ ] 加 `npm run gen:api` script（读 `localhost:8000/openapi.json`）
- [ ] 检查所有 router 是否声明 `response_model` 和 Pydantic 请求模型，补全
- [ ] 生成 `src/api/schema.gen.ts`，删除 `client.ts` 和 `types.ts` 里手写的 REST 类型
- [ ] CI 加 drift 检测：`npm run gen:api && git diff --exit-code src/api/schema.gen.ts`

**工作量**：0.5 天  
**前置**：无  
**收益**：~80 行手写类型消除，REST 漂移 CI 拦截

---

### Phase 3 · SSE 类型契约

- [ ] 建 `kernel/sse_protocol.py`：定义 15+ 个 `Chunk` Pydantic 模型（`ContentChunk` / `ToolChunk` / `MetaChunk` / `QueueChunk` / `MidUserChunk` / `CompactStartChunk` / ...）
- [ ] 用 `Annotated[Union[...], Field(discriminator="type")]` 做联合
- [ ] camelCase 字段用 `Field(alias=...)` + `model_dump(by_alias=True)`
- [ ] 全量替换 `emit(...)` / `runner.publish(...)` 调用：从传 dict 改成传 Pydantic 实例
- [ ] 加一个仅用于 OpenAPI codegen 的 endpoint `GET /turns/{tid}/stream_schema_only`，`response_model=Chunk`
- [ ] 重跑 `npm run gen:api`，删 `src/types.ts` 里的 `Chunk` 联合
- [ ] 前端 `useChat.ts` 的 SSE generator 加 dev-mode Zod 校验（drift 时 console 报错）

**工作量**：1.5 天  
**前置**：Phase 2  
**收益**：
- SSE 漂移彻底消除
- 后端 runtime 校验每条 chunk
- 加 chunk type 时前端 switch 漏 case 由 TS exhaustiveness 报错

---

### Phase 4 · Capability 层

> Bonus 3（impersonation）的基础设施，且对现有所有 feature 都有用。

- [ ] 建 `capabilities/auth_context.py`：`AuthContext`（user / acting_as / capabilities / audit_session_id）
- [ ] 替换 `routers/auth.py::current_user` dep 为 `current_auth` 返回 `AuthContext`
- [ ] 所有 router 依赖 `AuthContext` 而不是 `User`，`ctx.effective_user` 取实际用户
- [ ] `llm.py::use_user_api_key` 改用 `ctx.effective_user.api_key`
- [ ] `capabilities/audit.py`：DB write hook，impersonation 期自动写 audit log
- [ ] `capabilities/permission.py`：admin / owner / shared 判定 helper

**工作量**：1 天  
**前置**：Phase 0  
**收益**：
- 所有 `if user.id != session.user_id and not user.is_admin` 检查统一收口
- Impersonation 自动有 audit 痕迹
- 多租户 / 共享会话需求出现时只改 capability 层

---

### Phase 5 · Pipeline / Hooks 抽象

- [ ] `kernel/pipeline.py`：定义 `Stage = Literal["pre_turn", "pre_iter", "post_iter", "post_turn"]` + `Hook` 类型 + `register_hook` 装饰器
- [ ] `kernel/llm_loop.py::StreamingLLMLoop.run` 改成按 Stage 顺序执行 hook
- [ ] **lite gate 拆 hook**：
  - `features/triage_guard/hooks.py` → PreTurn
  - `features/memory_ranker/hooks.py` → PreTurn
  - `features/skill_gate/hooks.py` → PreTurn
  - `features/spawn_advisor/hooks.py` → PreTurn
- [ ] **compact 拆 hook**：
  - `features/compact/hooks.py::soft_compact_hook` → PreTurn（条件：context_tokens > SOFT）
  - `features/compact/hooks.py::hard_compact_hook` → PostIter（条件：context_tokens > HARD）
- [ ] **pre-inject 拆 hook**：`features/pre_inject/hooks.py` → PreTurn
- [ ] `services/chat/turn.py::run_turn` 退化成 ~50 行编排：构造 ctx + 跑 hooks + StreamingLLMLoop

**工作量**：2 天  
**前置**：Phase 1  
**收益**：
- 主循环不再有"为这个 feature 加分支"的诱惑
- bonus 1 实现成本从"改 turn.py"降到"加一个 hook"

---

### Phase 6 · 重组到 features/

> 大规模文件移动，借 Phase 1-5 已经把内核挖空，剩下的搬运可以一次完成。

- [ ] 按第三部分对照表移动文件
- [ ] 每个 `features/X/` 内统一结构：`router.py` / `service.py` / `tool.py` / `hooks.py` / `schemas.py` / `__init__.py`
- [ ] `features/X/__init__.py` 负责 `register()`：把 hook 装到 pipeline、把 tool 装到 registry
- [ ] `main.py` 的 lifespan 里显式 import 所有 feature 触发自注册
- [ ] `routers/` 目录消失，每个 feature 自己拥有 router

**工作量**：2-3 天（机械活但量大）  
**前置**：Phase 1 + Phase 5  
**收益**：
- 加 feature = 加一个目录，0 行碰旧文件
- 任何"X feature 在哪"都是一个目录路径回答

---

### Phase 7 · 工具协议升级（FormTool）

> Bonus 1（next_options）的基础设施。

- [ ] `kernel/tool_protocol.py::FormTool` 抽象（已在 Phase 1 占位，此 phase 真正接入）
- [ ] `kernel/llm_loop.py` 的 iter 分支：`isinstance(spec, FormTool)` 时不回灌、emit SSE 事件、写 message.meta
- [ ] `features/next_options/form_tool.py`：`SuggestOptionsTool(FormTool)` 实现
- [ ] SYSTEM_PROMPT 加一段：在合适时机调 `suggest_options`
- [ ] SSE chunk 新增 `NextOptionsChunk`（已在 sse_protocol.py 里）
- [ ] 前端 `useChat.ts` 接 `case 'next'`（已存在）→ 实际生效

**工作量**：1 天  
**前置**：Phase 1 + Phase 3 + Phase 6  
**收益**：bonus 1 上线；同时铺平未来所有"结构化侧带输出"feature

---

### Phase 8 · Event Bus + Jobs 框架

> Bonus 2（auto_research）的基础设施。

- [ ] `kernel/event_bus.py`：in-process pub/sub，`publish(event_name, payload)` / `subscribe(event_name, handler)`
- [ ] `jobs/runtime.py`：`Job` ABC + 调度 + 持久化（DB 表 `jobs`）
- [ ] `features/workspace/hooks.py`：上传完成后 `event_bus.publish("FolderUploaded", {...})`
- [ ] `features/auto_research/events.py`：订阅 FolderUploaded → 启动 `FolderIndexerJob`
- [ ] `features/auto_research/jobs.py::FolderIndexerJob`：扫文件 / lite 分类 / 写 memory / 决定要不要主动起 turn
- [ ] `features/auto_research/service.py::spawn_kickoff_turn`：合成 user prompt 起新 turn（turn.meta.kind = "auto_research_kickoff"）

**工作量**：2 天  
**前置**：Phase 6  
**收益**：bonus 2 上线；同时铺平未来所有"异步反应链"feature

---

### Phase 9 · Bonus 3 落地（impersonation）

- [ ] `features/impersonation/router.py`：`POST /admin/impersonate/{user_id}` → 返回临时 token，签发新 `AuthContext`
- [ ] AuthContext 在 impersonation token 下 `acting_as` 字段被设置
- [ ] 所有 DB write 自动经 `capabilities/audit.py` 留痕
- [ ] 前端 admin 面板加 "临时访问该用户" 按钮 + 顶部 banner 提示当前在借身份状态
- [ ] 被借的用户在 session 列表看到"管理员曾在 X 时间访问过此会话" 提示

**工作量**：1 天  
**前置**：Phase 4  
**收益**：bonus 3 上线

---

### Phase 10 · 前端重组

> 后端稳定后再动。前后端不要同时大改。

- [ ] 拆 `store.ts` 到 `stores/{chat,bg,workspace,memory,ui}.ts`
- [ ] 拆 `Composer.tsx` 705 行：mention / slash / cite chip 各自独立子组件
- [ ] 拆 `Sidebar.tsx` 807 行：SessionList / WorkspaceSwitch / BgBadge 独立
- [ ] 拆 `Workspace.tsx` 790 行：FileTree / FilePreview / Uploader 独立
- [ ] 按 `features/` 镜像后端组织
- [ ] `useChat.ts` 的 consumeStream switch 改成 dispatch table（每个 case 一个 handler 函数）
- [ ] 评估上 react-router（仅在需要深链时）

**工作量**：3 天  
**前置**：后端 Phase 6 完成  
**收益**：前端文件全部 <400 行；feature 与后端对应

---

## 第五部分 · 总时间线

```
Phase 0  测试基线              ┃ 1.5 天 ┃ 持续维护
Phase 1  内核抽取               ┃ 2 天   ┃ 立刻砍三个巨型文件
Phase 2  REST 类型契约          ┃ 0.5 天 ┃ 可并行
Phase 3  SSE 类型契约           ┃ 1.5 天 ┃ 依赖 Phase 2
Phase 4  Capability 层          ┃ 1 天   ┃ 可并行
Phase 5  Pipeline / Hooks       ┃ 2 天   ┃ 依赖 Phase 1
Phase 6  重组到 features/       ┃ 2.5 天 ┃ 依赖 Phase 1, 5
Phase 7  FormTool 协议          ┃ 1 天   ┃ 依赖 Phase 1, 3, 6
Phase 8  Event Bus + Jobs       ┃ 2 天   ┃ 依赖 Phase 6
Phase 9  Bonus 3 (impersonation)┃ 1 天   ┃ 依赖 Phase 4
Phase 10 前端重组               ┃ 3 天   ┃ 依赖后端稳定

合计：~18 个工作日（约 4 周专注 / 8 周兼做）
```

**关键路径**：Phase 0 → 1 → 5 → 6 → 7 / 8（约 9 天），这条路径完成后所有 bonus feature 都能"加目录就上线"。

**强烈建议的最小可上线版本**：Phase 0 + 1 + 2 + 3 = 5.5 天。这一档完成就已经：

- 不再有 800+ 行文件
- 类型契约 CI 兜底
- 后端 / 前端类型一致性彻底解决
- 主循环抽出，后续 feature 加 hook 而不是改主流程

---

## 第六部分 · 不要走的弯路

| 反模式 | 为什么不做 |
|---|---|
| Hexagonal / Ports-and-Adapters / Clean Architecture 全家桶 | 这是个 LLM agent，不是企业 ERP。Use Case + Application Service + Repository Interface 三层抽象是纯负担 |
| Microservices | in-process pub/sub + 单 binary 部署。等真有水平扩容需求换 Redis Stream 是局部改造 |
| Repository pattern over ORM | 已经有 `repo/`，再包一层 `IRepo` interface 是纯负担 |
| 自动 plugin discovery / entry points | 所有 hook 在 `main.py::lifespan` 显式 import + 注册，等真的有 5+ 三方插件再做 dynamic |
| 100% feature 解耦 | `features/auto_research` 调 `features/chat.spawn_turn` 是合理依赖。要禁的是 feature 互相 import 内部状态 |
| 一次性大改 | 每个 Phase 必须独立可上线、独立可回滚 |
| 前后端同时大改 | 后端稳定后再动前端。Phase 6 之前不要动前端 |
| 上 GraphQL / tRPC 解决类型契约 | 换协议成本远大于收益。OpenAPI codegen 是同等效果的 1/10 工作量 |
| `pydantic2ts` / `pydantic-to-typescript` 直转 TS | Pydantic v2 维护跟不上，输出不稳。走 OpenAPI codegen |
| 把 LLM agent 抽成 LangChain / LangGraph | 你已经写出了比这两个更贴合需求的实现，回头套框架是大倒退 |

---

## 第七部分 · 与 ROADMAP 的关系

ROADMAP 当前正在做「通知协议 v2」+「Phase 3 进程后台任务 V1」。

**和本文协调建议**：

1. **通知协议 v2 完成（已基本落定）后再启动 Phase 1**——避免 sub-agent / digest turn / pending 路径同时大改
2. **Phase 3 V1（sandbox + run_sandbox）** 可在本文 Phase 1 完成后开始，自然落到 `features/sandbox_runner/`
3. **Phase 3 V1.5（experimenter preset）** 可在本文 Phase 7（FormTool）完成后开始，experimenter 用的多 sandbox 并行需要 deferred wait 抽到 kernel
4. **湿实验 skill 支持（Phase 7）** 应在本文 Phase 6（features/）完成后开始

---

## 第八部分 · 决策记录

| # | 决定 | 理由 |
|---|---|---|
| R1 | 用 5 元结构（kernel/domain/infra/features/capabilities），不用 DDD 全家桶 | 项目规模够大但还不到企业级；轻量分层够用 |
| R2 | in-process event bus，不上 Redis | 单 binary 部署目标不变；扩容时局部改 |
| R3 | OpenAPI codegen 解 REST 类型契约 | FastAPI 原生支持，工具成熟 |
| R4 | Pydantic 联合 + 假 endpoint 解 SSE 类型契约 | 不另起一套 schema 系统 |
| R5 | FormTool 而不是 fence parsing 做 next_options | 协议可靠性高 + 已有 `type:'next'` 通道；fence parsing 留给将来非结构化侧带 |
| R6 | AuthContext 通过 ContextVar 透传，不污染签名 | 沿用现有 `_active_api_key` 模式 |
| R7 | features 之间通过顶层 service 函数 + event 通信 | 避免 lazy import 解循环这种 hack |
| R8 | 前端拆完 store 之后再考虑 router | 当前 store-driven 还撑得住，router 是真有深链需求再上 |
| R9 | bg_runner 和 chat turn 共享 `StreamingLLMLoop` 而不是两套维护 | 两边已稳定，共享代价低于不共享 |
| R10 | 每个 Phase 独立上线，不允许"大爆炸"重构 | 单 binary 项目最大风险是回归 |

---

*最后更新：2026-05-20*
