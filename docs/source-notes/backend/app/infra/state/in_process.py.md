# `backend/app/infra/state/in_process.py`

> 316 行 · 32 个符号　|　[源码导读对应模块](../source-guide/index.md)

**整体作用**：

<!-- 一句话写这个文件干嘛的 -->

## 符号清单（对照手写）

- `class _SessionChannel` · L20
- `class InProcessPubSub` · L25
    - `def __init__()` · L28
    - `def _chan()` · L32
    - `def publish()` · L39
    - `def subscribe()` · L54
- `class _SignalSlot` · L70
- `class InProcessSignalHub` · L76
    - `def __init__()` · L86
    - `def _slot()` · L90
    - `def emit_signal()` · L98
    - `def wait_for_signal()` · L116
    - `def has_waiter()` · L156
- `class InProcessWakeHub` · L163
    - `def __init__()` · L166
    - `def wake_all()` · L169
    - `def wait_for_seconds()` · L173
- `class InProcessLockManager` · L214
    - `def __init__()` · L217
    - `def lock()` · L220
- `class _KVEntry` · L231
- `class InProcessKVStore` · L236
    - `def __init__()` · L239
    - `def set()` · L242
    - `def get()` · L245
    - `def delete()` · L254
- `class InProcessStreamLog` · L260
    - `def __init__()` · L263
    - `def append()` · L267
    - `def tail()` · L274
- `class InProcessStateStore` · L301
    - `def __post_init__()` · L311

<!-- 写法建议：在某个符号下另起一行缩进写它的职责/入参/副作用/坑 -->

