# Fix lint and formatting

Auto-fix ESLint violations and Prettier formatting issues across the repo.

## Do this

1. Use the repository root as the working directory.
2. Run both fixers in one command:

```bash
yarn fix
```

This runs two steps in sequence:

```
yarn fix:other   # Prettier --write on *.{css,scss,json,md,html,yml}
yarn fix:code    # ESLint --fix on *.{js,ts,tsx}
```

After running, verify nothing unexpected changed:

```bash
git diff
```

## Run individual fixers

```bash
yarn fix:other   # Prettier only (CSS, SCSS, JSON, Markdown, HTML, YAML)
yarn fix:code    # ESLint only (JS, TS, TSX)
```

## What cannot be auto-fixed

Some ESLint rules require manual intervention (e.g., unused variables, forbidden imports, logic errors). After `yarn fix:code`, re-run the check to see remaining issues:

```bash
yarn test:code
```

Resolve remaining errors at the source. Do NOT suppress them with `// eslint-disable` comments unless there is a documented reason.

## Pre-commit hook

`husky` + `lint-staged` automatically runs Prettier and ESLint on **staged files** before every commit. If a commit is rejected by the hook, fix the reported issues and re-stage — do NOT bypass hooks with `--no-verify`.

## Orientation

- Prettier config: `package.json` → `"prettier": "@excalidraw/prettier-config"`
- ESLint config: `.eslintrc.json` at repo root
- Lint-staged config: `.lintstagedrc.js` at repo root
- Ignored paths for lint: `.eslintignore`

See `.cursor/rules/testing.mdc` for the zero-warnings ESLint policy.
