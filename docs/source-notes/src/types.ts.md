# `src/types.ts`

> 677 行 · 60 个符号　|　[源码导读对应模块](../source-guide/index.md)

**整体作用**：

<!-- 一句话写这个文件干嘛的 -->

## 符号清单（对照手写）

- `type WireNextOption` · L3
- `type Role` · L5
- `interface ThinkChip` · L7
- `interface ToolMeta` · L12
- `interface ToolCall` · L19
- `type ToolProgress` · L30
- `interface Attach` · L58
- `type MessageStatus` · L64
- `interface MidInsert` · L66
- `interface Segment` · L74
- `type MessageKind` · L79
- `interface Message` · L88
- `type BgStatus` · L132
- `interface BgTask` · L134
- `type BgEventKind` · L154
- `type BgUrgency` · L156
- `interface BgEvent` · L158
- `interface Skill` · L167
- `type FsStatus` · L184
- `interface FsNode` · L186
- `type MemoryScope` · L199
- `type MemorySource` · L200
- `type MemoryStatus` · L201
- `interface EntityRef` · L203
- `interface Memory` · L209
- `interface Workspace` · L225
- `interface Session` · L232
- `interface QueuedInput` · L248
- `type Chunk` · L259
- `interface ChatInput` · L285
- `interface FinanceProvider` · L297
- `interface FinanceMemorySource` · L309
- `interface FinanceMemorySourceInput` · L320
- `interface FinanceWorkflow` · L328
- `interface FinanceStatus` · L339
- `interface FinanceProviderUsageRecord` · L350
- `interface FinanceProviderUsageSummary` · L367
- `interface FinanceMemoryRecord` · L379
- `interface FinanceMemoryBulkDeleteResult` · L398
- `interface FinanceMemoryPromoteResult` · L404
- `interface FinanceMemoryIngestionJob` · L413
- `interface FinanceMemoryExtractionCandidate` · L426
- `interface FinanceMemoryExtractionBatch` · L443
- `interface FinanceMemoryGap` · L461
- `interface FinanceEntity` · L473
- `interface FinanceGraphEdge` · L482
- `interface FinanceGraphFact` · L496
- `interface FinanceGraphEvidence` · L506
- `interface FinanceGraphContext` · L518
- `interface FinancePortfolioPositionInput` · L530
- `interface FinancePortfolioInput` · L542
- `interface FinancePortfolioPosition` · L557
- `interface FinancePortfolioTransaction` · L574
- `interface FinancePortfolioSnapshot` · L589
- `interface FinancePortfolioBacktestPoint` · L602
- `interface FinancePortfolioBacktest` · L610
- `interface FinanceRebalanceLog` · L626
- `interface FinancePortfolioEvent` · L637
- `interface FinancePortfolioSummary` · L647
- `interface FinancePortfolioDetail` · L670

<!-- 写法建议：在某个符号下另起一行缩进写它的职责/入参/副作用/坑 -->

