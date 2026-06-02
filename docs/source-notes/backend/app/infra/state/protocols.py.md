# `backend/app/infra/state/protocols.py`

> 84 行 · 19 个符号　|　[源码导读对应模块](../source-guide/index.md)

**整体作用**：

<!-- 一句话写这个文件干嘛的 -->

## 符号清单（对照手写）

- `class PubSubBus(Protocol)` · L19
    - `def publish()` · L23
    - `def subscribe()` · L24
- `class SignalHub(Protocol)` · L27
    - `def emit_signal()` · L31
    - `def wait_for_signal()` · L32
    - `def has_waiter()` · L36
- `class WakeHub(Protocol)` · L39
    - `def wait_for_seconds()` · L42
- `class LockManager(Protocol)` · L48
    - `def lock()` · L51
- `class KVStore(Protocol)` · L54
    - `def set()` · L58
    - `def get()` · L59
    - `def delete()` · L60
- `class StreamLog(Protocol)` · L63
    - `def append()` · L67
    - `def tail()` · L68
- `class StateStore(Protocol)` · L74

<!-- 写法建议：在某个符号下另起一行缩进写它的职责/入参/副作用/坑 -->

