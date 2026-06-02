# `backend/app/kernel/sse_protocol.py`

> 241 行 · 29 个符号　|　[源码导读对应模块](../source-guide/index.md)

**整体作用**：

<!-- 一句话写这个文件干嘛的 -->

## 符号清单（对照手写）

- `class _Chunk(BaseModel)` · L21
- `class ThinkChunk(_Chunk)` · L27
- `class ToolChunk(_Chunk)` · L32
- `class ContentChunk(_Chunk)` · L44
- `class NextOption(_Chunk)` · L49
- `class NextChunk(_Chunk)` · L56
- `class FsNodePayload(_Chunk)` · L61
- `class FsChunk(_Chunk)` · L75
- `class MetaUsage(_Chunk)` · L80
- `class MetaChunk(_Chunk)` · L88
- `class QueuedItem(_Chunk)` · L96
- `class QueueChunk(_Chunk)` · L108
- `class SpawnChunk(_Chunk)` · L113
- `class MidUserChunk(_Chunk)` · L119
- `class SegmentBreakChunk(_Chunk)` · L128
- `class ParkedChunk(_Chunk)` · L132
- `class ResumedChunk(_Chunk)` · L137
- `class CompactStartChunk(_Chunk)` · L141
- `class CompactInlineDoneChunk(_Chunk)` · L148
- `class RestoreSeed(_Chunk)` · L153
- `class CompactDoneChunk(_Chunk)` · L158
- `class DoneChunk(_Chunk)` · L168
- `class BgTaskChunk(_Chunk)` · L186
- `class BgEventChunk(_Chunk)` · L203
- `class WakeChunk(_Chunk)` · L213
- `_chat_adapter` · L228
- `_session_adapter` · L229
- `def validate_chat_chunk()` · L232
- `def validate_session_chunk()` · L239

<!-- 写法建议：在某个符号下另起一行缩进写它的职责/入参/副作用/坑 -->

