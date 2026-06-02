# `backend/app/lib/errors.py`

> 89 行 · 9 个符号　|　[源码导读对应模块](../source-guide/index.md)

**整体作用**：

<!-- 一句话写这个文件干嘛的 -->

## 符号清单（对照手写）

- `class ErrorEnvelope(BaseModel)` · L17
- `_STATUS_CODES` · L26
- `def _trace_id_from_request()` · L37
- `def _envelope()` · L44
- `def install()` · L49
    - `def _http_exc()` · L54
    - `def _val_err()` · L67
    - `def _value_err()` · L81
    - `def _catch_all()` · L85

<!-- 写法建议：在某个符号下另起一行缩进写它的职责/入参/副作用/坑 -->

