# `backend/app/features/finance_memory/service.py`

> 1874 行 · 67 个符号　|　[源码导读对应模块](../source-guide/index.md)

**整体作用**：

<!-- 一句话写这个文件干嘛的 -->

## 符号清单（对照手写）

- `GRAPH_LINK_THRESHOLD` · L62
- `EXTRACTION_ENTITY_CREATE_THRESHOLD` · L63
- `VALID_RECORD_KINDS` · L64
- `class FinanceMemoryContext` · L75
- `def _clean_list()` · L82
- `def _record_view()` · L96
- `def _job_view()` · L117
- `def _candidate_view()` · L140
- `def _batch_view()` · L159
- `def _gap_view()` · L182
- `def _entity_view()` · L196
- `def _edge_view()` · L207
- `def _fact_view()` · L229
- `def _evidence_view()` · L241
- `def _visible_record_filter()` · L255
- `def _normalize_kind()` · L277
- `def _normalize_source()` · L284
- `def _memory_query_terms()` · L292
- `def _like_pattern()` · L311
- `def list_records()` · L316
- `def get_record()` · L363
- `def _ref_name()` · L377
- `def _ref_confidence()` · L381
- `def _merge_unique()` · L388
- `def _resolve_entity_for_graph()` · L402
- `def _relation_type_from_refs()` · L464
- `def _has_explicit_relation_endpoint()` · L472
- `def _relation_candidate_gaps()` · L482
- `def _guess_entity_type()` · L493
- `def _can_create_extraction_entity()` · L504
- `def _ensure_extraction_entities()` · L514
- `def _ref_relation_role()` · L565
- `def _relation_endpoints_from_refs()` · L569
- `def _refs_include_record()` · L590
- `def _soft_delete_record_graph()` · L599
- `def _materialize_record_graph()` · L623
- `def create_record()` · L712
- `def patch_record()` · L795
- `def delete_record()` · L859
- `def bulk_delete_records()` · L883
- `def promote_records_to_shared()` · L918
- `_TICKER_RE` · L1028
- `_NAME_RE` · L1029
- `def extract_mentions()` · L1032
- `def resolve_entities()` · L1045
- `def list_entities()` · L1101
- `def create_entity()` · L1119
- `def patch_entity()` · L1143
- `def create_edge()` · L1169
- `def log_gap()` · L1199
- `def graph_context()` · L1226
- `def list_gaps()` · L1349
- `def update_gap_status()` · L1372
- `def ingest_text()` · L1393
- `def list_ingestion_jobs()` · L1508
- `def get_ingestion_job()` · L1534
- `def get_ingestion_job_with_artifact()` · L1546
- `def create_failed_ingestion_job()` · L1560
- `def mark_ingestion_retried()` · L1604
- `def _normalize_candidate()` · L1615
- `def stage_extraction_batch()` · L1649
- `def list_extraction_batches()` · L1712
- `def get_extraction_batch()` · L1743
- `def accept_extraction_batch()` · L1759
- `def ignore_extraction_batch()` · L1831
- `def context_from_user()` · L1855
- `def selected_sources_or_default()` · L1869

<!-- 写法建议：在某个符号下另起一行缩进写它的职责/入参/副作用/坑 -->

