# Run the test suite

Run all quality checks before merging or releasing.

## Do this

1. Use the repository root as the working directory.
2. Run the full gate in one command:

```bash
yarn test:all
```

This runs four steps in sequence:

```
yarn test:typecheck   # TypeScript strict-mode check across all packages
yarn test:code        # ESLint with --max-warnings=0 (zero warnings allowed)
yarn test:other       # Prettier formatting check
yarn test:app         # Vitest unit and component tests (non-watch)
```

All four steps must pass clean. A failure in any one of them blocks the merge.

## Run individual steps

When iterating on a specific failure, target only the step that's broken:

```bash
yarn test:typecheck      # TypeScript errors only
yarn test:code           # ESLint errors/warnings only
yarn test:other          # Prettier formatting only
yarn test                # Vitest in watch mode (interactive)
yarn test:coverage       # Vitest with coverage report
```

## Update snapshots

If a component or render output intentionally changed, update snapshots — then review the diff carefully:

```bash
yarn test:update
```

Never commit a snapshot update you haven't manually verified.

## Canvas tests

Tests run against a mocked Canvas 2D API (`vitest-canvas-mock`). Do not assert on actual pixel output — assert on element state and scene properties instead.

## Orientation

- Test files: `packages/excalidraw/__tests__/`, `excalidraw-app/__tests__/`
- Snapshots: `__snapshots__/` directories alongside test files
- Vitest config: `vitest.config.mts` at repo root
- ESLint config: `.eslintrc.json` at repo root

See `.cursor/rules/testing.mdc` for the full testing policy.
