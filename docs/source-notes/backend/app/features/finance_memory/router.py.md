# `backend/app/features/finance_memory/router.py`

> 542 行 · 31 个符号　|　[源码导读对应模块](../source-guide/index.md)

**整体作用**：

<!-- 一句话写这个文件干嘛的 -->

## 符号清单（对照手写）

- `router` · L64
- `_FNAME_BAD` · L66
- `def _safe_file_name()` · L69
- `def _authorize_workspace()` · L75
- `def _raise_bad_request()` · L83
- `def get_finance_memory_records()` · L92
- `def post_finance_memory_record()` · L116
- `def post_finance_memory_records_bulk_delete()` · L130
- `def _ensure_shared_source_async()` · L142
- `def post_admin_finance_memory_record()` · L148
- `def post_admin_finance_memory_records_import()` · L163
- `def post_admin_finance_memory_records_promote()` · L183
- `def get_finance_memory_record()` · L202
- `def patch_finance_memory_record()` · L214
- `def delete_finance_memory_record()` · L231
- `def post_finance_memory_ingestion()` · L247
- `def post_finance_memory_upload()` · L269
- `def get_finance_memory_ingestions()` · L321
- `def get_finance_memory_ingestion()` · L332
- `def retry_finance_memory_ingestion()` · L344
- `def get_finance_memory_extractions()` · L386
- `def accept_finance_memory_extraction()` · L398
- `def ignore_finance_memory_extraction()` · L414
- `def get_finance_memory_gaps()` · L426
- `def patch_finance_memory_gap()` · L439
- `def get_finance_entity_resolve()` · L456
- `def get_finance_graph_entities()` · L466
- `def post_finance_graph_entity()` · L476
- `def patch_finance_graph_entity()` · L488
- `def post_finance_graph_edge()` · L504
- `def get_finance_graph_context()` · L516

<!-- 写法建议：在某个符号下另起一行缩进写它的职责/入参/副作用/坑 -->

