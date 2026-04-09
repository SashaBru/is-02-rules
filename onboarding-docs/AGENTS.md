# AGENTS.md — guidance for automated assistants and contributors

This repository is the **Excalidraw** monorepo (`name`: `excalidraw-monorepo` in root `package.json`): a hand-drawn style whiteboard with a publishable editor package and a first-party web app.

**Cross-tool context:** **[`CLAUDE.md`](../CLAUDE.md)** summarizes the same policies as **`.cursor/rules/`** for Claude Code and other assistants that do not read Cursor rule files.

---

## 1. Overview

| Item | Detail |
|------|--------|
| **Monorepo tool** | Yarn **1.x** workspaces (`packageManager`: `yarn@1.22.22`) |
| **Workspaces** | `excalidraw-app`, `packages/*`, `examples/*` |
| **Node** | `>= 18` (engines in root `package.json`) |
| **Primary languages** | TypeScript, TSX |
| **App shell** | Vite-based host in `excalidraw-app/` |
| **Core editor package** | `packages/excalidraw` (published as `@excalidraw/excalidraw`) |
| **Tests** | Vitest (`vitest.config.mts` at repo root) |

---

## 2. Repository structure

| Path | Role |
|------|------|
| **`excalidraw-app/`** | Deployable web app (Vite, PWA, collaboration, Firebase). Run via `yarn start` from root. |
| **`packages/common/`** | Shared constants and utilities consumed by all other packages. |
| **`packages/math/`** | 2D math primitives (`@excalidraw/math`) — framework-agnostic, no React/DOM. |
| **`packages/element/`** | Element model, geometry, bindings, hit-testing (`@excalidraw/element`). |
| **`packages/excalidraw/`** | Main editor: React UI, canvas/scene rendering, actions, data, locales, tests. |
| **`packages/utils/`** | Additional shared helpers. |
| **`examples/`** | Sample integrations (`with-script-in-browser`, `with-nextjs`). |
| **`scripts/`** | Release automation, locales coverage tooling. |
| **`firebase-project/`** | Firebase security rules (`firestore.rules`, `storage.rules`). |
| **`vitest.config.mts`** | Vitest config with `@excalidraw/*` path aliases. |
| **`tsconfig.json`** | Root TypeScript project; `yarn test:typecheck` runs `tsc`. |
| **`.cursor/rules/*.mdc`** | Cursor project rules (architecture, testing, elements, packages, etc.). |
| **`.cursor/commands/*.md`** | Cursor slash commands (`/run-tests`, `/fix-lint-format`). |
| **`dev-docs/ab-validation-rules.md`** | A/B validation log — rule text checked against `rg` / source inspection. |

---

## 3. Prerequisites and setup

1. Install **Node.js ≥ 18** and **Yarn 1 (Classic)**.
2. From the **repository root**, install all workspace dependencies:

   ```bash
   yarn install
   ```

3. Husky pre-commit hooks are wired via `prepare` — `yarn install` enables them automatically.

---

## 4. Development workflow

| Goal | Command (from repo root) |
|------|--------------------------|
| **Dev server (main app)** | `yarn start` → delegates to `yarn --cwd ./excalidraw-app start` |
| **Rebuild workspace packages** | `yarn build:packages` → `common` → `math` → `element` → `excalidraw` (ESM) |
| **Example (script in browser)** | `yarn start:example` — builds packages then starts the example |
| **Production-like local** | `yarn start:production` |

If you change a low-level package and the dev server shows resolution or build issues, run `yarn build:packages` before `yarn start`.

---

## 5. Build and release

| Goal | Command |
|------|---------|
| **Full app build** | `yarn build` |
| **Individual packages** | `yarn build:common`, `yarn build:math`, `yarn build:element`, `yarn build:excalidraw` |
| **Clean build outputs** | `yarn rm:build` |
| **Clean install** | `yarn clean-install` (removes all `node_modules`, then reinstalls) |
| **Release** | `yarn release:latest` / `yarn release:next` / `yarn release:test` |

---

## 6. Testing, linting, and formatting

| Goal | Command |
|------|---------|
| **Full CI gate** | `yarn test:all` — `typecheck` → `ESLint` → `Prettier` → `Vitest once` |
| **Typecheck only** | `yarn test:typecheck` (`tsc`) |
| **ESLint** | `yarn test:code` (`--max-warnings=0`) |
| **Prettier check** | `yarn test:other` |
| **Vitest (watch)** | `yarn test` |
| **Vitest once** | `yarn test:app --watch=false` |
| **Snapshot update** | `yarn test:update` |
| **Coverage** | `yarn test:coverage` |
| **Auto-fix** | `yarn fix` — Prettier write + ESLint fix |

Run **`yarn test:all`** before opening a PR. See `.cursor/commands/run-tests.md` for a step-by-step breakdown.

---

## 7. Architecture expectations

These match the constraints in `.cursor/rules/architecture.mdc`:

- **State (ActionManager):** Editor state is driven by `ActionManager` (`packages/excalidraw/actions/manager.tsx`). Run actions via **`executeAction`** — there is no `actionManager.dispatch()`. Do NOT introduce Redux, Zustand, or MobX for core editor state.
- **Rendering:** All scene content is drawn with **Canvas 2D** — not React DOM. Do not add `react-konva`, `fabric.js`, or `pixi.js` to the canvas pipeline.
- **Jotai (two layers):** Component UI state uses Jotai in two isolated layers: `packages/excalidraw/editor-jotai.ts` (editor store) and `excalidraw-app/app-jotai.ts` (app-shell store). Both use `jotai-scope` isolation — do NOT import atoms across the boundary. Always import from the correct layer's file, never directly from `'jotai'`.
- **Types:** `AppState` and core element types live in `packages/excalidraw/types.ts`.
- **Heavy deps:** Locales, Mermaid, and image processing must be dynamically imported — never statically bundled.

---

## 8. Element model conventions

These match `.cursor/rules/element-model.mdc`:

- All shapes extend `ExcalidrawElement`. Never mutate element objects directly — use `scene.mutateElement(element, updates)`.
- Access elements via `scene.getNonDeletedElementsMap()` or `scene.getNonDeletedElements()`.
- Arrow-to-element binding is managed by `packages/element/src/binding.ts` — do not set `startBinding` / `endBinding` directly.
- Generate element IDs with `nanoid()` from `@excalidraw/common`.

---

## 9. Monorepo package boundaries

These match `.cursor/rules/monorepo-packages.mdc`:

Packages form a strict dependency DAG — lower packages must NOT import from higher ones:

```
@excalidraw/common        ← no internal deps
       ↓
@excalidraw/math          ← may import common
       ↓
@excalidraw/element       ← may import common, math
       ↓
@excalidraw/excalidraw    ← may import common, math, element
       ↓
excalidraw-app            ← may import all (not published)
```

Do not import `excalidraw-app` code from any published package. Circular dependencies are forbidden.

---

## 10. Collaboration and Firebase

These match `.cursor/rules/collaboration-firebase.mdc`:

- Collaboration code lives exclusively in `excalidraw-app/` — never in published packages.
- Real-time sync uses Socket.io; conflict resolution uses `reconcileElements()` from `packages/excalidraw/data/reconcile.ts`.
- All data sent to the server is **end-to-end encrypted** client-side — never log or expose decrypted element data.
- Firebase security rules in `firebase-project/` are security-critical; test changes against the Firebase emulator before deploying.

---

## 11. Protected and high-risk files

Do **not** modify these without explicit approval, a full `yarn test:all` pass, and manual QA (see `.cursor/rules/protected-files.mdc`):

| File | Risk |
|------|------|
| `packages/excalidraw/scene/Renderer.ts` | Canvas render pipeline |
| `packages/excalidraw/data/restore.ts` | File format compatibility and migration |
| `packages/excalidraw/actions/manager.tsx` | Action dispatch system |
| `packages/excalidraw/types.ts` | Core type definitions |

---

## 12. Security and dependencies

- Do not commit secrets, API keys, or Firebase private config.
- Treat imported `.excalidraw` JSON, clipboard payloads, and collaboration data as **untrusted** — validate through established restore paths in `packages/excalidraw/data/`.
- No new npm packages without explicit approval.

---

## 13. Cursor rules and commands reference

| File | Scope | Purpose |
|------|-------|---------|
| `.cursor/rules/architecture.mdc` | `packages/**` | ActionManager, Canvas 2D, Jotai layers, forbidden deps |
| `.cursor/rules/protected-files.mdc` | always | Files requiring explicit approval before modification |
| `.cursor/rules/element-model.mdc` | `packages/excalidraw/**`, `packages/element/**` | Element mutation, binding, ID conventions |
| `.cursor/rules/monorepo-packages.mdc` | `packages/**` | Package DAG, build order, circular dep prohibition |
| `.cursor/rules/testing.mdc` | always | `yarn test:all` gate, snapshot discipline, ESLint zero-warnings |
| `.cursor/rules/collaboration-firebase.mdc` | `excalidraw-app/**`, `firebase-project/**` | E2E encryption, Firebase rules, local-first persistence |
| `.cursor/commands/run-tests.md` | — | Step-by-step guide for `yarn test:all` and individual test commands |
| `.cursor/commands/fix-lint-format.md` | — | Step-by-step guide for `yarn fix` and per-fixer commands |

**Rule vs. code checks:** See `dev-docs/ab-validation-rules.md` for documented A/B validations (e.g. Jotai scope, `ActionManager` API surface).

---

## 14. Troubleshooting

| Issue | Suggestion |
|-------|------------|
| Module resolution / stale package output | `yarn build:packages`, then retry `yarn start` |
| Lint or format failures | `yarn fix` then `yarn test:all` |
| Type errors | `yarn test:typecheck` for full `tsc` output |
| Outdated or broken `node_modules` | `yarn clean-install` |

---

## 15. Quick reference

```bash
yarn install          # install all workspace deps
yarn start            # dev server (excalidraw-app)
yarn build:packages   # rebuild @excalidraw/* packages (ESM)
yarn test:all         # full CI gate — run before every PR
yarn fix              # auto-fix Prettier + ESLint
```
