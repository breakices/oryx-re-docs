# `backend/app/jobs/runtime.py`

> 89 行 · 9 个符号　|　[源码导读对应模块](../source-guide/index.md)

**整体作用**：

<!-- 一句话写这个文件干嘛的 -->

## 符号清单（对照手写）

- `class JobHandle` · L22
- `class Job(ABC)` · L34
    - `def run()` · L39
- `_JOBS` · L42
- `def submit()` · L45
    - `def _runner()` · L52
- `def get()` · L73
- `def cancel()` · L77
- `def list_all()` · L87

<!-- 写法建议：在某个符号下另起一行缩进写它的职责/入参/副作用/坑 -->

