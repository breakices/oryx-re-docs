# `backend/app/db/repo/users.py`

> 228 行 · 17 个符号　|　[源码导读对应模块](../source-guide/index.md)

**整体作用**：

<!-- 一句话写这个文件干嘛的 -->

## 符号清单（对照手写）

- `def ensure_default_user()` · L23
- `def get_user()` · L66
- `def get_user_by_email()` · L71
- `def update_user()` · L77
- `def get_user_llm_settings()` · L83
- `def upsert_user_llm_settings()` · L88
- `def create_user()` · L118
- `def create_invite()` · L135
- `def get_invite()` · L150
- `def list_invites()` · L155
- `def list_invite_uses()` · L172
- `def append_invite_use()` · L181
- `def update_invite()` · L190
- `def create_auth_session()` · L198
- `def get_auth_session()` · L210
- `def touch_auth_session()` · L215
- `def delete_auth_session()` · L224

<!-- 写法建议：在某个符号下另起一行缩进写它的职责/入参/副作用/坑 -->

