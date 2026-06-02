# Oryx-RE Roadmap

Single source of truth for "what's shipped / what's planned / what's TBD."
Update at the end of every phase.

---

## 已落地

### Phase 1+2+5 — Bg-task / sub-agent framework

- 表：`bg_tasks`、`bg_events`、`Session.kind`
- `bg_hub`：session 级 SSE 订阅 + 命名信号（sticky）+ 通用 wake 总线
- `bg_runner`：`SubAgentRunner` + `_run_sub_turn`；三模式 `isolated` / `snapshot` / `inherit`；snapshot 支持
  attachments / memory_ids / skill_ids 预注入
- 工具：`spawn_sub_agent` / `final_report` / `report_to_parent` / `wait_for_seconds` / `wait_for_signal` / `get_bg_task`
- 工具预设：`report-only` / `producer`
- 取消语义：wait 工具 cancel-aware；mid-batch cancel 补 synthetic tool 行（避免 OpenAI 400 协议失衡）
- final_report 兜底：sub-agent 用普通正文结束时，runner 合成 final_report event + signal
- auto-fetch：parent 的 `wait_for_signal('bg:<id>:done')` 触发后自动 fake-cycle `get_bg_task`，把 final_report
  注入主上下文
- 路由：`/api/bg-tasks` REST + `/api/sessions/:sid/events` SSE + `bgCounts` 进 `/api/sessions`
- 前端：`useSessionEvents` hook、Sidebar bg badge、ChatPane 头部 `BgHeaderStrip`（dropdown + hover/pin）、
  `BgDrawer` 双栏详情面板、`Message` 渲染 `bg_handoff` 卡片

### 已合入的小型修复
- `MainTurn._db_msg_to_openai` 识别 `bg_handoff` kind，把 payload 折成 system 行可读
- Bubble `contain: paint` 把 chat header 抬到 z-index: 50 的修复
- composer 拆 slash 命令 + `/compact`

---

## 未落地（按优先级，下次决定时按此参照）

### 通知协议 v2 · 简化设计（当前在做）

**核心抽象**

```
main 状态：running | complete
  · running  = 主 turn 活着（含 wait_* parked 状态，等同对待）
  · complete = 没有活跃主 turn
```

**信道（统一化）**

- 任何 content 投递永远走 mid_inject 路径
  · running 时：注入到当前 turn 的下个 iter
  · complete 时：先起新 turn，作为开头的 mid_inject
- `wait_for_signal` 退化为 ack-only：仅作"对应 sub 有 emit 了"的通知，不携带 content；返回 `{ok, fired, terminal, signal}` 无 content 字段

**bg_hub 极简**

```
per (sid, name):
  waiters: list[Event]
  count:   int      # 已发生但还没 wait 的 emit 个数
  terminal: bool    # sub 是否终态

emit_signal(sid, name, terminal=False):
  if waiters: 弹一个 set()
  else:       count += 1
  if terminal: 标记 terminal=True，wake 全部 waiter
  _wake_all(sid)

wait_for_signal(sid, name, timeout):
  if terminal: return {fired:true, terminal:true}
  if count>0:  count-=1; return {fired:true, terminal:false}
  park; race timeout/cancel
```

无 queue / sticky / payload / revoke / delivered_via。

**Tool_call 重平衡（核心新机制）**

- `wait_for_signal` 改为 deferred dispatch：作为后台 `asyncio.Task` 启动，不阻塞 iter
- iter 通过 `runner.iter_trigger` 事件唤醒，触发源：deferred 完成 / 新 pending_input 到达
- DB 持久化：assistant 行 `tool_calls` 字段始终保留完整列表；tool 行落在 `next_position`（自然顺序，可能远离父 assistant）
- API 序列化 (`build_api_messages`)：
  · 每条 assistant 的 `tool_calls` 过滤为"自然衔接"的项（紧邻的 tool 行所对应）
  · 后续 deferred tool 行用 synthetic assistant 包裹追加到尾部，**绝不重排既有位置**
  · 保证 KV cache prefix 单调增长 → cache 命中
- Turn end 清理：所有未完成的 deferred 任务取消，持久化 `{ok:false, cancelled:true}` 兜底行

**Digest turn（complete 状态的 sub-emit）**

```
锁内:
  1. system 行 kind='bg_divider'         ← 横线，compact_boundary 同壳
  2. user 行 kind='continue_trigger'     ← 隐藏，作为新 turn 的 user_message_id 锚点
  3. TurnPendingInput(state='A', source='sub_agent:<bg>', content=emit_content)
  4. turn(meta.kind='digest_bg')
  5. spawn _run_turn task
  6. session SSE 发 spawn chunk → 前端 attach

iter 1 顶部 pop_pending_a 自然弹出 sub 内容 → 落 mid_inject → LLM call
```

**消息行 / Turn 类型**

```
role       kind                  含义
─────      ──────────────────    ──────────────────────────────
user       null                  常规首问
user       mid_inject            补充行；source='user'|'sub_agent:<bg>'
user       continue_trigger      隐藏锚点；digest turn 起点
system     bg_divider            "↩ 后台自动触发" 横线
system     compact_boundary      压缩边界
assistant  compact_summary       压缩摘要
```

```
turn.meta.kind   起点               场景
──────────────   ──────             ──────
null             user 行            常规对话 / pending B chain / rewind
compact          无 user            /compact
digest_bg        continue_trigger   sub emit + main complete
```

**禁用 / 保留**

- `_do_emit` 服务层拒绝 `mode='B'`；schema 保留 `mode` 字段
- 旧 `bg_handoff` system 行机制废弃；既有 `_drain_bg_handoffs` 删除

**前端 UI**

```
Composer pending 列表：仅显示 source IS NULL（用户键入的 A/B）
BgDrawer / HeaderStrip：显示 sub-agent in-queue 计数 + bg_task 总览
聊天：
  bg_divider        横线（compact_boundary 同视觉）
  continue_trigger  不渲染
  mid_inject        既有"补充"样式；source='sub_agent:%' 折叠为默认
deferred tool ack 显示：完成时 tool row 落在 DB 尾部，但前端按 tool_call_id 折回原 assistant card 的 tools 列表显示
```

### Phase 3 · 进程后台任务（实验通信框架）

**第一性原理**：实验通信 ≡ "无 LLM 的 sub-agent"。submit / progress / final / cancel / mid_inject 全套
复用现有 bg_task / bg_event / bg_hub / emit_signal 五件套；缺失只在三处：
（A）emit 触发器从 LLM 自反思换为规则引擎；
（B）资源 / 安全包装层；
（C）`_do_emit` 从 bg_runner 上移为公共 `deliver_emit`。

**两条调用路径并存**：
- 主 agent ── `run_sandbox` ──→ process（V1 默认；轻量一次性实验）
- 主 agent ── `spawn_sub_agent(preset='experimenter')` ──→ sub-agent ── `run_sandbox` ──→ process（V1.5 解锁；重 log / 调参循环）

#### V1 (1.0x) — 本地 docker sandbox + uv cache + 主 agent 入口

**核心重构**
- 新模块 `bg_emit.py`：`deliver_emit(bg, content, terminal, source_tag)` 公共 helper
  - `bg_runner._do_emit` / `process_runner._on_exit` 都调它
  - 路径：bg_event 落库 + `emit_signal('bg:<id>',terminal)` + pending_input → 主 turn 或 spawn digest
- 新模块 `process_runner.py`：`spawn_process` / `_run_process`，对偶 `bg_runner.spawn` / `_run_sub_turn`

**Backend protocol**（为未来 docker 化留接口）
```python
class SandboxBackend(Protocol):
    async def run(self, spec: SandboxSpec) -> SandboxHandle: ...
    # SandboxSpec.mounts 中所有路径必须是 "宿主 docker daemon 可见的 absolute path"
```
- V1 实现 `LocalDockerBackend`（host docker daemon 直接调）
- 选择走 env var `ORYX_SANDBOX_BACKEND=local_docker`（V2+ 加 `ssh` / `dood` / `dind` / `k8s`）
- 路径契约：`repo.get_workspace().fs_path` 已是 host absolute，无需重映射

**子表**
```
bg_processes:
  bg_task_id (FK)
  kind                # 'sandbox' V1 | 'remote' V2 | 'experiment' V3
  source_dir, entrypoint, pip_spec
  resources (JSON: cpu/mem_mb/timeout_s/pids/fd)
  container_id, exit_code, output_path
```

**规则引擎 V1（最小集）**
1. **stdout JSON 心跳协议**：行首 `__ORYX__ ` 的 JSON → 直接 `deliver_emit(content=...)`
   ```
   __ORYX__ {"kind":"progress","content":"epoch 3/10 loss=0.42"}
   ```
2. **exit 终结**：进程退出 → 自动 `deliver_emit(terminal=true)`，内容模板：
   `[exit {code}] artifact: outputs/<bg_id>/ · last_lines: <tail 20>`
3. 用户自发 terminal 心跳覆盖默认模板（若 entrypoint 在 exit 前发了 `terminal:true` JSON）

正则匹配 / metric 提取 / log_pattern DSL 留 V5。

**uv cache 机制**
- host 维护 `/var/lib/oryx/uv_cache/` 全局 cache（持久）
- container run 时 `-v /var/lib/oryx/uv_cache:/root/.cache/uv:ro`
- container 内 `uv pip install -r requirements.txt --offline`，命中 cache 即装，未命中即失败
- base image build 用 BuildKit cache mount 预热常见科学栈：
  ```dockerfile
  RUN --mount=type=cache,id=oryx-uv,target=/root/.cache/uv \
      uv pip install --system numpy scipy pandas torch-cpu pytest
  ```

**docker run flags（V1 固定）**
```
--network=none
--cpus=$CPU --memory=${MEM}m --pids-limit=$PIDS
--read-only --tmpfs /tmp
--cap-drop=ALL --security-opt no-new-privileges
-v /var/lib/oryx/uv_cache:/root/.cache/uv:ro
-v $WORKSPACE/<source_dir>:/work/src:ro
-v $WORKSPACE/outputs/<bg_id>:/output:rw
```

**工具（V1 主 agent + V1.5 experimenter preset）**
```
run_sandbox(source_dir, entrypoint='pytest', pip=None|str|list[str],
            resources={cpu:2, mem_mb:1024, timeout_s:300, pids:256, fd:1024},
            wait=true) → {bg_task_id, ...}
```
- `wait=true`（V1 默认）→ 同步等终结，直接返回最终结果（轻量场景）
- `wait=false` → 返回 bg_task_id，agent 自己 `wait_for_signal`（V1.5 sub-agent 用）

**前端**
- BgDrawer / HeaderStrip 微调：kind='process' 行的 icon / 状态色（kind 字段已预留）

**V1 不做**
- sub-agent spawn process（preset schema 占位，dispatch 报错 "V1.5+"）
- 远程 SSH（V2）
- experiment platform（V3）
- 联网装包"破例 mode"（V2 视需要）
- uv cache LRU / 配额（V5）

**V1 Acceptance Suite —— 蒙特卡洛 π 全流程**

用户输入：`用蒙特卡洛模拟估算圆周率，N=1e7，跑一下，给我结果和误差。`

| # | 验证点 | 通过条件 |
|---|---|---|
| 1 | agent 写代码 | LLM 写出 numpy 脚本，含 `__ORYX__` 心跳 + `/output/result.json` 写入 |
| 2 | uv cache 命中 | numpy 安装 < 1s（无 network） |
| 3 | network=none 真生效 | 故意写 `urlopen` 版本 → connection refused |
| 4 | mem 限制 | N=1e10 OOM → bg_task.error 含 "OOMKilled" |
| 5 | timeout | `time.sleep(120)` → SIGTERM→SIGKILL，error 含 "timeout" |
| 6 | 心跳 → emit | 每条 `__ORYX__` 实时 bg_event(progress)；BgDrawer 可见 |
| 7 | terminal + artifact | `outputs/<bg>/result.json` workspace 可读，final_report 含路径 |
| 8 | 主 agent 消化 | 不复述 raw log；总结一句答 |
| 9 | 取消 | docker kill → status=cancelled → 主 agent 收到 cancelled emit |
| 10 | 迭代 | "N=1e8 再跑一次" → 不重写代码，复用 cache |

#### V1.5 (~0.3x) — sub-agent experimenter preset 解锁

- `_run_sub_turn` 升级：移植 `deferred_calls` + `iter_trigger`，pop_pending_a 在 sub iter top 也执行
- `_maybe_spawn_digest_turn` 泛化支持 `kind='sub_agent'` session
- `SUB_PRESET_TOOLS['experimenter']` dispatch 放开
- 验证：sub-agent 跑 5 组超参 sandbox，最后给主 agent 一份对比表（不传 raw log）

#### V2 (~0.4x) — 远程 SSH（原 Phase 4）

- `machines` 表：id, user_id, name, host, port, username, key_enc, gpu_info, system_info(JSON), last_seen, status
- key 加密：本地 dev 0600 明文 / 部署 Fernet（主密钥 env `ORYX_MACHINE_KEY` / OS keychain）
- 抽象 `SSHBackend` 实现 `SandboxBackend` protocol
- 工具 `run_remote(machine_id, command, cwd?, env?, timeout?)`
  - 完整命令字符串透传（用户授权机器 = 全权代理）
  - asyncssh 长连接 + 流式 stdout/stderr → bg_event
- 监控 worker：5min 周期 ssh 探活 + `nvidia-smi` + `/proc/loadavg`
- 一次授权 UX：machine 绑定时勾选「允许 session 自由使用」（D4）
- machine offline 已派 job：保留 pending + `stale_after_minutes`

#### V3 (~0.3x) — 实验平台预设（原 Phase 6）

- `experiment_platforms` yaml 目录（与 skills 同源）：container_image / env_template / dataset_mount / default_resources
- 工具 `run_experiment(preset_id, source_dir, dataset_refs=[], ...)`：合并 preset → sandbox spec
- `checkpoint(name, state_path)` / `resume_from(bg_task_id, name)`：跨 job ckpt 复用
- 失败预处理：stderr ≤ 200 字直传；超长 lite-model 摘要；重复 traceback 聚类

### Phase 7 · 湿实验 skill 支持

- skill frontmatter `domain: wet-lab` 触发湿实验工具集挂载
- 工具：
  - `confirm_step(prompt)` — 暂停 sub-agent，发通知给用户，用户回复后继续
  - `read_sensor(device_id, channel)` — 抽象设备 API（先 mock）
  - `safety_check` 钩子 — 每步前 lite-model 评估"危险动作"
  - `checkpoint(name, state)` / `resume_from(name)` — 长时实验断点续跑
- bg_task 状态加 `paused-by-user`
- 通知通道：前端 toast + 浏览器 Notification API（外部 email/IM 留 Phase 8+）

### Phase 8+ · 杂项

- 外部通知通道：email / Slack / IM webhook（湿实验长任务用）
- `compact` 后的"二次压缩"策略（summary-of-summaries）—— 等观察实际增长再决
- Memory `hit_count` 接入 lite ranker 作 sort tie-breaker
- 多机调度（一个 bg_task 跨多台）——目前每个 bg_task 单机
- cross-workspace fork —— 湿实验场景克隆 workspace
- DEV-mode toggle UI 暴露给用户（目前 backend env 控制）

---

## 已搁置的设计决定（决定了但未实现，避免重提）

| # | 决定 | 备注 |
|---|---|---|
| D1 | session 删除时若有 running bg_task → 拒绝（不级联）  | 与级联强删两种 UI 入口待 Phase 8 |
| D2 | bg_event urgency=interrupt → 仅 mini-turn，notify → 等下次 /chat | 已被 B 方案的 emit-hook spawn digest 取代（任何 terminal emit 在主死时都 spawn）。interrupt-only mini-turn 不再需要 |
| D3 | container 资源池：平台 light + 用户真机 双队列，先 FIFO + 上限可配 | 具体参数 Phase 3 落地时实测调 |
| D4 | 远程机器绑定 → 一次授权（session 内自由使用，无需每条 shell 弹窗） | Phase 4 |
| D5 | report_to_parent / final_report 等限流参数化 | 与 Phase 3 队列上限同表 |
| D6 | signal 命名空间：per-bg_task（`bg:<id>:...`），跨任务用统一前缀 | 已落地 |
| D7 | artifact 拉回：按 bg_task_id 隔离子目录 `outputs/<bg_task_id>/` + 显式 promote 到 workspace 顶层 | Phase 3 |
| D8 | machine_id 归属 user，workspace 可声明默认 | Phase 4 |
| D9 | kind=process 用独立工具（`pull_artifact` / `spawn_process`），不复用 `read_file` 统一抽象 | Phase 3 |
| D10 | wet-lab confirm_step 仅前端推送通道（Phase 7 起点）；外部通道 Phase 8+ | — |
| D11 | sub-agent 不能 spawn 子 sub-agent（避免不收敛） | 已落地 |
| D12 | 进程 cancel 默认 NOT 级联到子 agent（保留已投入的算力 / 时间） | 用户可手动 cancel 单个 bg_task |

---

## UI 待办（小项，攒着批量做）

- BgDrawer 事件列表：同 bg_task 的 final_report + handoff 在 ≤1s 内合并显示一行
- BgDrawer 详情过滤：默认隐藏 `tool` kind 内部活动事件，加 toggle
- 副标题"消化 N 条后台结果"按钮：BgHeaderStrip 加未消费 bg_event 计数 + 一键触发 digest turn（option 2，先于 D2 mini-turn 上）
- composer pending 队列区分用户 / 子 agent 来源样式
- BgDrawer 详情：sub-agent 全程 prose / tool trace 跳转入口（进入 sub-Session 只读视图）

---

## 测试 / 验证

| 场景 | 测过 | 备注 |
|---|---|---|
| 多 sub-agent 并发读 PDF + 综合汇报 | ✅ s-853301da | 4 个 turn 行无碎碎念 |
| 中途 cancel | ✅ s-d79a728b → fix A | 不再 400 |
| 子 agent fallback（无 final_report 工具）| ✅ s-47da7244 bg-954da3d3 | fallback emit 修复后 |
| 暴力 wait timeout（无信号源）| ⚠️ 未测 | 验证 cancel 路径 |
| inherit 模式（继承父历史）| ⚠️ 未测 | 跨场景 |
| 多层 / interleaving bg 中途 emit progress | ⚠️ 未测 | 等 emit 重构 |
| 长程跨 session 重启 | ⚠️ 未测 | reap_dangling_bg_tasks 是否清得干净 |

---

## 决策注释

- 注释风格延续 codebase：保留 WHY 不保留 WHAT；中文 inline / 英文 block-doc 混用。
- 新增工具的 schema description 写给 LLM 看，必须明确告诉它何时用 / 何时不用 / 与同类工具的区别。
- 任何"幽灵 tool 行 / 失衡 tool_calls"问题第一时间往 `_run_turn` 工具循环 + `mergeMessages` 防御查。

*最后更新：2026-05-16（Phase 3 重整 + V1 落地启动）*
