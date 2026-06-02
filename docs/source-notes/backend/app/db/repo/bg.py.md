# `backend/app/db/repo/bg.py`

> 265 行 · 16 个符号　|　[源码导读对应模块](../source-guide/index.md)

**整体作用**：

<!-- 一句话写这个文件干嘛的 -->

## 符号清单（对照手写）

- `def create_bg_task()` · L20
- `def update_bg_task()` · L54
- `def get_bg_task()` · L60
- `def list_bg_tasks()` · L65
- `def list_bg_tasks_by_status()` · L80
- `def session_bg_counts()` · L100
- `def session_bg_counts_bulk()` · L116
- `def has_running_bg_tasks()` · L134
- `def create_bg_event()` · L149
- `def list_bg_events()` · L172
- `def list_unconsumed_bg_events()` · L191
- `def mark_bg_events_consumed()` · L208
- `def create_bg_process()` · L222
- `def update_bg_process()` · L241
- `def get_bg_process()` · L247
- `def reap_dangling_bg_tasks()` · L254

<!-- 写法建议：在某个符号下另起一行缩进写它的职责/入参/副作用/坑 -->

