# `backend/app/features/finance_memory/schemas.py`

> 275 行 · 26 个符号　|　[源码导读对应模块](../source-guide/index.md)

**整体作用**：

<!-- 一句话写这个文件干嘛的 -->

## 符号清单（对照手写）

- `class FinanceMemoryRecordCreate(BaseModel)` · L9
- `class FinanceMemoryRecordPatch(BaseModel)` · L22
- `class FinanceMemoryBulkDeleteRequest(BaseModel)` · L33
- `class FinanceMemoryBulkDeleteResult(BaseModel)` · L37
- `class FinanceMemoryPromoteRequest(BaseModel)` · L43
- `class FinanceMemoryPromoteResult(BaseModel)` · L49
- `class FinanceMemoryRecordView(BaseModel)` · L58
- `class FinanceMemoryIngestionCreate(BaseModel)` · L77
- `class FinanceMemoryIngestionJobView(BaseModel)` · L87
- `class FinanceMemoryExtractionCandidateInput(BaseModel)` · L100
- `class FinanceMemoryExtractionBatchCreate(BaseModel)` · L112
- `class FinanceMemoryExtractionCandidateView(BaseModel)` · L124
- `class FinanceMemoryExtractionBatchView(BaseModel)` · L141
- `class FinanceEntityCandidate(BaseModel)` · L159
- `class FinanceEntityResolveResult(BaseModel)` · L168
- `class FinanceEntityView(BaseModel)` · L173
- `class FinanceEntityCreate(BaseModel)` · L182
- `class FinanceEntityPatch(BaseModel)` · L191
- `class FinanceGraphEdgeView(BaseModel)` · L200
- `class FinanceGraphEdgeCreate(BaseModel)` · L214
- `class FinanceGraphFactView(BaseModel)` · L223
- `class FinanceGraphEvidenceView(BaseModel)` · L233
- `class FinanceGraphContextView(BaseModel)` · L245
- `class FinanceMemoryGapView(BaseModel)` · L257
- `class FinanceMemoryGapPatch(BaseModel)` · L269
- `class OkOut(BaseModel)` · L273

<!-- 写法建议：在某个符号下另起一行缩进写它的职责/入参/副作用/坑 -->

