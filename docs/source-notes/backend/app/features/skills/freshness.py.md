# `backend/app/features/skills/freshness.py`

> 304 行 · 21 个符号　|　[源码导读对应模块](../source-guide/index.md)

**整体作用**：

<!-- 一句话写这个文件干嘛的 -->

## 符号清单（对照手写）

- `FORCE_LATEST` · L21
- `NOTIFY_ON_CHANGE` · L22
- `SKILL_FRESHNESS_NOTICE_KIND` · L23
- `class SkillVersion` · L27
- `class StaleSkillNotice` · L37
- `def current_skill_version()` · L42
- `def current_skill_versions()` · L61
- `def version_to_meta()` · L74
- `def skill_loads_meta()` · L85
- `def load_skill_tool_result_meta()` · L96
- `def _version_from_meta_payload()` · L115
- `def _legacy_version_from_tool_content()` · L131
- `def latest_loaded_skill_versions()` · L148
- `def is_current()` · L176
- `def _notice_key()` · L187
- `def emitted_skill_freshness_notice_keys()` · L195
- `def stale_loaded_skill_notices()` · L214
- `def skill_freshness_notice_content()` · L235
- `def persist_skill_freshness_notice_cycle()` · L250
- `def workspace_preset_override_message()` · L270
- `def persist_workspace_preset_override_cycle()` · L284

<!-- 写法建议：在某个符号下另起一行缩进写它的职责/入参/副作用/坑 -->

