# Frontend Decoupling and Fast Feature Delivery Spec

## Goals

- Keep the current UI behavior stable while reducing large-file coupling.
- Make new features land as vertical slices with clear ownership boundaries.
- Preserve local development, single-node deployment, and multi-pod backend compatibility.
- Avoid rewriting generated API schema files or reshaping visual design in this round.

## Current Hotspots

- `src/store.ts` mixes app state, workspace state, chat state, background task state, memories, pagination, and all actions.
- `src/api/client.ts` mixes auth transport, typed request wrappers, feature endpoints, stream helpers, and mock-mode branching.
- `src/hooks/useChat.ts` owns composer orchestration, stream consumption, turn state, reload behavior, and store mutation.
- Large components such as `Sidebar`, `Workspace`, `Composer`, and `ChatPane` combine data wiring with view rendering.
- `src/styles.css` is a global style surface; future changes should avoid adding more unrelated feature styles to the same area.

## Target Shape

The frontend should move toward stable public facades with feature-local modules behind them:

```text
src/
  api/
    client.ts                 # compatibility facade
    transport.ts              # auth, request, error, SSE primitives
    features/
      chat.ts
      workspace.ts
      memories.ts
      background.ts
  store.ts                    # compatibility facade
  store/
    types.ts
    utils.ts
    slices/
      chat.ts
      workspace.ts
      sessions.ts
      background.ts
      memories.ts
    initialState.ts
  hooks/
    useAppShellData.ts
    useAuthStage.ts
    useImpersonationStatus.ts
    chat/
      messages.ts
      options.ts
      reload.ts
      streamBuffer.ts
      streamEvents.ts
  features/
    chat/
      hooks/
      components/
      model/
    workspace/
    sidebar/
  styles/
    tokens.css
    layout.css
    features/
```

During migration, existing imports from `src/store.ts` and `src/api/client.ts` remain valid. New code should prefer the feature modules once they exist.

## Boundary Rules

- Components render UI and dispatch intent; they should not encode API request details.
- Hooks coordinate workflows and own async UI behavior; they should call API modules and store actions.
- Store slices own state transitions only; they should avoid calling network APIs directly.
- API feature modules own endpoint paths, payload mapping, stream parsing, and typed transport calls.
- Generated files such as `src/api/schema.gen.ts` are read-only integration inputs.
- Global CSS accepts tokens and shared layout primitives; feature-specific styles should move under feature style sections or files.

## Fast Feature Workflow

Every new feature should ship as a small vertical slice:

1. Add or reuse typed API methods in `src/api/features/<feature>.ts`.
2. Add minimal store state/actions in `src/store/slices/<feature>.ts`.
3. Add a hook in `src/features/<feature>/hooks` to compose API and state.
4. Add focused components in `src/features/<feature>/components`.
5. Add route or shell wiring through existing facades.
6. Add tests around the store/action or hook behavior when the feature has branching logic.

This keeps development agile: each feature can be reviewed by API, state, workflow, and UI layers independently, while still landing as one user-visible increment.

## Migration Plan

### Phase 1: Non-behavioral Frontend Split

- Split store types and pure utilities out of `src/store.ts`.
- Keep `useStore` exported from `src/store.ts`.
- Add this spec as the frontend refactor contract.
- Status: done.

### Phase 2: Store Slices

- Move workspace tree actions into a workspace slice.
- Move sessions and workspaces list actions into a sessions slice.
- Move chat messages, queue, turn, and pagination actions into a chat slice.
- Move background tasks/events and memories into separate slices.
- Preserve the facade export until all call sites are migrated or intentionally kept stable.
- Status: done. `src/store.ts` is now a compatibility facade over `initialState` and slice action builders.

### Phase 3: API Modules

- Extract transport primitives from `src/api/client.ts`.
- Move chat, workspace, memory, background task, auth, and admin endpoints into feature API modules.
- Keep `api` facade compatibility until callers are migrated.
- Status: done. `src/api/client.ts` is now a compatibility facade over transport and feature API modules.

### Phase 4: Workflow Hooks and Components

- Split `useChat` into smaller workflow hooks: send/stream, reload/history, queue, and turn lifecycle.
- Move feature-specific component logic out of shell components.
- Keep visual changes separate from structural refactors unless a component cannot be safely split otherwise.
- Status: done for this final-decoupling pass. Stream buffering, stream event application, message factories, option bypass, and reload logic are split under `src/hooks/chat`.
- Shell/auth status: done for the current layer. `AppShell` delegates data bootstrapping to `useAppShellData`; `AuthGate` delegates auth state to `useAuthStage`; `ImpersonationBanner` delegates identity polling and exit flow to `useImpersonationStatus`.
- Component hygiene status: done for this pass. Common async handlers now use `ignoreAsync` / `toVoidHandler`, caught errors use `errorMessage`, and time-driven render refreshes use `useNow`.

### Phase 5: Release Discipline

- Use env-gated or config-gated feature flags for partially deployed frontend features.
- Do not rely on pod-local state from the frontend; always treat backend ownership redirects/proxies as transparent transport behavior.
- Keep local and single-node behavior as the default development path.

## Acceptance Checks

- `npm run build`
- `npm run lint`
- `git diff --check`
- Smoke test: load the app, switch sessions, send a chat request, open workspace tree, and confirm background task updates still render.

## Code Review Findings

### Fixed in this iteration

- API client coupling: `src/api/client.ts` was a mixed transport, endpoint, SSE, mock, and auth module. It is now a facade over `src/api/transport.ts`, `src/api/types.ts`, and `src/api/features/*`.
- Store coupling: `src/store.ts` mixed all state and actions. It is now a facade over `src/store/initialState.ts` and `src/store/slices/*`.
- Chat workflow coupling: `src/hooks/useChat.ts` mixed message construction, stream buffering, stream event mutation, visible-window reload, and send/rewind orchestration. The reusable pieces now live under `src/hooks/chat/*`.
- Shell coupling: `src/components/AppShell.tsx` mixed layout with bootstrapping, session loading, workspace tree loading, memory loading, and spawn recovery. Data orchestration now lives in `src/hooks/useAppShellData.ts`.
- Auth coupling: `AuthGate` and `ImpersonationBanner` no longer own API/token workflow details directly; those flows live in dedicated hooks.
- Transport typing: API JSON reads now pass through typed helpers instead of leaking `Promise<any>` from the transport layer.
- Lint target hygiene: generated app bundles under `dist-app` are ignored by ESLint.
- Markdown rendering coupling: `MdComponents` now exports renderer wiring while block/span renderers live in `MdRenderers`.
- Component async/error policy: common UI actions in chat, sidebar, account panel, workspace, feedback, login, background task controls, code copy, and selection saving now use shared async/error helpers.
- Time-driven UI refresh: elapsed-time displays use `useNow` instead of reading `Date.now()` during render.

### Remaining component-level risks after this pass

- `src/components/Sidebar.tsx`, `src/components/Workspace.tsx`, and `src/components/Composer.tsx` are still large view/controller files. They are lint-clean now, but should be split by user workflow only when we are ready to review DOM and visual behavior together.
- `src/styles.css` remains a large global stylesheet. Future UI work should split shared tokens/layout first, then move feature styles with component ownership.
- The main production bundle remains above Vite's default 500 kB chunk warning. This is not a build failure, but future feature work should consider route/feature-level dynamic imports.
- Full frontend lint is clean after this pass.

## New Feature Landing Rules

- New API endpoints go into `src/api/features/<domain>.ts`; export through `src/api/client.ts` only for compatibility.
- New global state goes into a focused `src/store/slices/<domain>.ts`; avoid adding action bodies to `src/store.ts`.
- New chat stream events go through `src/hooks/chat/streamEvents.ts`; content buffering remains in `streamBuffer.ts`.
- New optimistic chat rows should use factories in `src/hooks/chat/messages.ts`.
- New component async handlers should wrap promises and surface/report errors deliberately; do not add raw async callbacks to JSX props.
- New time-driven UI should subscribe through `useNow` or a feature hook instead of reading wall-clock time directly during render.
