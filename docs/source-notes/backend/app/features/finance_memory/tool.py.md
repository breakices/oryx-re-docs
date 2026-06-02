# `backend/app/features/finance_memory/tool.py`

> 559 行 · 15 个符号　|　[源码导读对应模块](../source-guide/index.md)

**整体作用**：

<!-- 一句话写这个文件干嘛的 -->

## 符号清单（对照手写）

- `def _finance_context()` · L26
- `def _latest_memory_sources()` · L44
- `def _str_list()` · L55
- `class FinanceMemoryQueryTool(Tool)` · L64
    - `def dispatch()` · L109
- `class FinanceMemoryWriteTool(Tool)` · L147
    - `def dispatch()` · L178
- `class FinanceMemoryStageExtractionTool(Tool)` · L211
    - `def dispatch()` · L295
- `class FinanceEntityResolveTool(Tool)` · L343
    - `def dispatch()` · L370
- `class FinanceGraphContextTool(Tool)` · L383
    - `def _clip()` · L416
    - `def _compact_context()` · L422
    - `def dispatch()` · L543

<!-- 写法建议：在某个符号下另起一行缩进写它的职责/入参/副作用/坑 -->

