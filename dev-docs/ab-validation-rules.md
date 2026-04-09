# A/B validation — Cursor rules vs. codebase

This document records **A/B validation** for project rules: a **stated rule (A)** is checked against **what the repository actually implements (B)** using grep and source inspection.

---

## Rule under test

**File:** `.cursor/rules/architecture.mdc`  
**Section:** State Management (as validated on 2026-04-09)

**Original claim (variant A):**  
"App-shell UI (toolbar, panels) uses **Jotai atoms** — acceptable only outside the core editor package."

**Competing hypothesis (variant B):**  
Jotai is a first-class dependency of `@excalidraw/excalidraw` itself, with a dedicated isolated store (`editor-jotai.ts`) used across the core package's components, hooks, and actions — not confined to `excalidraw-app`.

---

## Test scenario

1. **Objective:** Determine whether Jotai is confined to `excalidraw-app` (app shell) or whether it is also used inside `packages/excalidraw` (the published core package).
2. **Commands run** (from repository root):

   ```bash
   # Is jotai a declared dependency of the core package?
   rg 'jotai' packages/excalidraw/package.json

   # Does the core package import jotai directly?
   rg "from ['\"]jotai['\"]" packages/excalidraw --glob '*.{ts,tsx}' -l

   # Does an editor-specific jotai wrapper exist?
   cat packages/excalidraw/editor-jotai.ts

   # How many core-package files re-import from that wrapper?
   rg "from ['\"].*editor-jotai['\"]" packages/excalidraw --glob '*.{ts,tsx}' -l | wc -l

   # Is jotai used in excalidraw-app as well?
   rg "from ['\"]jotai['\"]" excalidraw-app --glob '*.{ts,tsx}' -l
   ```

3. **Source check:** Open `packages/excalidraw/editor-jotai.ts` and inspect its purpose.

---

## Results

### Variant A — Rule text (Jotai app-shell only)

| Check | Result |
|-------|--------|
| `jotai` in `packages/excalidraw/package.json` dependencies | **Present** — `"jotai": "2.11.0"` and `"jotai-scope": "0.7.2"` listed as runtime deps |
| Direct `from 'jotai'` import inside `packages/excalidraw` | **Present** — `editor-jotai.ts` imports `atom`, `createStore`, `PrimitiveAtom`, `WritableAtom` |
| Files in `packages/excalidraw` importing `editor-jotai` | **39 files** — components, hooks, actions, and utilities across the core package |

**Interpretation:** The rule claim ("acceptable only outside the core editor package") is **false**. Jotai is a declared dependency of the published core package and is used in 39 source files within it.

### Variant B — Codebase (Jotai inside core package)

| Check | Result |
|-------|--------|
| `packages/excalidraw/editor-jotai.ts` | **Exists** — creates an isolated Jotai store via `jotai-scope` (`createIsolation()`), exports `EditorJotaiProvider`, `editorJotaiStore`, and scoped hooks |
| Consumer files (sample) | `App.tsx`, `LayerUI.tsx`, `Dialog.tsx`, `LibraryMenu.tsx`, `CommandPalette.tsx`, `ColorPicker/*.tsx`, `Sidebar.tsx`, `data/library.ts`, `actions/actionToggleShapeSwitch.tsx`, … |
| Jotai in `excalidraw-app` | **Also present** — `excalidraw-app/app-jotai.ts` (separate app-level store, same isolation pattern) |

**Interpretation:** Jotai is used in **two layers**:
- `packages/excalidraw/editor-jotai.ts` — isolated editor store for UI component state within the core package.
- `excalidraw-app/app-jotai.ts` — separate app-shell store for the deployed web app.

The `jotai-scope` isolation (`createIsolation()`) ensures the two stores do not bleed into each other, but both exist and are legitimate.

---

## Conclusion

- **Variant A (app-shell only)** is **invalid**: Jotai is a published dependency of `@excalidraw/excalidraw` and used in 39 files in the core package.
- **Variant B (core + app-shell, isolated)** is **valid**: Jotai is intentional at both levels, with isolation provided by `jotai-scope`.

**Action taken:** `.cursor/rules/architecture.mdc` was corrected — the Jotai note now states that it is used across both the core package (via `editor-jotai.ts`) and the app shell (via `app-jotai.ts`), and that `jotai-scope` isolation must be preserved when adding new atoms.

**How to re-run this check:** After any dependency audit or Jotai upgrade, repeat the `rg` commands above. If atoms from the editor store leak into the app-shell store (or vice versa), `EditorJotaiProvider` / `jotai-scope` isolation may have been broken.

---

## Follow-up A/B ideas (not yet run)

| Rule | Possible A/B check |
|------|---------------------|
| `element-model.mdc` — "never mutate elements directly" | `rg 'element\.\(x\|y\|width\|height\) =' packages/excalidraw --glob '*.{ts,tsx}'` — direct property assignments should be absent outside tests |
| `monorepo-packages.mdc` — no upward imports | `rg 'from.*@excalidraw/excalidraw' packages/element` — should be empty (element must not import from excalidraw) |
| `testing.mdc` — ESLint max-warnings=0 | Check `.eslintrc.json` or `package.json` scripts for `--max-warnings` flag |
