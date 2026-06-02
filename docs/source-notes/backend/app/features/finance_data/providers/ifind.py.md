# `backend/app/features/finance_data/providers/ifind.py`

> 617 行 · 24 个符号　|　[源码导读对应模块](../source-guide/index.md)

**整体作用**：

<!-- 一句话写这个文件干嘛的 -->

## 符号清单（对照手写）

- `REALTIME_FIELDS` · L17
- `ASHARE_HQ_FIELDS` · L18
- `BOARD_CODE_OVERRIDE` · L19
- `def _normalize_symbol()` · L33
- `def _setup_sdk_path()` · L44
- `def _import_ifind()` · L54
- `def _value_at()` · L82
- `def _extract_rq()` · L89
- `def _extract_history()` · L120
- `def _pick()` · L155
- `def _extract_constituents()` · L162
- `def _remote_ifind_url()` · L215
- `def _remote_data_rows()` · L219
- `def _remote_row_value()` · L228
- `def _remote_history_quotes()` · L235
- `def _remote_realtime_quote()` · L267
- `def _remote_board_items()` · L291
- `def _board_kind_id()` · L308
- `class IFindProvider(FinanceProvider)` · L324
    - `def _remote_enabled()` · L325
    - `def health()` · L328
    - `def _call_remote_ifind()` · L355
    - `def quote()` · L376
    - `def search()` · L515

<!-- 写法建议：在某个符号下另起一行缩进写它的职责/入参/副作用/坑 -->

