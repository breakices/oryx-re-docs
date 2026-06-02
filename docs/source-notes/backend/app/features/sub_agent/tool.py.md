# `backend/app/features/sub_agent/tool.py`

> 533 行 · 21 个符号　|　[源码导读对应模块](../source-guide/index.md)

**整体作用**：

<!-- 一句话写这个文件干嘛的 -->

## 符号清单（对照手写）

- `def _normalize_string_ids()` · L36
- `def spawn_sub_agent()` · L48
- `_ARG_CLIP_CHARS` · L140
- `def _clip_arg_value()` · L143
- `def inspect_sub_agent()` · L151
- `def get_bg_task()` · L223
- `def wait_for_seconds()` · L252
- `def wait_for_signal()` · L278
- `class SpawnSubAgentTool(Tool)` · L325
    - `def dispatch()` · L363
- `class GetBgTaskTool(Tool)` · L376
    - `def dispatch()` · L396
- `class InspectSubAgentTool(Tool)` · L401
    - `def dispatch()` · L425
- `class WaitForSecondsTool(Tool)` · L433
    - `def dispatch()` · L458
- `class WaitForSignalTool(Tool)` · L466
    - `def dispatch()` · L491
- `class EmitToolStub(Tool)` · L501
    - `def dispatch()` · L527
- `EMIT_SCHEMA` · L532

<!-- 写法建议：在某个符号下另起一行缩进写它的职责/入参/副作用/坑 -->

