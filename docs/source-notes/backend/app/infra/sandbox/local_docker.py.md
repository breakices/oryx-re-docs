# `backend/app/infra/sandbox/local_docker.py`

> 245 行 · 16 个符号　|　[源码导读对应模块](../source-guide/index.md)

**整体作用**：

<!-- 一句话写这个文件干嘛的 -->

## 符号清单（对照手写）

- `ORYX_LINE_PREFIX` · L24
- `class SandboxSpec` · L28
- `class SandboxHandle` · L54
    - `def wait_exit()` · L65
    - `def cancel()` · L68
- `class SandboxBackend(Protocol)` · L73
    - `def run()` · L77
    - `def cancel()` · L79
- `class LocalDockerBackend` · L84
    - `def run()` · L90
        - `def _line_iter()` · L152
        - `def _wait_then_set()` · L162
    - `def cancel()` · L207
        - `def _kill()` · L211
- `_backend_singleton` · L230
- `def get_backend()` · L233

<!-- 写法建议：在某个符号下另起一行缩进写它的职责/入参/副作用/坑 -->

