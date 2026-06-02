# `backend/app/features/sub_agent/router.py`

> 182 行 · 14 个符号　|　[源码导读对应模块](../source-guide/index.md)

**整体作用**：

<!-- 一句话写这个文件干嘛的 -->

## 符号清单（对照手写）

- `router` · L19
- `def _session_chunk_schema_only()` · L23
- `def _assert_owns_session_or_404()` · L29
- `def _assert_owns_bg()` · L38
- `def _bg_view()` · L46
- `def _ev_view()` · L61
- `def list_bg()` · L71
- `def get_bg()` · L86
- `def list_events()` · L92
- `def cancel_bg()` · L101
- `class WakeIn(BaseModel)` · L114
- `def wake_bg()` · L119
- `def session_events()` · L137
    - `def gen()` · L146

<!-- 写法建议：在某个符号下另起一行缩进写它的职责/入参/副作用/坑 -->

