# Oryx-RE 重构 v3 — 2026-05-21

> v2([REFACTOR-260520-v2.md](REFACTOR-260520-v2.md))关注三短板 + Redis + 多 pod;A/C 已完成,B 基础设施完成但 caller 未接管,Redis / Pods 等部署需要。
> v3 把 v2 未列入的「类型契约对齐」编号为短板 D,把 v2 末尾 blockquote 里的「小尾巴」正式化为 E1–E3,新增跨切纪律轨道 Q1–Q6,并补一个多-pod 代码侧细项 G1。

## 编号约定

- **A–E**:功能性重构。可独立 ship。
- **F–G**:基础设施。依赖外部部署 / 跨进程基建。
- **Q**:跨切纪律。一次性 + 渐进采用,不绑 feature 工作。
- **R**:决策记录,延续 v2 的 R1–R8。

## 现状结转

| 编号 | 项 | 状态 | 备注 |
|---|---|---|---|
| A | StateStore 抽象 | ✅ 完成 | v2 落 |
| B | domain entity + ORM mapping | ✅ 基础设施完成 / 接管推迟 | 触发未到 |
| C | features/ 物理迁移 | ✅ 完成 | v2 落 |
| D | 类型契约完全对齐 | ⏸ 未做 | v3 新增 |
| E1 | `tools/bg.py` 拆 | ⏸ 未做 | v2 blockquote 升格 |
| E2 | `features/chat/` 内部拆 compact / pre_inject | ⏸ 未做 | 同上 |
| E3 | domain entity caller 接管 | ⏸ 未做 | 同上;B 的延续 |
| F | Redis 状态面 | ⏸ 未做 | 依赖部署需要 |
| G | 多 pod 拓展 | ⏸ 未做 | 依赖 F |
| G1 | 跨 pod cancel + bg_task 接管 | ⏸ 未做 | 新增;v2 R5 之外的代码部分 |

---

## 第一部分 · 短板 D · 类型契约完全对齐

### 现状

v1 README 「类型契约」章节定义的标准是:

> REST 请求 / 响应 → router Pydantic 模型 + `response_model` → openapi.json → `src/api/schema.gen.ts`

实际盘点(2026-05-21):

| 维度 | 数 |
|---|---|
| `features/*/router.py` 总 endpoint | 54 |
| 有 `response_model` 的 | 10 |
| **缺口** | **44** |

缺口 endpoint 簇:

| feature | 涉及 endpoint | 影响的前端类型 |
|---|---|---|
| admin | workspaces / sessions / sessions/:sid/messages / users/me / feedback / fork / reset / archive / patch | `ServerMessage` / `Session` / `Workspace` / 用户偏好 |
| memory_store | memories CRUD(5) | `Memory` |
| skills | skills 列表 + 详情 | `Skill` |
| sub_agent | bg-tasks 列表 / 详情 / events / cancel / wake | `BgTask` / `BgEvent` |
| workspace | tree / file / raw / download / mkdir / upload / delete | `FsNode` 系列 |
| impersonation | impersonate / end-impersonation / audit | `AuditEntry` |
| auth | me / invites | `UserMe` / `Invite` |

前端侧手写 mirror:
- `src/api/client.ts`:`ServerMessage`(7 字段)直接手写
- `src/types.ts`:`Message` / `Workspace` / `Session` / `Memory` / `BgTask` / `BgEvent` / `Skill` / `FsNode` / `QueuedInput` 全部手写;部分字段 wire-only(应该派生),部分 FE-only(应该保留)

### 做法

1. **建模**:为每个缺口 endpoint 写视图 Pydantic 模型 — 多数现有 `_xxx_view()` dict 构造器已经做了半套,直接转 BaseModel。命名约定:`<Entity>View`(eg. `MessageView`,`SessionView`,`FsNodeView`)。
2. **挂 `response_model`**:逐 endpoint 加,集中在一个 PR 内。
3. **生成**:`npm run gen:api`(backend 跑着)或 `gen:api:offline`(dump openapi)。
4. **前端拆 wire / FE**:
   - `src/types.ts` 顶部 `import type { components } from './api/schema.gen'`
   - 每个 entity 拆成 `WireXxx = components['schemas']['XxxView']` + `interface Xxx extends WireXxx { ... FE-only }`
   - FE-only 字段(`Message.thinking / segments / inserts / pickedOptionDescription`,`BgTask.lastNote`)显式保留
5. **删除手写**:`src/api/client.ts::ServerMessage` 删除,改 `type ServerMessage = components['schemas']['MessageView']`。
6. **CI 闸门**(并入 Q1):`gen:api:offline && git diff --exit-code src/api/schema.gen.ts`。

### 工作量

| 项 | 工作量 |
|---|---|
| 视图 Pydantic 模型 + response_model 装饰 | 1.5d |
| 前端 wire / FE 拆解 + import 改造 | 1.0d |
| CI hook(并 Q1)+ 烟测 | 0.5d |
| **合计** | **3.0d** |

无功能变更。

### 触发条件

无外部依赖,但下面任意一个出现立刻做:

- 任何一次 schema.gen.ts drift 导致线上 bug
- 任何一次前后端字段不一致排查 >1h
- 任何新 feature 提交时未自带 response_model

---

## 第二部分 · 小尾巴正式化(E1–E3)

v2 doc 第一节末把这三项放在 blockquote 里,工作量 / 触发条件没写。现在编号化。

### E1 · `tools/bg.py` 拆

**现状**:`tools/bg.py` 含 `spawn_sub_agent` / `run_sandbox` / `get_bg_task` / `inspect_sub_agent` / `wait_for_seconds` / `wait_for_signal` — 跨 `sub_agent` 与 `sandbox_runner` 两个 feature。

**做法**:
- `features/sub_agent/tool.py` ← spawn_sub_agent / get_bg_task / inspect_sub_agent / wait_for_signal(LLM 维度)
- `features/sandbox_runner/tool.py` ← run_sandbox / wait_for_seconds(共享)
- `tools/bg.py` 保留作 thin re-export 一段时间,无引用后删

**工作量**:0.5d。机械活。

**触发**:第 3 个跨 feature 工具被加进 bg.py 时立刻做。或并行有 capacity 时随手做。

### E2 · `features/chat/` 内部拆 compact / pre_inject

**现状**:`features/chat/` 含 `turn.py`(740 行)/ `compact.py` / `bootstrap.py` / `messages.py` / `router.py`。compact 与 pre_inject(attachment 注入,搜 `persist_attachment_cycle`)是独立子流程,跟主 turn 编排耦合度低。

**做法**:
- `features/compact/`:把 `chat/compact.py` 整体迁出 + `chat/turn.py` 里 `compact_if_over_threshold` 调用点改成 import
- `features/pre_inject/`:把 attachments + skill_load + memory_load 三个 cycle 函数(`chat/messages.py` 里)迁出
- 主 `chat/turn.py` 改成只编排 loop,不知具体子流程实现

**工作量**:1.0d。

**触发**:`chat/turn.py` 行数 > 1200,或主编排 + compact 同步改动出现 review 范围混乱。

### E3 · domain entity caller 接管

**现状**:v2 短板 B 留下了 `domain/` 6 entity + `db/mapping.py`,但**没有 caller 用**。所有 router / service 仍直接拿 ORM 操作 + repo。

**做法**:逐个 entity 接管。优先级:
1. `Session.archive() / unarchive() / assert_writable()` — `admin/router.py` 已有 archive 入口,先收
2. `Turn.cancel() / complete() / fail()` — `chat/turn.py` 的状态转换收口
3. `Memory.promote_to_workspace() / assert_scope_consistent()` — `sub_agent/service.py` 与 `memory_store/router.py` 各自实现 promote,合并入口

**单次接管模板**:
```python
sess = await repo.get_session(sid)           # 老:返回 ORM
↓
sess_d = to_session(sess)                    # ORM → domain
sess_d.archive()                             # 不变式入口
apply_session(sess, sess_d)                  # domain → ORM
await repo.save(sess)
```

**工作量**:2.0d(3 个 entity × ~0.6d 平均)。

**触发**:第 3 处「应该长在 entity 上的规则」被复制粘贴时立刻做(v2 R6 已经写过这条触发,延续)。

---

## 第三部分 · 跨切纪律(Q1–Q6)

不绑任何 feature,目标是「让后续重构不回头」。**任何时刻都该做**,排在批次 1–2 内尽快闭环。

### Q1 · 类型 codegen drift CI 闸

**做法**:
- GitHub Actions(或本地 git hook):`npm run gen:api:offline && git diff --exit-code src/api/schema.gen.ts`
- backend 在 PR pipeline 跑起来,FE 跑 gen:api:offline 是离线模式不需要 server
- 不一致 → PR 失败

**工作量**:0.2d。

### Q2 · 编译 / lint / 迁移 CI 闸

**做法**:
- `npx tsc -b` 必须无 error
- `ruff check backend/` 必须通过(项目已有 ruff config?需确认)
- `alembic heads` 必须只有一个 head(防 merge 出双 head 迁移)
- 进 PR pipeline

**工作量**:0.3d。

### Q3 · 统一错误响应格式

**现状**:HTTPException 散用,前端 `r.ok` 检查 + `await r.text()` 各处不同。

**做法**:
- 后端定义 `class ErrorEnvelope(BaseModel)`:`{ code: str, message: str, detail?: dict }`(参 RFC 7807 但简化)
- FastAPI exception handler 统一包装
- 前端 `api/client.ts::authFetch` 解析 envelope → 抛 `ApiError(code, message, detail)`
- 现有 endpoint 渐进采用(新 feature 强制,老 endpoint 触发时改)

**工作量**:1.0d。

### Q4 · KV cache 字节稳定性纪律

**现状**:SYSTEM_PROMPT 与 history serialization 的字节稳定性是 KV cache 命中的前提,但没工程化检查。改 `STRIP_OLD_REASONING` / `COMPACT_PROMPT` / `SYSTEM_PROMPT` 任意一字节就破 cache。

**做法**:
- CLAUDE.md(backend 目录新增 `backend/CLAUDE.md`)加一节「KV cache prefix discipline」,列出哪些 string 是 cache prefix
- 提供脚本 `scripts/prefix_hash_check.py`:计算当前 SYSTEM_PROMPT / boot head messages 的 hash,与上次 commit 对比,变化超阈值时警告(不阻塞)
- 关键 prompt 改动写 commit message:`prompt-cache-break: <reason>`

**工作量**:0.5d。

### Q5 · 新 feature 接入 observability 检查清单

**做法**:`backend/CLAUDE.md` 加一节「feature checklist」:

```markdown
新 feature 提交前:
- [ ] router 每个 endpoint 都有 response_model(D 闸已自动检测)
- [ ] 关键操作 log.info / log.warning,key 字段含 trace_id / session_id
- [ ] 有 bg activity 的接 bg_event(drawer 可见)
- [ ] settings 字段写注释 + 默认值合理
- [ ] alembic 迁移单调递增,upgrade/downgrade 都写
```

**工作量**:0.3d(文档,无代码)。

### Q6 · API 版本化策略

**现状**:`/api/...` 无 version 前缀。一旦 wire 字段破坏性变更,前端老版本会断。

**做法**:
- 加 `/api/v1/` 前缀(`APIRouter(prefix="/api/v1")`),老 `/api/...` 保留 thin re-export 半年
- 写策略文档:wire 字段加只能 minor,删 / 改语义只能 major
- v3 文档把这个写明,作为往后所有 schema 变更的判断基线

**工作量**:0.3d 改 prefix + 0.3d 写策略文档 = 0.6d。

---

## 第四部分 · 基础设施延续(F / G / G1)

v2 第二、第三部分定了方向;这里补成可开发 SPEC。

### F · Redis 状态面 SPEC

**目标**:把跨请求 / 跨 SSE / 跨 pod 必须共享的 live state 从进程内 dict
迁到 Redis。DB 仍是真实业务状态;Redis 只保存短 TTL、可重建的协调状态。

**新增实现**:

```
backend/app/infra/state/redis.py
  RedisStateStore
  RedisPubSubBus
  RedisSignalHub
  RedisWakeHub
  RedisLockManager
  RedisKVStore
  RedisStreamLog
```

`main.lifespan` 根据 `STATE_BACKEND=memory|redis` 选择:

```python
if settings.STATE_BACKEND == "redis":
    configure(RedisStateStore(settings.REDIS_URL, prefix=settings.REDIS_PREFIX))
else:
    configure(InProcessStateStore())
```

**Redis key 约定**:

| 协议 | Key / channel | TTL | 语义 |
|---|---|---:|---|
| pubsub | `oryx:{env}:pubsub:sid:{sid}` | 无 | session SSE fan-out |
| signal slot | `oryx:{env}:signal:{sid:name}:slot` | 24h | `{count, terminal}` ack-only 状态 |
| signal wait | `oryx:{env}:signal:{sid:name}:wait` | ≤timeout+60s | waiter 唤醒 channel/list |
| wake | `oryx:{env}:wake:{sid}` | 无 | wait_for_seconds 早醒 |
| lock | `oryx:{env}:lock:{key}` | 30s renew | digest spawn / pod claim 互斥 |
| kv | `oryx:{env}:kv:{key}` | caller 指定 | impersonation token / audit scratch |
| stream | `oryx:{env}:stream:{name}` | 24h | 可 replay 的短事件流(后续跨 pod runner 用) |

**语义要求**:

- `pubsub.publish(sid, chunk)` 必须同时 wake `wait_for_seconds(sid)`。
- `emit_signal(terminal=True)` 必须释放同名全部 waiter,并 sticky terminal。
- 非 terminal signal 无 waiter 时 `count += 1`;下一个 waiter 立即消费。
- Redis lock 必须有 owner token;release 只能释放自己的 token。
- 所有 Redis payload 都是 JSON;禁止 pickle。
- Redis down 时:
  - 单 pod dev 可 fallback memory 并打 `state.redis_fallback` WARN
  - 多 pod prod 启动失败,不要半降级

**Settings**:

```
STATE_BACKEND=memory|redis
REDIS_URL=redis://...
REDIS_MODE=single|cluster|auto
REDIS_PREFIX=oryx:prod
REDIS_REQUIRED=false|true
REDIS_LOCK_TTL_S=30
STATE_SIGNAL_TTL_S=86400
STATE_STREAM_TTL_S=86400
```

实现约束:

- `REDIS_MODE=cluster` 支持单 URL 或逗号分隔多个 startup node;`auto`
  在多个 URL 时按 cluster 初始化。
- cluster 下 signal key 使用 hash tag 固定单槽,确保 ack-only 消费脚本只碰
  一个 key。

**验收**:

1. 两个 backend 进程连同一个 Redis;A 进程 session SSE 能收到 B 进程 publish。
2. A 进程 `wait_for_signal`,B 进程 `emit_signal` 能唤醒且 ack-only。
3. terminal signal 后新 waiter 立即返回 `{fired:true, terminal:true}`。
4. Redis 断开时 prod (`REDIS_REQUIRED=true`) 启动失败。
5. `maybe_spawn_digest_turn` 并发 10 次只创建 1 个 digest turn。

### G · 指向 pod 的流量 SPEC

**目标**:workspace / turn / bg_task 有 owner pod 时,请求必须落到正确 pod。
不依赖浏览器知道 pod;入口层或 middleware 自动转发/重定向。

**数据模型**:

```
workspaces.owning_pod_id nullable string
turns.owning_pod_id nullable string
bg_tasks.owning_pod_id nullable string
```

`owning_pod_id` 格式为稳定 pod id,例如 StatefulSet hostname `oryx-0`。

**已落地基础切片**:

- Alembic `0011_pod_ownership` 新增三张表的 nullable owner 字段和
  owner/status 查询索引。
- 新建 workspace / turn / bg_task 写入当前 `POD_ID`。
- `POD_MULTI_ENABLED=false` 时启动恢复保持单机旧行为;`true` 时只恢复
  `owning_pod_id == POD_ID` 的 active turn/bg_task。
- `POD_MULTI_ENABLED=true` 时 middleware 会在 body 解析前按 path owner
  proxy `workspace / turns / bg-tasks` 请求;`/chat`、rewind、compact 在
  JSON body 解析后按 session workspace owner proxy。
- 内部 proxy 携带 `x-oryx-forwarded-by`、`x-oryx-target-pod`、
  `x-oryx-route-ts`;配置 `POD_ROUTE_HMAC_SECRET` 后会校验
  `x-oryx-route-signature`。

**Pod identity settings**:

```
POD_ID=oryx-0
POD_MULTI_ENABLED=false|true
POD_BASE_URL=http://oryx-0.oryx-headless:8000
POD_BASE_URL_TEMPLATE=http://{pod_id}.oryx-backend-headless:8000
POD_ROUTE_MODE=redirect|proxy
POD_ROUTE_HEADER=x-oryx-target-pod
POD_ROUTE_HMAC_SECRET=...
POD_HEARTBEAT_TTL_S=90
```

**路由规则**:

| 请求类型 | owner 解析 | 行为 |
|---|---|---|
| `/api/v1/workspace/{sid}/...` | session → workspace.owning_pod_id | 非 owner → 指向 owner |
| `/api/v1/turns/{tid}/stream|cancel|pending` | turn.owning_pod_id | 非 owner → 指向 owner |
| `/api/v1/bg-tasks/{id}/cancel|wake|events` | bg_task.owning_pod_id | 非 owner → 指向 owner |
| `/api/v1/sessions/{sid}/events` | session active turn/bg owner 优先,否则本 pod 可接 | SSE 可任意接,但 owner 事件来自 Redis pubsub |
| 纯 CRUD(`/sessions`, `/memories`) | DB only | 任意 pod |

**redirect mode**:

- middleware 返回 `307 Temporary Redirect`
- `Location` 指向内部 LB 可路由地址或同域 sticky path
- 仅用于服务端/内部调用;浏览器跨 pod 暴露复杂时用 proxy mode

**proxy mode**:

- 当前 pod 用 `httpx` 转发请求到 owner pod 的 headless DNS
- 附带 `x-oryx-forwarded-by`, `x-oryx-target-pod`, `x-oryx-route-ts`,
  `x-oryx-route-signature`
- owner pod 校验签名,防止外部伪造 pod-target header
- SSE 代理必须 stream pass-through,禁 buffering

**claim 规则**:

- 创建 workspace 时:写 `workspace.owning_pod_id = POD_ID`
- 创建 turn/bg_task 时:写 `owning_pod_id = POD_ID`
- `POD_MULTI_ENABLED=true` 时 boot resume 只恢复 `owning_pod_id == POD_ID`
  的 row;单机/本地默认 false,继续恢复全部 active row
- owner pod 心跳写 Redis:`pod:{POD_ID}:heartbeat` TTL=`POD_HEARTBEAT_TTL_S`
- owner 不在线时:
  - workspace 请求可由新 pod claim workspace(文件系统/PVC 运维保证可见)
  - running turn/bg_task 不活体接管,按 G1 标 failed/cancelled

**验收**:

1. 两 pod 启动;workspace 归属 `oryx-0`;请求打到 `oryx-1` 会被路由到 `oryx-0`。
2. turn stream 在 owner pod 持续 streaming;非 owner pod 的 stream 请求不中断转发。
3. cancel 请求打到非 owner pod,owner runner 收到 cancel_event。
4. 伪造 `x-oryx-target-pod` 无签名 → 403。
5. owner heartbeat 过期后,workspace 可重新 claim;running turn/bg_task 标 terminal。

### G1 · 跨 pod cancel + bg_task 接管

**现状**:本轮已把 main turn / sub_agent / sandbox 的 live runner registry
拆成独立 runner 模块;这为 G1 替换成 Redis/owner-pod route 留出了边界。

**做法**:

- `TurnRunner` / `SubAgentRunner` / `ProcessRunner` 继续保留本 pod live state。
- DB row 持久化 `owning_pod_id`。
- cancel / wake / pending 先经 G 的 pod route 到 owner pod;owner pod 调本地 runner。
- owner pod dead:
  - `turn.status='failed', error='pod dead: <pod_id>'`
  - `bg_task.status='failed', error='pod dead: <pod_id>'`
  - pending input 标 `cancelled`
  - session SSE 通过 Redis pubsub 发 terminal 状态
- 接管:**不做活体接管**。后续如要做,必须先把 LLM stream cursor /
  tool dispatch / sandbox container handle 外部化,另开 SPEC。

**工作量**:F 2d + G 2.5d + G1 0.8d。G1 比原 0.5d 略增,因为需要补
DB 字段 / route middleware / terminal event 三件套。

---

## 第五部分 · 批次与时间线

### 批次划分

| 批次 | 包含 | 工作量 | 依赖 |
|---|---|---|---|
| **1 · 类型契约 + CI 护栏** | D + Q1 + Q2 | 3.5d | 无 |
| **2 · 跨切纪律** | Q3 + Q4 + Q5 + Q6 | 2.5d | 1 完成后 |
| **3 · 小尾巴(触发驱动)** | E1 / E2 / E3 | 0.5–3.5d | 触发条件 |
| **4 · 基础设施(部署需要)** | F → G → G1 | 5.3d | 部署需求 |

### 关键路径

```
现在 ──→ 批次 1  3.5d   类型契约 + CI 护栏闭环
     ──→ 批次 2  2.5d   跨切纪律
─────────────── 6.0d   独立可 ship,不依赖外部基建
     ──→ [触发] 批次 3  0.5–3.5d
     ──→ [部署] 批次 4  5.0d
```

### 与 v2 时间线对比

v2 原列:A 1.5d ✅ + B 3.0d(基础设施完成,接管 2.0d) + C 2.5d ✅ + Redis 2.0d + Pods 2.5d = **11.5d**

剩余:**B 接管 2d + F 2d + G 2.5d + G1 0.8d** = 7.3d

v3 新增:**D 3d + Q1-Q6 2.5d + E1 0.5d + E2 1d** = 7d

**全做完合计**:7.3 + 7 = **14.3 工作日**。

---

## 第六部分 · 决策记录(R9–)

延续 v2 R1–R8。

| # | 决定 | 理由 |
|---|---|---|
| R9 | 类型契约对齐(D)作为短板编号,而非「持续维护项」 | 44 endpoint 缺口已经形成债;需要明确闸门和工作量,纳入计划而非日常 PR |
| R10 | wire / FE-only 字段在前端通过 `interface X extends WireX { ... }` 拆解,不试图把 FE-only 字段反向回 backend | FE 渲染态(thinking 流式中间态、segments 切分)本质就在 FE 派生,后端不应该知道 |
| R11 | API 版本化(Q6)做但**不破坏老路径** | 前端老版本仍能跑;为将来真正的 v2 wire 留口子 |
| R12 | KV cache 字节稳定性(Q4)只做警告不做阻塞 | 真正的 prefix 变更是有意的(改 prompt 是常事),工具化检查是提醒,不是阻拦 |
| R13 | 跨 pod failover 不做活体接管(G1) | 与 v2 R5 一致的精神延伸;运维选择 + 标 failed 让用户重试 |
| R14 | D 与 Q1 必须同 PR / 邻 PR 落 | D 修完没 CI 闸守,下次手抖又 drift,白修 |
| R15 | Redis 是协调状态,DB 仍是真实业务状态 | Redis TTL/丢失必须可恢复;业务事实不能只存在 Redis |
| R16 | pod 指向流量默认走 proxy mode,redirect 仅保留给内部调用 | 浏览器/SSE/同域 cookie 下 307 跨 pod 复杂;proxy 更可控 |

---

*最后更新:2026-05-23*
