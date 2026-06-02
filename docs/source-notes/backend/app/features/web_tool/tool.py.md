# `backend/app/features/web_tool/tool.py`

> 669 行 · 23 个符号　|　[源码导读对应模块](../source-guide/index.md)

**整体作用**：

<!-- 一句话写这个文件干嘛的 -->

## 符号清单（对照手写）

- `_FILENAME_FALLBACK` · L36
- `_SAFE_NAME_RE` · L37
- `def _sandbox_path()` · L40
- `def _disabled_payload()` · L46
- `def _has_search_provider()` · L50
- `def _ensure_search_enabled()` · L54
- `def _ensure_extract_enabled()` · L62
- `def _normalize_search_engine()` · L77
- `def _ensure_requested_engine()` · L84
- `def _used_search_engine()` · L100
- `def _user_id_of_session()` · L108
- `def _filename_from_url()` · L113
- `def _unique_save_path()` · L131
- `def _resolve_save_path()` · L146
- `def web_search()` · L175
- `def web_fetch()` · L286
- `def web_cite()` · L474
- `class WebSearchTool(Tool)` · L549
    - `def dispatch()` · L588
- `class WebFetchTool(Tool)` · L601
    - `def dispatch()` · L628
- `class WebCiteTool(Tool)` · L636
    - `def dispatch()` · L662

<!-- 写法建议：在某个符号下另起一行缩进写它的职责/入参/副作用/坑 -->

