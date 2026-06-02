# `backend/app/features/finance_data/providers/ftshare.py`

> 485 行 · 14 个符号　|　[源码导读对应模块](../source-guide/index.md)

**整体作用**：

<!-- 一句话写这个文件干嘛的 -->

## 符号清单（对照手写）

- `BASE_URL` · L12
- `HEADERS` · L13
- `FTSHARE_DATASETS` · L15
- `def _ft_symbol()` · L200
- `def _cn_symbol()` · L216
- `def _dataset_catalog()` · L231
- `def _clean_params()` · L244
- `def _infer_dataset()` · L257
- `def _normalize_dataset_params()` · L269
- `def _build_dataset_request()` · L289
- `class FTShareProvider(FinanceProvider)` · L311
    - `def quote()` · L312
    - `def search()` · L387
    - `def _dataset_search()` · L436

<!-- 写法建议：在某个符号下另起一行缩进写它的职责/入参/副作用/坑 -->

