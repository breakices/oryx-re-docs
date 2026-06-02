# `backend/app/features/finance_memory/jobs.py`

> 392 行 · 21 个符号　|　[源码导读对应模块](../source-guide/index.md)

**整体作用**：

<!-- 一句话写这个文件干嘛的 -->

## 符号清单（对照手写）

- `MAX_INGEST_BYTES` · L19
- `MAX_CHUNK_CHARS` · L20
- `MIN_CHUNK_CHARS` · L21
- `class FinanceParseError(ValueError)` · L24
    - `def __init__()` · L25
- `def _decode_text()` · L30
- `def _extract_docx()` · L34
- `def _extract_pdf()` · L56
- `def _extract_pdf_pages()` · L65
- `def _extract_csv()` · L90
- `def _infer_chunk_kind()` · L111
- `def _compact_title()` · L124
- `def _split_oversized()` · L132
- `def _chunk_markdown()` · L170
- `def _chunk_plain()` · L201
- `def _chunk_pdf_text()` · L235
- `def _chunk_pdf_pages()` · L250
- `def build_ingestion_chunks()` · L275
- `def extract_supported_text()` · L293
- `def run_text_ingestion()` · L321
- `def run_file_ingestion()` · L348

<!-- 写法建议：在某个符号下另起一行缩进写它的职责/入参/副作用/坑 -->

