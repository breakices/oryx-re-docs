# `backend/app/infra/state/redis.py`

> 439 行 · 51 个符号　|　[源码导读对应模块](../source-guide/index.md)

**整体作用**：

<!-- 一句话写这个文件干嘛的 -->

## 符号清单（对照手写）

- `_CONSUME_SIGNAL_SCRIPT` · L16
- `_RELEASE_LOCK_SCRIPT` · L31
- `def _json_dumps()` · L39
- `def _json_loads()` · L43
- `def _urls()` · L51
- `def _key_part()` · L55
- `def _cluster_auth_kwargs()` · L59
- `def _create_client()` · L71
- `class RedisWakeHub` · L103
    - `def __init__()` · L104
    - `def _channel()` · L108
    - `def wake_all()` · L111
    - `def wait_for_seconds()` · L114
- `class RedisPubSubBus` · L150
    - `def __init__()` · L151
    - `def _channel()` · L156
    - `def publish()` · L159
    - `def subscribe()` · L163
- `class RedisSignalHub` · L180
    - `def __init__()` · L181
    - `def _slot_key()` · L191
    - `def _wait_channel()` · L195
    - `def emit_signal()` · L199
    - `def wait_for_signal()` · L214
    - `def has_waiter()` · L275
    - `def _consume()` · L278
- `class RedisLock` · L288
    - `def __init__()` · L289
    - `def __aenter__()` · L296
    - `def __aexit__()` · L306
    - `def _renew_loop()` · L312
- `class RedisLockManager` · L325
    - `def __init__()` · L326
    - `def lock()` · L331
- `class RedisKVStore` · L339
    - `def __init__()` · L340
    - `def _key()` · L344
    - `def set()` · L347
    - `def get()` · L350
    - `def delete()` · L353
- `class RedisStreamLog` · L357
    - `def __init__()` · L358
    - `def _key()` · L364
    - `def append()` · L367
    - `def tail()` · L373
    - `def _read_key()` · L395
- `class RedisStateStore` · L405
    - `def __init__()` · L406
    - `def ping()` · L422
    - `def close()` · L425
- `def _wait_pubsub_message()` · L433

<!-- 写法建议：在某个符号下另起一行缩进写它的职责/入参/副作用/坑 -->

