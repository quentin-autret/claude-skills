# Coverage

## Provider

- Default provider `v8`, recommended: no instrumentation, faster, less memory, accuracy matching Istanbul through AST remapping. Install `@vitest/coverage-v8`. https://vitest.dev/guide/coverage
- Escape hatch: `@vitest/coverage-istanbul` for non-V8 runtimes (Firefox, Bun, Cloudflare Workers) or to instrument only some files. https://vitest.dev/guide/coverage
- Run `vitest run --coverage` or set `coverage.enabled: true`. With dot options pass `--coverage.enabled`, never a bare `--coverage` next to them. https://vitest.dev/config/coverage

## What is measured

- Only files imported during the run appear by default. Set `coverage.include` to list untested files too. https://vitest.dev/guide/coverage

```ts
coverage: { include: ['src/**/*.{ts,tsx}'], exclude: ['**/utils/users.ts'] }
```
https://vitest.dev/config/coverage

- Patterns are relative to the project root. A pattern without a wildcard is a directory: `['src']` equals `['src/**']`. https://vitest.dev/config/coverage
- Test files matched by `include` are excluded from coverage automatically. https://vitest.dev/config/include
- Excluded files still showing up after source maps: `coverage.excludeAfterRemap: true`. https://vitest.dev/config/coverage
- `coverage.autoAttachSubprocess` (v8) covers child processes and worker threads, at a cost. https://vitest.dev/config/coverage

## Reports

- `coverage.reportsDirectory` (`./coverage`) is deleted before each run while `coverage.clean` is true (the default). https://vitest.dev/config/coverage
- No report is written when tests fail unless `coverage.reportOnFailure: true`. https://vitest.dev/config/coverage
- Default reporters `['text', 'html', 'clover', 'json']`. Options per reporter: `[['lcov', { projectRoot: './src' }], ['text']]`. `skipFull` hides fully covered files. https://vitest.dev/config/coverage
- Under an AI agent Vitest sets `skipFull` on `text` and adds `text-summary`. https://vitest.dev/guide/coverage

## Thresholds

- A positive number is a minimum percentage. A negative number is a maximum count of uncovered items. https://vitest.dev/config/coverage

```ts
thresholds: { functions: 90, lines: -10 }
```
https://vitest.dev/config/coverage

- `thresholds.100: true` sets every metric to 100. `thresholds.autoUpdate` rewrites the config when coverage rises. https://vitest.dev/config/coverage
- Glob thresholds inherit neither global thresholds nor `perFile`: set `perFile: true` on each glob. Their files still count in the global aggregate, unlike Jest. https://vitest.dev/config/coverage

```ts
thresholds: { functions: 95, 'src/utils/**.ts': { statements: 95, perFile: true }, '**/math.ts': { 100: true } }
```
https://vitest.dev/config/coverage

## Ignore hints

- esbuild strips comments from TypeScript, so hints need `@preserve`: `/* v8 ignore next -- @preserve */`. They may then reach production builds. https://vitest.dev/guide/coverage
- v8 hints: `ignore if`, `ignore else`, `ignore next`, `ignore start` / `ignore stop`, `ignore file`. https://vitest.dev/guide/coverage
- Generated code that cannot carry a hint: `coverage.ignoreNode(node, type)` (v8). https://vitest.dev/config/coverage

## Slow coverage

- Run `DEBUG=vitest:coverage vitest --run --coverage` to spot large or minified files included by accident, then narrow `include` / `exclude`. https://vitest.dev/guide/profiling-test-performance
