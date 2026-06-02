# Frontend Style System Decoupling Spec

## Goals

- Keep the current visual behavior stable while splitting the global stylesheet by ownership.
- Make future feature styling land in a predictable layer instead of growing `src/styles.css`.
- Preserve one CSS entrypoint so Vite, app imports, and deployment behavior stay unchanged.

## Current Structure

`src/styles.css` is now an import manifest only:

```text
src/styles.css
src/styles/
  tokens.css
  base.css
  layout.css
  components.css              # shared component manifest
  components/
    empty.css
    nav-controls.css
    picker.css
    dialog.css
  features/
    loader.css
    sidebar.css               # sidebar manifest
    sidebar/
      shell.css
      account-panels.css
      auth.css
      invites.css
      navigation.css
      insights.css
    chat.css                  # chat manifest
    chat/
      shell.css
      messages.css
      status-chip.css
      tool-stream.css
      turn-controls.css
      composer.css
      selection.css
    workspace.css             # workspace manifest
    workspace/
      shell.css
      markdown.css
      tree-actions.css
      expand-editor.css
      compact-summary.css
      math.css
      media.css
    background-tasks.css      # background task manifest
    background-tasks/
      badge.css
      header-strip.css
      drawer-detail.css
      inline-handoff.css
  overlays.css                # overlay manifest
  overlays/
    drawer.css
    insight-detail.css
  responsive.css
```

The import order in `src/styles.css` and each feature manifest is part of the contract. Later modules may rely on earlier tokens and base rules, and responsive/background task overrides remain near the end to preserve cascade behavior.

## Layer Rules

- `tokens.css`: CSS variables only. Add semantic aliases here before repeating raw colors or shadows across modules.
- `base.css`: reset and document-level primitives only.
- `layout.css`: app shell grid and top-level layout behavior.
- `components/`: shared primitives such as buttons, pickers, fields, empty states, and truncation helpers. Add here only when at least two features share the class contract.
- `features/<domain>/`: feature-owned selectors and local states. Keep selectors near the component workflow they style.
- `overlays/`: drawer, modal, detail bubble, and portal surfaces.
- `responsive.css`: breakpoint-only overrides. Do not add base styling here.
- Manifest files (`components.css`, `features/*.css`, `overlays.css`) should contain `@import` statements only.

## New Feature Workflow

1. Reuse tokens from `tokens.css`.
2. Put shared control styling in `components/` only if at least two features use it.
3. Put feature-specific selectors in the owning `features/<domain>/` stylesheet.
4. Put mobile or compact overrides in `responsive.css` using existing class names.
5. Keep `src/styles.css` and local manifests as import-only.

## Next Cleanup Candidates

- Rename raw color variables into semantic roles (`--accent`, `--success`, `--danger`, `--surface-*`) without changing rendered colors.
- Move compact-summary styles from `features/workspace/compact-summary.css` to chat ownership after the compact-history component is next touched. It currently remains in the original cascade position to avoid visual drift.
- Audit raw colors against `tokens.css` in a visual QA pass, replacing repeated literals with semantic aliases gradually.

## Acceptance Checks

- `npm run lint`
- `npm run build`
- `git diff --check`
