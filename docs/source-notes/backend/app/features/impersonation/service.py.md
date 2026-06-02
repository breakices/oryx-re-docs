# `backend/app/features/impersonation/service.py`

> 94 行 · 11 个符号　|　[源码导读对应模块](../source-guide/index.md)

**整体作用**：

<!-- 一句话写这个文件干嘛的 -->

## 符号清单（对照手写）

- `IMPERSONATION_TTL_S` · L18
- `AUDIT_CHANNEL` · L19
- `class ImpersonationToken` · L23
- `class AuditEntry` · L32
- `def _token_key()` · L42
- `def mint()` · L46
- `def lookup()` · L58
- `def revoke()` · L65
- `def record_audit()` · L69
- `def list_audit()` · L82
    - `def _filter()` · L85

<!-- 写法建议：在某个符号下另起一行缩进写它的职责/入参/副作用/坑 -->

