# `backend/app/features/finance_portfolio/schemas.py`

> 214 行 · 18 个符号　|　[源码导读对应模块](../source-guide/index.md)

**整体作用**：

<!-- 一句话写这个文件干嘛的 -->

## 符号清单（对照手写）

- `class FinancePortfolioPositionInput(BaseModel)` · L9
- `class FinancePortfolioCreate(BaseModel)` · L21
- `class FinancePortfolioPatch(BaseModel)` · L36
- `class FinancePortfolioPositionView(BaseModel)` · L48
- `class FinancePortfolioTransactionInput(BaseModel)` · L65
- `class FinancePortfolioTransactionView(BaseModel)` · L76
- `class FinancePortfolioSnapshotCreate(BaseModel)` · L91
- `class FinancePortfolioBacktestRequest(BaseModel)` · L102
- `class FinancePortfolioBacktestPoint(BaseModel)` · L110
- `class FinancePortfolioBacktestView(BaseModel)` · L118
- `class FinancePortfolioSnapshotView(BaseModel)` · L134
- `class FinanceRebalanceProposalCreate(BaseModel)` · L147
- `class FinanceRebalanceConfirm(BaseModel)` · L156
- `class FinanceRebalanceLogView(BaseModel)` · L160
- `class FinancePortfolioEventView(BaseModel)` · L171
- `class FinancePortfolioSummaryView(BaseModel)` · L181
- `class FinancePortfolioDetailView(FinancePortfolioSummaryView)` · L204
- `class OkOut(BaseModel)` · L212

<!-- 写法建议：在某个符号下另起一行缩进写它的职责/入参/副作用/坑 -->

