# `backend/app/db/models.py`

> 766 行 · 38 个符号　|　[源码导读对应模块](../source-guide/index.md)

**整体作用**：

<!-- 一句话写这个文件干嘛的 -->

## 符号清单（对照手写）

- `def utcnow()` · L23
- `class Base(DeclarativeBase)` · L30
- `class User(Base)` · L34
- `class InviteCode(Base)` · L57
- `class InviteUse(Base)` · L72
- `class AuthSession(Base)` · L83
- `class UserLlmSettings(Base)` · L95
- `class Workspace(Base)` · L111
- `class Session(Base)` · L123
- `class Turn(Base)` · L142
- `class Message(Base)` · L156
- `class ChipOption(Base)` · L183
- `class TurnPendingInput(Base)` · L201
- `class Memory(Base)` · L227
- `class BgTask(Base)` · L264
- `class BgEvent(Base)` · L297
- `class WebUsage(Base)` · L328
- `class FinanceProviderUsage(Base)` · L353
- `class FinanceMemoryRecord(Base)` · L380
- `class FinanceMemorySource(Base)` · L407
- `class FinanceMemoryEvent(Base)` · L423
- `class FinanceMemoryArtifact(Base)` · L435
- `class FinanceMemoryIngestionJob(Base)` · L450
- `class FinanceMemoryExtractionBatch(Base)` · L467
- `class FinanceMemoryExtractionCandidate(Base)` · L490
- `class FinanceEntity(Base)` · L513
- `class FinanceEntityEdge(Base)` · L529
- `class FinanceEntityFact(Base)` · L548
- `class FinanceEvidenceChunk(Base)` · L564
- `class FinanceMemoryGap(Base)` · L583
- `class FinancePortfolio(Base)` · L601
- `class FinancePortfolioPosition(Base)` · L627
- `class FinancePortfolioTransaction(Base)` · L648
- `class FinancePortfolioSnapshot(Base)` · L669
- `class FinanceRebalanceLog(Base)` · L688
- `class FinancePortfolioEvent(Base)` · L707
- `class Feedback(Base)` · L724
- `class BgProcess(Base)` · L739

<!-- 写法建议：在某个符号下另起一行缩进写它的职责/入参/副作用/坑 -->

