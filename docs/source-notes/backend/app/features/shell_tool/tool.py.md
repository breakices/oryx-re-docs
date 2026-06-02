# `backend/app/features/shell_tool/tool.py`

> 635 行 · 24 个符号　|　[源码导读对应模块](../source-guide/index.md)

**整体作用**：

<!-- 一句话写这个文件干嘛的 -->

## 符号清单（对照手写）

- `_SYSTEM_FIRST_SEGMENTS` · L36
- `_PATH_TOKEN` · L43
- `_LONE_SLASH` · L47
- `def _split_pipeline()` · L50
- `def _first_disallowed()` · L87
- `_ESCAPE_RE` · L107
- `_HOST_PATH_ALLOWED` · L114
- `def _host_path_token()` · L119
- `_FIND_BAD_FLAGS` · L147
- `_META_IN_UNQUOTED` · L158
- `def _shell_metaprogramming()` · L161
- `def _has_unquoted_dollar()` · L178
- `def _maybe_reject_cat()` · L191
- `def _maybe_handle_ls()` · L211
- `def _bad_find_flag()` · L253
- `def _escape_path()` · L274
- `def _split_quoted()` · L286
- `def _normalize_paths()` · L315
    - `def repl()` · L323
- `def _reject()` · L344
- `def _hide_real_root()` · L355
- `def run_shell()` · L364
- `class ShellTool(Tool)` · L601
    - `def dispatch()` · L630

<!-- 写法建议：在某个符号下另起一行缩进写它的职责/入参/副作用/坑 -->

