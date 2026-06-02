# `backend/app/features/auth/router.py`

> 234 行 · 15 个符号　|　[源码导读对应模块](../source-guide/index.md)

**整体作用**：

<!-- 一句话写这个文件干嘛的 -->

## 符号清单（对照手写）

- `router` · L35
- `__all__` · L41
- `def _user_view()` · L44
- `class SignupIn(BaseModel)` · L59
- `class LoginIn(BaseModel)` · L66
- `class TokenOut(BaseModel)` · L71
- `def _expiry()` · L78
- `def _start_session()` · L82
- `def _client_ip()` · L88
- `def _set_auth_cookie()` · L97
- `def _clear_auth_cookie()` · L111
- `def signup()` · L118
- `def login()` · L183
- `def logout()` · L199
- `def me()` · L220

<!-- 写法建议：在某个符号下另起一行缩进写它的职责/入参/副作用/坑 -->

