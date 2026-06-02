# `backend/app/features/chat/router.py`

> 441 行 · 17 个符号　|　[源码导读对应模块](../source-guide/index.md)

**整体作用**：

<!-- 一句话写这个文件干嘛的 -->

## 符号清单（对照手写）

- `router` · L58
- `def _user_message_meta()` · L61
- `def _assert_writable()` · L90
- `def _assert_owns_session()` · L101
- `def _assert_owns_turn()` · L110
- `def _proxy_if_remote_session_owner()` · L118
- `def chat()` · L130
- `def cancel_turn()` · L206
- `def list_pending()` · L222
- `def add_pending()` · L230
- `def patch_pending()` · L258
- `def delete_pending()` · L279
- `def rewind()` · L294
- `def stream_turn()` · L371
    - `def gen()` · L381
- `def _chat_chunk_schema_only()` · L403
- `def trigger_compact()` · L413

<!-- 写法建议：在某个符号下另起一行缩进写它的职责/入参/副作用/坑 -->

