# Oryx-RE 重构 v2 — 2026-05-20

> v1 ([REFACTOR-260520.md](REFACTOR-260520.md)) 完成了内核抽取 / 类型契约 / 特性化重组(P1–P10)。
> v2 关注 v1 显式留下的三个短板 + 多 pod 部署。

## 当前进度

| 项 | 状态 | 备注 |
|---|---|---|
| 短板 A · StateStore 抽象 | ✅ 已落地 | `infra/state/` + `InProcessStateStore`,bg_hub / impersonation / digest lock 全部走 StateStore;切 Redis = 一行 `configure(...)` |
| 短板 C · features/ 迁移 | ✅ 已落地 | 11 个老 feature + 5 个新 feature 共 16 个 `features/<name>/`;`infra/` 含 state / llm / sandbox / websearch / trace_sink;`services/` 与 `routers/` 目录消失 |
| 短板 B · domain entity 拆 ORM | ✅ 已落地(基础设施) | `domain/` 6 个 entity + 不变式方法;`db/mapping.py` 双向 mapping;**opt-in**,老 caller 不动 |
| Redis 状态面 | ⏸ 未做 | 等部署需要;短板 A 已经把口子收好,实现 = 加一个 `RedisStateStore` |
| 多 pod 拓展 | ⏸ 未做 | 等部署需要;依赖 Redis;`domain.Workspace.owning_pod_id` 字段已预留 |

**留作后续的小尾巴**:

- `tools/bg.py` 还跨 sub_agent + sandbox_runner 两 feature 注册工具;切干净 = 把它拆成 `features/sub_agent/tool.py` + `features/sandbox_runner/tool.py`。
- `features/chat/` 内部还混 compact / pre_inject 两类逻辑(v1 doc 计划把它们拆出为独立 feature);属于"代码再组织",不是 git mv,留作触发性重构。
- domain entity 的实际接管:目前老路径仍直接用 ORM。要让 `Session.archive()` 这类不变式真正强制起来,需要 router/service 改为先 `to_session()` → 方法调用 → `apply_session()` → 写库。当前没有"复制粘贴的规则需要收口"的实际触发点,留作 trigger-driven 迁移。
>
> **顺序是技术依赖,不是产品优先级**:
> 1. 先补 **三个短板**(单进程内可完成,不需外部基建)
> 2. 再上 **Redis 状态面**(让进程间通信成为可能)
> 3. 最后 **多 pod 拓展**(workspace 亲和性路由,踩在前两步之上)
>
> 跳步会回头返工。

---

## 第一部分 · v1 留下的三个短板

每个短板独立可推进。这一部分**单进程下就可以做完**,不需要 Redis 或多 pod 基建,但前两个短板是后面所有跨进程工作的前提。

### 短板 A · 进程内 module-level state

#### 现状

业务流转里的"实时事件 / 注册表"全部是 module-level dict。单 binary 部署下没问题,但任何水平扩容、热重载、跨进程协作都不行。

| 文件 | 进程内状态 | 用途 |
|---|---|---|
| `app/runner.py` | `_runners: dict[str, TurnRunner]` | 主 turn 流式事件 pub/sub + 重连重放 |
| `app/bg_runner.py` | `_runners: dict[str, SubAgentRunner]` | sub_agent 引用 + 取消 |
| `app/process_runner.py` | `_runners: dict[str, ProcessRunner]`,`_active_sandboxes: set[str]` | sandbox 进程引用 + 并发上限 |
| `app/bg_hub.py` | `_channels`(session SSE 订阅者)、`_slots`(signal waiter 队列)、`_wake_waiters`(任意活动唤醒) | 跨任务通信 |
| `app/services/chat/turn.py` | `_digest_locks: dict[str, asyncio.Lock]` | 防同 session 并发 spawn digest turn |
| `app/jobs/runtime.py` | `_JOBS: dict[str, JobHandle]` | 长跑 job 注册表 |
| `app/features/impersonation/service.py` | `_TOKENS`,`_AUDIT`(deque) | 借身份 token + audit ring |
| `app/kernel/pipeline.py` | `_HOOKS: dict[Stage, list[Hook]]` | hook 注册 — 这个**不需要外移**,启动时构建,运行期只读 |
| `app/kernel/tool_registry.py` | `_REGISTRY: dict[str, Tool]` | 同上,启动构建 + 只读 |
| `app/kernel/event_bus.py` | `_HANDLERS: dict[str, list[HandlerFn]]` | 同上,启动构建 + 只读 |

后三个是**启动期写入、运行期只读**,不算可变状态。前八个全是运行期可变,要么持久化到 DB,要么换成跨进程基建。

#### 边界已就位

每一个上述模块都是**函数级 API**(`register` / `get` / `publish` / `subscribe` / `emit_signal` / `wait_for_signal` / ...),caller 不知道实现。换实现 = 改一个文件,不动 caller。这是 v1 攒下的资产。

#### 做法

短板 A 在单进程下能做的是**收紧抽象 + 把可变状态从 module-level 抬到注入式**,不真正解决跨进程问题(那是 Redis 那一节的事)。当前推进路径:

1. **`StateStore` protocol** 定义 `register / get / publish / subscribe / emit_signal / wait_for_signal` 接口。
2. 单进程实现 `InProcessStateStore` = 当前所有 dict 的封装。
3. `runner` / `bg_hub` / `bg_runner._runners` / `process_runner._runners` / `_digest_locks` / `_JOBS` / `impersonation._TOKENS` 接口从 module-level dict 改成"从 ContextVar / 单例拿 StateStore"。
4. 此时:依然单进程,但**换 Redis 实现 = 一个文件**,不再要扫一遍 8 个模块。

工作量:~1.5 工作日,纯重构,无功能变更。**这一步做完才是 Redis 工作的前置就绪状态。**

---

### 短板 B · 持久化与领域分离

#### 现状

`db/models.py` 的 SQLAlchemy ORM 类(`User` / `Session` / `Turn` / `Message` / `Memory` / `BgTask` / `Workspace`)**直接漂出 repo**,经过 router 进入 frontend 视图层。

```
ORM (db/models.py)  ──直接→  repo  ──直接→  service  ──直接→  router  ──_view()→  JSON
```

后果:
- 无聚合根。`Session.archive()` 这样的规则散在 `routers/admin.py` + `db/repo/sessions.py` + 前端;每加一处 archive 入口就要重新对一遍。
- 想换 ORM(eg. SQLModel / pure dataclass + asyncpg)= 改一片。
- 想加缓存层 / 读时投影 = 没地方插。
- ORM lazy-load 在异步代码里偶尔表现迷:`m.workspace.fs_path` 这种行为耦合到 session lifetime。

#### 做法

经典 DDD 解法:加 `domain/` 层,纯 dataclass entity 不知 ORM 存在;repo 负责双向翻译。

```
domain/                  纯 Python entity(dataclass)+ 不变式方法
  ├── session.py         Session(id, user_id, workspace_id, archived_at, ...)
  │                      Session.archive() / Session.can_rewind(msg_id) / ...
  ├── turn.py
  ├── message.py
  └── ...

infra/db/                  
  ├── models.py          SQLAlchemy ORM(原 db/models.py)
  ├── repo/<domain>.py   接口:取出 → 翻译成 domain entity 返回
  │                      存入:domain entity → ORM mapping → commit
  └── mapping.py         双向 mapping(可手写或用 dataclass_factory / cattrs)
```

router 拿到 domain entity,业务规则全在 entity 上:

```python
sess: Session = await repo.get_session(sid)
sess.archive()                          # 内部 raise 如果已经 archived;调用方 try/except
await repo.save(sess)
```

#### 做法的代价

- 每个 repo 函数多 5-10 行 mapping。
- service 大量 `await repo.get_X()` 变成 domain entity 调用 + `await repo.save()` 双步。
- 现有 frontend SSE / REST 视图字段名要重新对齐(应该问题不大,中间加一层 `<entity>_to_view()` 就行)。

工作量:~3 工作日,**碰所有 service**,要全套手工冒烟。

#### 这一步**是否值得现在做**?

诚实判断:**目前不是最痛**。Oryx 的写规则不复杂(archive / fork / cancel / compact),没出现"同一规则在四处 if-else 各写一份"的尾大病。
- 强建议触发条件:出现第三个 `if not sess.user_id != user.id and not user.is_admin: raise` 类似的"应该长在 entity 上的规则"被复制粘贴时,立刻做。
- 否则可以 Redis + 多 pod 都做完再看。

---

### 短板 C · features/ 未贯彻

#### 现状

v1 (P6) 只放了 `features/` scaffold,**新增 feature**(next_options / auto_research / impersonation)进去了。**老 feature** 仍住在分层目录:

| 老 feature | 当前位置 | 目标位置(v1 doc 第三部分对照表) |
|---|---|---|
| chat 编排 | `app/services/chat/{turn,compact,messages,bootstrap}.py` | `features/chat/` + `features/compact/` + `features/pre_inject/` |
| sub_agent | `app/bg_runner.py` + `app/bg_hub.py` + `app/bg_emit.py` + `app/tools/bg.py`(部分) | `features/sub_agent/` |
| sandbox | `app/process_runner.py` + `app/tools/bg.py::run_sandbox` | `features/sandbox_runner/` |
| workspace fs ops | `app/tools/fs.py` + `app/routers/workspace.py` + `app/workspace.py` | `features/workspace/` |
| memory | `app/tools/memory.py` + `app/routers/memory.py` | `features/memory_store/` |
| skills | `app/skills_loader.py` + `app/routers/skills.py` + `app/tools/skill.py` | `features/skills/` |
| shell tool | `app/tools/shell.py` + `app/services/lite/shell.py` | `features/shell_tool/` |
| web tool | `app/tools/web.py` + `app/services/lite/_client.py`(部分) + `app/services/websearch/` | `features/web_tool/` + `infra/websearch/` |
| auth | `app/auth.py` + `app/routers/{auth,invites}.py` + `app/capabilities/auth_context.py` | `features/auth/` + 保留 capabilities |
| admin | `app/routers/admin.py` | `features/admin/` |
| 4 个 lite gate | `app/services/lite/{triage,memory,skill,insight}.py` | `features/{triage_guard,memory_ranker,skill_gate,spawn_advisor}/` |
| trace_sink | `app/trace_sink/` | `infra/trace_sink/` |

#### 为什么没做

不是技术债,是**有意识延后**:
- 每次迁移 = 大批 import path 改 + 风险大、收益小(没新能力)。
- v1 时机:刚搭完 kernel + capabilities + features/ scaffold,搬运负担最重;让新代码先沉淀几个月。
- 触发条件(出现一个就动):
  1. 加第 8 个 feature 时(`features/` 里已经有 6 个,杂目录已经有 12 个 + 分散文件)
  2. 多人协作出现「修一个 feature 要碰 5 个目录」的实际抱怨
  3. 想做 feature-level test fixture 但跨目录依赖太乱

#### 做法

机械活,无技术难点。按 v1 doc 第三部分对照表逐个搬:
1. 一次一个 feature,新位置建好,把代码 git mv 过去
2. 旧位置留 thin re-export `from features.foo.service import *  # back-compat`
3. 全局 grep 旧 import path 更新
4. 跑全部冒烟场景(同 v1 README 列的 7 个 happy path)
5. 旧 re-export 文件保留 1-2 周观察,无引用后删

工作量:**~2.5 工作日**(11 个 feature × 0.2d 平均)。每个 feature 单独 commit。

#### 这是否解锁了 Redis / 多 pod?

**几乎不**。Redis / 多 pod 跟代码物理位置无关,跟接口边界有关 — 那个 v1 已经做完了。

但完成 C 之后,**"workspace 这个 feature 的多 pod 改造"读 / 改起来心智成本低**:所有 fs / 路由 / 上传 / sandbox 路径都在一个目录里。

---

## 第二部分 · Redis 状态面

> **前置**:短板 A 完成(StateStore 抽象 + InProcessStateStore 默认实现)。
> 没做完短板 A 就上 Redis = 在 8 个模块各自手写 Redis 客户端,负债翻倍。

### Redis 接管什么

| 内容 | 当前 | Redis 后 |
|---|---|---|
| Turn / sub_agent / process runner 注册表 | module dict | **不上 Redis**。注册表本质是"哪个 pod 上有这个 task 的 in-memory state",改成 DB 写一行 `tasks.owning_pod_id`;别 pod 想 cancel 就发跨 pod 信号到 owning pod |
| 实时事件 pub/sub(`bg_hub.publish` / `subscribe`)| in-process 队列 | **Redis Pub/Sub**(每 session 一个 channel `session:<sid>`),订阅在每个 pod 各拉一份 |
| 命名信号(`bg_hub.emit_signal` / `wait_for_signal`)| in-process slot | **Redis Streams**(`XADD signal:<sid>:<name> ...`,`XREAD BLOCK 0`);terminal 状态用 sticky key |
| 任意活动唤醒(`wait_for_seconds`)| in-process Event | 同 Pub/Sub,session channel 上任何消息都 wake |
| 数字 spawn lock(`_digest_locks`)| in-process Lock | **`SET key NX EX 30`** 即可;持有锁的 pod 完事后 DEL |
| 借身份 token(`_TOKENS`)| in-process | **Redis hash + TTL**:`HSET imp:<token> admin_id ... target_id ...; EXPIRE imp:<token> 900` |
| Audit ring(`_AUDIT`)| in-process deque | **Redis Stream**(`XADD audit:<admin_id> ...` + `XLEN` 上限) |
| Job runtime(`_JOBS`)| in-process | **改成 DB 表 `jobs`**(状态机持久化);不进 Redis |

**Redis 只做 in-flight signaling**;business state 留在 PG。这条边界守住,Redis 挂了重启 = 丢正在 wait 的 sub_agent 信号(已经在 DB 的 bg_task 状态不丢),业务可恢复。

### 实现要点

1. `infra/state/redis.py` — `RedisStateStore` 实现短板 A 定义的 `StateStore` Protocol。
2. `aioredis` / `redis.asyncio` 已成熟,选 `redis-py >= 5.0`(原生 async)。
3. **不引连接池抽象层**;直接用 `redis.asyncio.Redis`。
4. 连接配置走 `settings.REDIS_URL`,空字符串 fallback 到 `InProcessStateStore`(dev 单进程依旧能跑)。
5. `bg_hub.subscribe(sid)` 内部:每个 pod 启动一个 redis subscriber per active session;池子用 `dict[str, asyncio.Task]` 管,session 没活动 30s 后 cancel。
6. `wait_for_signal` 用 Streams `XREAD BLOCK`,timeout 走 redis 本地 timeout 而不是 asyncio.wait_for。

工作量:~2 工作日(`RedisStateStore` 1d + cutover 0.5d + 测试 0.5d)。

### 不必做的

- **不必**把 trace_sink 改 Redis(它已经走 file / ClickHouse)。
- **不必**把 DB 状态搬 Redis(没收益)。
- **不必**对 redis 加 cluster 配置(单 master 跑得起 99% 用例)。

---

## 第三部分 · 多 Pod 拓展(workspace 亲和性)

> **前置**:Redis 已上(跨 pod cancel / drawer / 实时事件靠它);短板 A / C **强烈建议**先完成,B 可后。

### 核心约束

每个 workspace 的文件实际落在某一台 host 的本地盘上。原因:
- Docker sandbox `--mount type=bind` 走 NFS 性能崩
- uv cache mount 假设本地 SSD
- workspace 一旦"在 pod-A 的盘上",pod-B 看不到

→ 必须 **session/workspace_id → pod_id 亲和**,非该 pod 的请求被路由 / 代理。

### 新增基础设施(总 ~2.5 工作日)

#### 1. Workspace 归属表(0.5d)
```sql
ALTER TABLE workspaces ADD COLUMN owning_pod_id VARCHAR(64);
CREATE TABLE pods (
    pod_id VARCHAR(64) PRIMARY KEY,
    hostname VARCHAR(128),
    last_seen TIMESTAMP,
    status VARCHAR(16)  -- 'active' | 'draining' | 'dead'
);
CREATE INDEX idx_pods_active ON pods(status, last_seen);
```

#### 2. Pod 自我意识 + 心跳(0.3d)
- `settings.POD_ID`:从 `HOSTNAME` env 取(k8s 的 StatefulSet 保证稳定)
- `main.lifespan` 启动时 `UPSERT pods(pod_id, hostname, last_seen=now(), status='active')`
- 后台 task 每 5s 更新 `last_seen`,关停时 `status='draining'`
- 另一个后台 task 每 30s 扫 `last_seen < now() - 60s` 标记 `status='dead'` — 触发后面的 failover

#### 3. Workspace 分配策略(0.4d)
- 创建 workspace 时 `new_workspace_root` 不再直接 `mkdir`:先选 pod(consistent hash on `user_id`,或 `active` pod 中 owning_count 最少的),写 `owning_pod_id`,本 pod 是 owner 就 mkdir,否则 RPC 让 owner pod mkdir
- v1 已有的 `register_workspace_path` 仅本 pod 的 owning workspace 调用

#### 4. 路由中间件(1d) **这是关键**

FastAPI middleware 拦截所有 `/api/*` 请求,提取归属用的 id:

| URL pattern | 归属字段 |
|---|---|
| `/api/sessions/{sid}/**` | sid → session.workspace_id → ws.owning_pod_id |
| `/api/turns/{tid}/**` | tid → turn.session_id → ws.owning_pod_id |
| `/api/bg-tasks/{bg_id}/**` | bg_id → bg_task.parent_session_id → ... |
| `/api/workspace/{sid}/**` | 同 sessions |
| `/api/chat`(POST,带 `sessionId`)| 解 body 取 sessionId |
| `/api/auth/**` / `/api/workspaces`(POST 新建)/ `/api/sessions`(POST 新建)| 任意 pod |
| `/api/admin/**` | 任意 pod(admin RPC 跨 pod 时各自查) |

判定流程:
```
1. middleware 解出 ws_id(用 1 个 redis cache key 加速:sid → ws_id, TTL 60s)
2. 查 owning_pod_id
3. 等于 self.POD_ID  → 放行
4. 不等 → 返回 307 Temporary Redirect,Location 指向 owner pod
   (StatefulSet 给每个 pod 稳定 DNS:`oryx-2.oryx-headless.ns.svc:8000`)
5. SSE 请求同样 307;curl / fetch 默认会跟随 redirect,EventSource 也会
```

为何用 307 而不是 nginx-level routing:
- 应用层判定信息全,nginx 还得复刻一遍 DB 查询
- 307 保留 method + body,POST 不会变 GET
- redirect 一次性事件,延迟可忽略
- 真要 nginx 化以后,把 middleware 逻辑搬到 OpenResty/Lua + Redis lookup

#### 5. k8s 部署形态(0.3d YAML)

```yaml
# StatefulSet, not Deployment — 给每个 pod 稳定 hostname 和 PVC
kind: StatefulSet
spec:
  serviceName: oryx-headless    # 配套 headless Service
  replicas: 3
  template:
    spec:
      containers:
        - name: oryx
          env:
            - name: POD_ID
              valueFrom: { fieldRef: { fieldPath: metadata.name } }
            - name: REDIS_URL
              value: "redis://redis:6379/0"
            - name: DATABASE_URL
              value: "postgresql+asyncpg://..."
          volumeMounts:
            - name: workspaces
              mountPath: /root/.oryx-workspaces
  volumeClaimTemplates:
    - metadata: { name: workspaces }
      spec:
        accessModes: [ ReadWriteOnce ]
        resources: { requests: { storage: 200Gi } }
---
# Headless Service:每个 pod 一个 DNS — oryx-0/1/2.oryx-headless.ns.svc.cluster.local
kind: Service
metadata: { name: oryx-headless }
spec:
  clusterIP: None
  selector: { app: oryx }
---
# 普通 ClusterIP Service:接受随机进来的请求,middleware 内部 redirect 到正确的 pod
kind: Service
metadata: { name: oryx }
spec:
  selector: { app: oryx }
```

#### 6. Failover(0.5d)

Pod-A 心跳 60s 没更新 → 后台 task 标 `dead`:
1. 把 pod-A 的 `owning_pod_id` 在 `workspaces` 表批量 reassign 给 active pod 之一(consistent hash 选目标)
2. 触发目标 pod 上的"workspace 接管"流程:从持久存储(PV / S3 / 别的 pod 的 rsync)拉 workspace 文件
3. `repo.list_turns_by_status(["running"])` 在新 owning pod 上的 boot resume 会接住

最简单版:**不做主动 failover**,workspace 文件假设丢失,用户重新上传。**生产**版本需要持久卷(k8s PV)或定期 rsync 到对象存储。这部分超出代码范围,属于运维选择。

#### 7. sub_agent / sandbox 永远 owning-pod-local(0.1d)

- middleware 已经把请求路由到 owning pod
- 内部 `bg_runner.spawn` / `process_runner.spawn_sandbox` 加 assert:必须 `workspace.owning_pod_id == settings.POD_ID`,否则报错(意味着 middleware 有 bug)

---

## 第四部分 · 完成后是不是 DDD+others?

把以上全做完 — 三短板 + Redis + 多 pod — 结构会长这样:

```
backend/app/
├── kernel/         #(P1-P10 已有)tool / loop / pipeline / event_bus / sse_protocol — 不知 feature
├── domain/         #(B 引入)纯 entity + 不变式;不依赖 ORM
├── infra/          #(C 引入)db.repo / llm / sandbox / state(redis|in_process) / trace_sink / websearch
├── capabilities/   #(P4 已有)auth_context / permission / audit
├── features/       #(C 完成后)每个 feature 自治目录
└── jobs/           #(P8 已有 + Redis 后持久化到 DB)
```

### 对照 DDD 检查表

| DDD 元素 | 完成后状态 | 备注 |
|---|---|---|
| Bounded Context | ✓ | `features/` 各自闭包,kernel 不知道 |
| Aggregate Root | △ | entity 有了,但聚合根的"`Session.archive()` 是修改 Session 的唯一入口"这个**纪律**不强制 |
| Entity vs Value Object | △ | entity 拆出来了;value object(`MemoryScope` / `SessionId` / `ModelName` 作为类)**没做** |
| Repository | ✓ | `infra/db/repo/` 形态对,带 mapping |
| Application Service | ✓ | `features/<x>/service.py` |
| Domain Service | △ | 没有显式 `domain_service/` 层,目前混在 application service 里 |
| Domain Event | ✓ | `event_bus` |
| Ubiquitous Language | ✗ | 词表是 LLM 工程域(turn / iter / compact / emit / mid_inject),不是业务域 — Oryx 没"业务域" |
| Anti-Corruption Layer | ✓ | `kernel/llm_loop.py` + `infra/sandbox` Protocol + `infra/websearch` Provider |
| Hexagonal Ports & Adapters | ✓ | trace_sink / sandbox / websearch / state 全是 Protocol + 多 adapter |
| Capability Layer | ✓ | `capabilities/` ContextVar 横切 |
| CQRS | ✗ | 没有读写分离;不需要 |
| Event Sourcing | ✗ | 没有;DB 是状态快照,trace_sink 是事件历史(只读 audit 用) |

### 一句话回答

**不是教科书 DDD,是「strategic-DDD + hexagonal + 事件总线 + 横切 capability + 水平扩容状态面」的混合体**。或者更准确:

> **DDD strategic patterns** (bounded contexts, ubiquitous-language-of-the-tech-domain, ACL, repositories)
> **+ Hexagonal**(infra Protocols)
> **+ Event-Driven Light**(event bus,不 sourcing)
> **+ Capabilities**(ContextVar 横切)
> **+ Horizontally-Scalable State Plane**(Redis pub/sub + Streams,DB 持久化)
>
> **− DDD tactical patterns**(value objects、aggregate root invariant 强制、CQRS)

DDD tactical 那块**不做是有意识的**:Oryx 的写规则不丰富到值得 entity-method-only 单一修改入口,加这层不变式守卫等于给薄逻辑套厚壳子。

要更靠近 DDD,触发条件是:
- 出现第三个 `if not is_admin: raise` 类型规则被复制粘贴 → 加 aggregate root 方法
- 出现"Scope 是 'user' 还是 'workspace' 还是 'session'?" 的 bug 来回看 → 引入 value object 替换 Literal

不到这两条触发,留着别动。

---

## 第五部分 · 时间线

```
短板 A(StateStore 抽象 + 单进程实现)        1.5d   独立可上线
短板 C(features/ 老 feature 迁入)            2.5d   独立可上线
短板 B(domain/ + repo mapping)                3.0d   独立可上线;触发条件未到可推迟
─────────────────────────────────────────────  
Redis 状态面                                   2.0d   依赖 A
─────────────────────────────────────────────  
多 pod 拓展                                    2.5d   依赖 Redis;workspace 失效保护额外算
─────────────────────────────────────────────  
                          关键路径合计:        8.0d (A → Redis → Pods,跳过 B 和 C 的延迟形态)
                          全做完合计:         11.5d
```

**最小可上线的多 pod 路径**:A → Redis → Pods = 6 工作日。
**全套**(含 B 和 C 并联):11.5 工作日。

C 可以**并行**插在 A / Redis / Pods 任意阶段做,因为它只是 git mv;B 推迟到出触发条件再做。

---

## 第六部分 · 决策记录

| # | 决定 | 理由 |
|---|---|---|
| R1 | 短板 A 必须先做,然后才 Redis | StateStore 抽象不先做,Redis 实现要在 8 个模块手抓接口,负债 ×2 |
| R2 | Job runtime 不上 Redis,改 DB 表 `jobs` | Redis 适合 in-flight signal;job 是持久状态机 |
| R3 | 跨 pod 路由用 application-level 307,不用 nginx | 判定信息在应用层,nginx 复刻 DB 查询不划算 |
| R4 | StatefulSet,不用 Deployment | 需要稳定 hostname → DNS 给 307 跳转 |
| R5 | Workspace failover **不上代码**,留运维 | k8s PV / 对象存储 / rsync 的选择是部署级别决定 |
| R6 | 短板 B 推迟到触发条件出现 | 当前写规则不复杂到值得 domain entity 单一入口 |
| R7 | 短板 C 可以并行,且鼓励**一次 mv 一个 feature** | 11 个 feature,机械活,小步走风险低 |
| R8 | DDD tactical patterns(value object / aggregate root invariant)**不强行落地** | 项目规模未到 |

---

*最后更新:2026-05-20*
