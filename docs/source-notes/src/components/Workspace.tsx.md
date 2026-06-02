# `src/components/Workspace.tsx`

> 2546 行 · 29 个符号　|　[源码导读对应模块](../source-guide/index.md)

**整体作用**：

<!-- 一句话写这个文件干嘛的 -->

## 符号清单（对照手写）

- `interface TreeItem` · L20
- `type FileKind` · L25
- `interface OpenTab` · L27
- `function buildTree()` · L39
- `function detectKind()` · L60
- `function FileKindIcon()` · L69
- `function fsNodeIconName()` · L73
- `interface RowCallbacks` · L81
- `function ActionMenu()` · L94
- `function Row()` · L164
- `function fmtBytes()` · L227
- `type WorkbenchTab` · L237
- `function isWorkbenchTab()` · L241
- `function financeWorkbenchTabKey()` · L245
- `function readFinanceWorkbenchTab()` · L249
- `function writeFinanceWorkbenchTab()` · L260
- `function statusLabel()` · L270
- `function FinanceOverview()` · L277
- `function FinanceProviders()` · L332
- `function FinanceMemory()` · L381
- `function FinanceAdminMemory()` · L578
- `interface FinanceGap` · L724
- `function asRecord()` · L734
- `function asStringList()` · L738
- `function asText()` · L742
- `function financeToolGaps()` · L748
- `function collectFinanceGaps()` · L797
- `function FinanceGaps()` · L816
- `function Workspace()` · L864

<!-- 写法建议：在某个符号下另起一行缩进写它的职责/入参/副作用/坑 -->

