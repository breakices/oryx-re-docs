# `backend/app/infra/trace_sink/clickhouse_sink.py`

> 172 行 · 9 个符号　|　[源码导读对应模块](../source-guide/index.md)

**整体作用**：

<!-- 一句话写这个文件干嘛的 -->

## 符号清单（对照手写）

- `_SCHEMA` · L27
- `_MIGRATIONS` · L63
- `_INSERT_COLS` · L72
- `class ClickHouseSink` · L84
    - `def __init__()` · L85
    - `def _ensure_client()` · L95
    - `def emit()` · L117
    - `def close()` · L132
- `def _record_to_row()` · L143

<!-- 写法建议：在某个符号下另起一行缩进写它的职责/入参/副作用/坑 -->

