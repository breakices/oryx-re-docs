# `backend/app/features/workspace/tool.py`

> 835 行 · 30 个符号　|　[源码导读对应模块](../source-guide/index.md)

**整体作用**：

<!-- 一句话写这个文件干嘛的 -->

## 符号清单（对照手写）

- `def _hash_bytes()` · L15
- `MAX_READ_BYTES` · L25
- `DEFAULT_LINE_LIMIT` · L26
- `DEFAULT_BYTE_LIMIT` · L31
- `DEFAULT_PDF_PAGES` · L32
- `IMAGE_SUFFIXES` · L37
- `def _fmt_bytes()` · L40
- `def _line_count()` · L50
- `def _root_for_session()` · L56
- `def _aroot_for_session()` · L64
- `def _with_line_numbers()` · L71
- `def _parse_pages()` · L80
- `def _extract_docx()` · L108
- `def _list_zip()` · L141
- `def _extract_pdf()` · L190
- `def _resolve_text_slice()` · L225
- `def read_file()` · L269
            - `def _build_fn()` · L333
                - `def _f()` · L334
- `def edit_file()` · L486
- `def write_file()` · L560
- `def list_dir()` · L643
- `_READ_FILE_DESC` · L679
- `class ReadFileTool(Tool)` · L704
    - `def dispatch()` · L732
- `class WriteFileTool(Tool)` · L743
    - `def dispatch()` · L769
- `class EditFileTool(Tool)` · L777
    - `def dispatch()` · L804
- `def fs_node_payload()` · L812

<!-- 写法建议：在某个符号下另起一行缩进写它的职责/入参/副作用/坑 -->

