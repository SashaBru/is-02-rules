# CLAUDE.md — project context for Claude and other AI assistants

This file mirrors the intent of **Cursor** project rules in **`.cursor/rules/*.mdc`** so behavior stays consistent across tools (Claude Code, Cursor, etc.). The **canonical extended guide** is **[`onboarding-docs/AGENTS.md`](./onboarding-docs/AGENTS.md)**; read it for full commands, layout, and workflows.

**Repository:** Excalidraw monorepo (`excalidraw-monorepo`), Yarn **1.x** workspaces, **Node ≥ 18**. Root commands: `yarn start`, `yarn test:all`, `yarn fix`, `yarn build:packages`.

---

## Parallel to Cursor rules

| Topic | Cursor rule file | Policy summary |
|-------|-----------------|----------------|
| **Architecture** | `architecture.mdc` | Canvas 2D for scene rendering — no `react-konva`, `fabric.js`, or `pixi.js`. Editor state via `ActionManager.executeAction(...)` in `packages/excalidraw/actions/manager.tsx` — there is no `actionManager.dispatch()`. No Redux / Zustand / MobX for core editor state. Jotai runs in two isolated layers: `packages/excalidraw/editor-jotai.ts` (editor store) and `excalidraw-app/app-jotai.ts` (app-shell store) — do not cross the boundary. |
| **Protected files** | `protected-files.mdc` | Do **not** modify without explicit approval + `yarn test:all` + manual QA: `packages/excalidraw/scene/Renderer.ts`, `packages/excalidraw/data/restore.ts`, `packages/excalidraw/actions/manager.tsx`, `packages/excalidraw/types.ts`. |
| **Element model** | `element-model.mdc` | Never mutate `ExcalidrawElement` objects directly — always use `scene.mutateElement(element, updates)`. Access elements via `scene.getNonDeletedElementsMap()`. Do not set `startBinding` / `endBinding` directly; use the binding API. |
| **Package boundaries** | `monorepo-packages.mdc` | Strict DAG: `common` → `math` → `element` → `excalidraw` → `excalidraw-app`. No circular imports. No upward imports from published packages into `excalidraw-app`. |
| **Testing** | `testing.mdc` | Vitest (`vitest.config.mts`). Run `yarn test:all` (typecheck → ESLint → Prettier → Vitest) before every PR. ESLint `--max-warnings=0` — zero warnings allowed. Never commit snapshot updates without reviewing the diff. |
| **Collaboration / Firebase** | `collaboration-firebase.mdc` | Collaboration logic stays in `excalidraw-app/` only. All server-bound data is end-to-end encrypted client-side. Firebase security rule changes must be tested against the emulator before deploying. Local storage is authoritative; Firebase writes are best-effort for collaboration. |

---

## What to run before proposing changes

```bash
yarn test:all         # typecheck → ESLint → Prettier → Vitest once
```

If touching packages and the app shows resolution issues:

```bash
yarn build:packages   # rebuild common → math → element → excalidraw
yarn start            # then verify in the browser
```

Auto-fix lint and format issues:

```bash
yarn fix              # Prettier write + ESLint --fix
```

---

## Cursor commands (equivalent yarn commands)

| Cursor command | Equivalent |
|----------------|-----------|
| `/run-tests` | `yarn test:all` (see `.cursor/commands/run-tests.md`) |
| `/fix-lint-format` | `yarn fix` (see `.cursor/commands/fix-lint-format.md`) |

---

## Rule validation

When a rule makes a specific claim about the codebase (a method name, a file, an API), verify it with `rg` before following it. Documented checks are in **[`dev-docs/ab-validation-rules.md`](./dev-docs/ab-validation-rules.md)**.

Example already validated: `ActionManager` exposes `executeAction`, not `dispatch`. Jotai is used inside `packages/excalidraw` (39 files via `editor-jotai.ts`), not only in the app shell.

---

## Dependencies

No new npm packages without explicit maintainer / team approval — aligns with `.cursor/rules/architecture.mdc` and `onboarding-docs/AGENTS.md`.
