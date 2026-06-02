# `backend/app/kernel/tool_protocol.py`

> 77 行 · 8 个符号　|　[源码导读对应模块](../source-guide/index.md)

**整体作用**：

<!-- 一句话写这个文件干嘛的 -->

## 符号清单（对照手写）

- `class ToolMeta` · L18
    - `def to_wire()` · L30
- `class Tool(ABC)` · L34
    - `def dispatch()` · L45
- `class FormTool(Tool)` · L50
    - `def dispatch()` · L62
    - `def to_event()` · L66
    - `def persist()` · L70

<!-- 写法建议：在某个符号下另起一行缩进写它的职责/入参/副作用/坑 -->

