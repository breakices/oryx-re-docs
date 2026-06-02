# `backend/app/features/finance_portfolio/service.py`

> 936 行 · 36 个符号　|　[源码导读对应模块](../source-guide/index.md)

**整体作用**：

<!-- 一句话写这个文件干嘛的 -->

## 符号清单（对照手写）

- `VALID_MODES` · L47
- `VALID_STATUSES` · L48
- `VALID_SIDES` · L49
- `class FinancePortfolioContext` · L53
- `def context_from_user()` · L61
- `def context_for_session()` · L77
- `def _clean_refs()` · L92
- `def _norm_symbol()` · L104
- `def _norm_mode()` · L111
- `def _norm_status()` · L118
- `def _norm_side()` · L125
- `def _market_value()` · L132
- `def _snapshot_payload_source()` · L137
- `def _snapshot_metrics()` · L143
- `def _position_view()` · L156
- `def _transaction_view()` · L177
- `def _snapshot_view()` · L194
- `def _rebalance_view()` · L209
- `def _event_view()` · L222
- `def _summary_view()` · L234
- `def _load_detail_rows()` · L272
- `def _portfolio_for_context()` · L311
- `def _append_event()` · L322
- `def _reweight_positions()` · L346
- `def _create_snapshot_from_positions()` · L358
- `def _position_from_input()` · L387
- `def create_portfolio()` · L408
- `def list_portfolios()` · L452
- `def get_portfolio()` · L475
- `def patch_portfolio()` · L492
- `def create_snapshot()` · L528
- `def propose_rebalance()` · L574
- `def confirm_rebalance()` · L612
- `def _history_points()` · L655
- `def backtest_portfolio()` · L674
- `def _apply_transaction()` · L870

<!-- 写法建议：在某个符号下另起一行缩进写它的职责/入参/副作用/坑 -->

