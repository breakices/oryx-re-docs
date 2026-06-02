# `backend/app/features/admin/router.py`

> 578 行 · 39 个符号　|　[源码导读对应模块](../source-guide/index.md)

**整体作用**：

<!-- 一句话写这个文件干嘛的 -->

## 符号清单（对照手写）

- `router` · L29
- `def _workspace_view()` · L32
- `def _session_view()` · L41
- `class WorkspaceCreate(BaseModel)` · L51
- `class WorkspacePatch(BaseModel)` · L56
- `def _normalize_domain_mode()` · L60
- `def list_workspaces()` · L68
- `def create_workspace()` · L74
- `def patch_workspace()` · L92
- `class SessionCreate(BaseModel)` · L110
- `def _assert_owns_session()` · L116
- `def _assert_owns_workspace()` · L125
- `def list_sessions()` · L135
- `def create_session()` · L157
- `def list_session_messages()` · L185
- `def _wire_message_meta()` · L232
- `def _wire_tool_calls()` · L254
- `def _message_view()` · L284
- `def reset_session()` · L323
- `class SessionPatch(BaseModel)` · L329
- `def patch_session()` · L334
- `def archive_session()` · L349
- `def unarchive_session()` · L361
- `class ForkIn(BaseModel)` · L371
- `class ApiKeyPatch(BaseModel)` · L378
- `class LlmSettingsPatch(BaseModel)` · L382
- `def _normalize_llm_base_url()` · L388
- `def _normalize_llm_model()` · L398
- `def _llm_settings_view()` · L407
- `def get_me()` · L422
- `class PreferencesPatch(BaseModel)` · L434
- `def set_preferences()` · L440
- `class FeedbackIn(BaseModel)` · L476
- `def submit_feedback()` · L484
- `def list_feedback_admin()` · L499
- `def set_api_key()` · L512
- `def get_llm_settings()` · L527
- `def set_llm_settings()` · L532
- `def fork_session()` · L561

<!-- 写法建议：在某个符号下另起一行缩进写它的职责/入参/副作用/坑 -->

