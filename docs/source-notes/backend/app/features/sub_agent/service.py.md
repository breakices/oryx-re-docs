# `backend/app/features/sub_agent/service.py`

> 870 行 · 21 个符号　|　[源码导读对应模块](../source-guide/index.md)

**整体作用**：

<!-- 一句话写这个文件干嘛的 -->

## 符号清单（对照手写）

- `SUB_SYSTEM_PROMPT` · L43
- `FINANCE_SUB_SYSTEM_PROMPT` · L75
- `def spawn()` · L93
- `def cancel()` · L152
- `def _emit()` · L171
- `def _publish_task_state()` · L177
- `def _persist_sub_pre_inject()` · L195
- `def _persist_attach_cycle()` · L244
- `def _persist_memory_cycle()` · L272
- `def _persist_skill_cycle()` · L285
- `def _result_preview()` · L298
- `def _run_sub_turn()` · L308
    - `def _record()` · L324
- `def _discard_later()` · L669
- `def resume_bg_task()` · L674
- `def _make_dispatch()` · L710
    - `def _disp()` · L716
- `def _do_emit()` · L732
- `def _do_wait_for_seconds()` · L796
- `def _do_wait_for_signal()` · L814
- `def _parent_msg_to_openai()` · L853

<!-- 写法建议：在某个符号下另起一行缩进写它的职责/入参/副作用/坑 -->

