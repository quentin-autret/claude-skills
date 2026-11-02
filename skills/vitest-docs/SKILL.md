---
name: vitest-docs
description: Teaches current Vitest 5 practices drawn from the official documentation. Use when writing, reviewing or fixing Vitest tests (test, describe, expect, test.for, fixtures with test.extend, hooks, snapshots), mocking with vi (vi.fn, vi.spyOn, vi.mock, vi.when, fake timers, stubGlobal, stubEnv), configuring vitest.config or projects, choosing an environment or pool, filtering and running tests in CI, measuring coverage with v8 or istanbul, debugging slow or flaky suites, testing components in browser mode with Playwright, or migrating from Jest, Mocha or Vitest 4. Triggers on vitest, vi.mock, vi.fn, vitest.config, toMatchSnapshot, expect.poll, coverage thresholds, browser mode, @vitest/browser-playwright, jsdom, happy-dom, migrate Jest to Vitest, upgrade Vitest 5.
metadata:
  version: "Vitest 5.x (npm latest 5.1.0)"
  source: https://vitest.dev/guide/
  fetched: "2026-11-02"
---

# Vitest 5

This skill covers Vitest 5.x as documented on https://vitest.dev/guide/. It holds what differs from habit: surprising defaults, pitfalls, deprecated APIs and the patterns the docs recommend. Each claim in the reference files ends with the page it comes from.

## How to navigate

1. Read the quick rules below. They catch the most frequent mistakes.
2. Open the one reference file that matches the task, from the table at the end.
3. When code uses Jest APIs or Vitest 4 behavior, read `references/migration.md` first.

## Quick start

```ts
// vitest.config.ts
import { defineConfig } from 'vitest/config'
export default defineConfig({ test: { restoreMocks: true } })
```

```ts
// math.test.ts
import { expect, test } from 'vitest'

test.for([[1, 1, 2], [1, 2, 3]])('add(%i, %i) -> %i', ([a, b, expected]) => {
  expect(a + b).toBe(expected)
})
```

- Requires Vite >= 6.4.0 and Node.js >= 22.12.0. `vite` is a peer dependency. https://vitest.dev/guide/migration
- Run `vitest run` from scripts, CI and agents. Bare `vitest` starts watch mode in a terminal. https://vitest.dev/guide/learn/writing-tests-with-ai
- Add `.vitest/` to `.gitignore`. https://vitest.dev/guide

## Quick rules

### Writing tests

- `globals` is off: import `test`, `expect`, `vi` from `vitest`. https://vitest.dev/config/globals
- Parameterize with `test.for`, not `test.each`. https://vitest.dev/guide/learn/writing-tests
- Share setup through fixtures, `test.extend('name', fn)`, and always destructure them. https://vitest.dev/guide/test-context
- Return a cleanup function from `beforeEach` rather than writing a matching `afterEach`. https://vitest.dev/api/hooks
- No `done` callback. Use async tests. https://vitest.dev/api/test
- Tests do not type-check during a run. Run `tsc` or `vitest --typecheck.enabled`. https://vitest.dev/guide/learn/writing-tests , https://vitest.dev/guide/cli
- In concurrent tests use `expect` from the context: `async ({ expect }) => {}`. https://vitest.dev/api/test

### Assertions

- `await` every `.resolves`, `.rejects`, `expect.poll` and `toMatchFileSnapshot`, otherwise the test fails. https://vitest.dev/guide/migration
- Poll with `expect.poll` or `vi.waitFor`, never a `setTimeout` sleep. https://vitest.dev/guide/recipes/wait-for
- Narrow types with `expect.assert(x)`. `toBeDefined()` does not narrow. https://vitest.dev/guide/recipes/type-narrowing
- Snapshots are not written in CI. Update with `vitest -u` after reading the diff. https://vitest.dev/guide/snapshot

### Mocking

- Prefer `vi.mock(import('./x.js'))` over a string: the path follows file moves and the factory is type-checked. https://vitest.dev/api/vi
- Keep `vi.mock` and `vi.hoisted` at module top level. Elsewhere they throw. https://vitest.dev/guide/migration
- A factory returns every export, `default` included. https://vitest.dev/guide/migration/jest
- `clearMocks` is on by default. Add `restoreMocks: true` to restore spies after each test. https://vitest.dev/config/clearmocks , https://vitest.dev/guide/learn/mock-functions
- Argument-specific returns: `vi.when(fn).calledWith(1).thenReturn(x)`. https://vitest.dev/guide/recipes/conditional-mocking
- Mock classes with `vi.fn(class { ... })`. https://vitest.dev/guide/mocking/classes
- A function called inside its own file cannot be mocked. https://vitest.dev/guide/mocking/modules

### Config and runs

- A `vitest.config.*` ignores `vite.config.*` unless you `mergeConfig` it. https://vitest.dev/config
- Setting `include`, `exclude` or `reporters` replaces the defaults. Spread `configDefaults`. https://vitest.dev/config/include
- Default environment `node`. Install `jsdom` or `happy-dom` yourself. https://vitest.dev/guide/environment
- Default pool `forks`. Keep it for native modules and `process.chdir`. https://vitest.dev/config/pool
- `.only` fails the run in CI (see `allowOnly`). https://vitest.dev/api/test
- `-t` matches the full name joined by `' > '`. https://vitest.dev/guide/filtering

## Old patterns

| Old | Current |
| :-- | :-- |
| `jest.fn`, `jest.mock`, `jest.requireActual` | `vi.fn`, `vi.mock`, `await vi.importActual` |
| `test.each` rows spread into arguments | `test.for` with a destructured row and the context |
| `test.sequential`, `describe.sequential` | `{ concurrent: false }` |
| `import { bench } from 'vitest'` | `bench` fixture inside `test()` |
| `test.scoped` | `test.override` |
| `workspace` | `test.projects` |
| `toMatchTypeOf` | `toExtend` |
| `vi.mock` inside a test or `describe` | Top-level `vi.mock`, or `vi.doMock` |
| Unawaited `.resolves` auto-awaited | `await expect(p).resolves` |
| `toHaveTextContent(/regex/)` | `toMatchTextContent(/regex/)` |
| `vitest --reporter=json` on stdout | `.vitest/json/output.json`, or `stdout: true` |
| `vitest/environments`, `vitest/coverage` | `vitest/runtime`, `vitest/node` |

Details and the full lists: `references/migration.md`.

## Reference files

| File | Read it for |
| :-- | :-- |
| [references/writing-tests.md](references/writing-tests.md) | Test options, `test.for`, concurrency, fixtures, hooks, setup files, global setup, tags, benchmarks, in-source tests |
| [references/assertions-and-snapshots.md](references/assertions-and-snapshots.md) | `expect` pitfalls, async assertions, polling, narrowing, schema matching, custom matchers, snapshots, type tests |
| [references/mocking.md](references/mocking.md) | Mock cleanup, `vi.fn`, `vi.spyOn`, `vi.mock`, class mocks, `vi.when`, fake timers, globals, env, MSW, memfs |
| [references/config-and-cli.md](references/config-and-cli.md) | Config resolution, `include`, `globals`, environments, pools, projects, CLI filters, watch, reporters, sharding |
| [references/coverage.md](references/coverage.md) | v8 and istanbul, `coverage.include`, reports, thresholds, ignore hints |
| [references/debugging-and-performance.md](references/debugging-and-performance.md) | Debugger setup, common errors, dependency inlining, CommonJS, profiling, speeding up, disabling isolation |
| [references/browser-mode.md](references/browser-mode.md) | Browser mode setup, instances, Playwright provider, locators, `expect.element`, `userEvent`, component tests, visual and ARIA snapshots |
| [references/migration.md](references/migration.md) | Vitest 4 to 5 changes and removals, Jest and Mocha mappings, upgrade hygiene |

Crawled pages: `SOURCES.md`.
