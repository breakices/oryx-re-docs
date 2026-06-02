# `backend/app/features/sandbox_runner/service.py`

> 529 行 · 19 个符号　|　[源码导读对应模块](../source-guide/index.md)

**整体作用**：

<!-- 一句话写这个文件干嘛的 -->

## 符号清单（对照手写）

- `DEFAULT_CPU` · L42
- `DEFAULT_MEM_MB` · L43
- `DEFAULT_TIMEOUT_S` · L44
- `DEFAULT_PIDS` · L45
- `DEFAULT_FD` · L46
- `TAIL_LINES` · L51
- `UV_CACHE_HOST` · L56
- `class SandboxBusy(Exception)` · L64
- `def _clamp_resources()` · L68
- `def spawn_sandbox()` · L89
    - `def _guarded()` · L156
- `def cancel()` · L170
- `def _publish_task_state()` · L188
- `def _build_pip_argv()` · L205
- `def _try_parse_oryx_line()` · L224
- `def _run_process()` · L240
        - `def _consume_lines()` · L332
- `def _finish_with_error()` · L476
- `def _discard_later()` · L526

<!-- 写法建议：在某个符号下另起一行缩进写它的职责/入参/副作用/坑 -->

