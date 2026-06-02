# `backend/app/infra/trace_sink/base.py`

> 95 行 · 6 个符号　|　[源码导读对应模块](../source-guide/index.md)

**整体作用**：

<!-- 一句话写这个文件干嘛的 -->

## 符号清单（对照手写）

- `class TraceRecord` · L9
    - `def duration_ms()` · L42
    - `def derive_summary()` · L45
- `class TraceSink(Protocol)` · L92
    - `def emit()` · L93
    - `def close()` · L94

<!-- 写法建议：在某个符号下另起一行缩进写它的职责/入参/副作用/坑 -->

