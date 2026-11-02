# Migration: Vitest 4 to 5, Jest, Mocha

## Contents

- Before upgrading to Vitest 5
- Vitest 5 behavior changes
- Vitest 5 removals and replacements
- Artifacts moved to .vitest
- Browser mode changes
- From Jest
- From Mocha, Chai and Sinon
- Upgrade hygiene

## Before upgrading to Vitest 5

- Check Vite >= 6.4.0 and Node.js >= 22.12.0 first. Older versions give unexpected errors. https://vitest.dev/guide/migration
- `vite` is now a peer dependency. Yarn does not install peers: run `yarn add -D vite`. https://vitest.dev/guide/migration
- Fix deprecation warnings from 4.x first: they are the 5.0 removals. https://vitest.dev/releases

## Vitest 5 behavior changes

| Change | Fix |
| :-- | :-- |
| `clearMocks` defaults to `true`: history recorded in setup files, top level or `beforeAll` is gone in tests | Assert inside the test, or `clearMocks: false` |
| `-t` matches the full name joined by `' > '` | `-t 'math > adds'`, `-t adds` or `-t 'math.*adds'` |
| Inline projects inherit the root (`extends: true`), arrays concatenated | `extends: false` on the project |
| A referenced config with `projects` yields nested projects | Merge a shared config, not the root one |
| `vi.mock`, `vi.unmock`, `vi.hoisted` throw outside the module top level | Move them up, or use `vi.doMock` |
| Unawaited `resolves`, `rejects`, `toMatchFileSnapshot` fail the test | `await expect(p).resolves.toBe(1)` |
| `expect.poll` rejects when it does not settle within `timeout` | Raise `timeout` |
| `toThrow('')` matches any message | `toThrow(/^$/)` |
| Fake timers also mock `Temporal` | `vi.useFakeTimers({ toNotFake: ['Temporal'] })` |
| Class mocks keep prototype methods and `instanceof` | Override on the mock's `prototype` to shadow |
| Titles formatted by `pretty-format`, `$` strings unquoted | Update snapshots of titles |
| `VITEST_POOL_ID` / `VITEST_WORKER_ID` are 1-based | Fix per-worker database names and indexes |
| Configs not looked up in parent directories | `vitest --config ../vitest.config.ts` |
| Vitest UI needs a token | Open the URL printed by `vitest --ui` |

https://vitest.dev/guide/migration

- Inline projects that leave the Vite config untouched share one Vite server, so plugin `config` hooks no longer run per project. Set `sharedViteServer: false` if a plugin needs per-project instances. https://vitest.dev/guide/migration

## Vitest 5 removals and replacements

| Old pattern | Current pattern |
| :-- | :-- |
| `test.sequential`, `describe.sequential`, `sequential` option | `{ concurrent: false }` |
| `import { bench } from 'vitest'`, `bench.skip/only/todo` | `bench` fixture in `test()`, `test.skip/only/todo` |
| `benchmark.compare`, `--compare` | `writeResult` plus `bench.from()` in `bench.compare()` |
| `benchmark.outputJson`, `--outputJson` | `--reporter=json --outputFile=<path>` |
| `vitest/coverage`, `vitest/reporters` | `vitest/node` |
| `vitest/environments`, `vitest/snapshot` | `vitest/runtime` |
| `vitest/runners`, `vitest/suite` | `TestRunner` from `vitest` |
| `vitest/mocker` | `@vitest/mocker` |
| `@vitest/expect` imports | `expect`, `chai` from `vitest` |
| `Assertion<string>` | `Assertion<void, string>` |
| Global `jest.Matchers` augmentation | Augment `Matchers<R, T>` in `vitest` |
| `resolveConfig(...)` as Vitest config | `(await resolveConfig(options)).test` |

https://vitest.dev/guide/migration

- Also deprecated: `test.scoped` (use `test.override`), `toMatchTypeOf` (use `toExtend`), `workspace` (use `test.projects`). https://vitest.dev/api/test , https://vitest.dev/api/expect-typeof , https://vitest.dev/guide/projects

## Artifacts moved to .vitest

- One `.vitest/` root: `attachments/`, `attachments/failure-screenshots/`, `blob/`, `index.html`, `json/output.json`, `junit/output.xml`. https://vitest.dev/guide/migration
- `vitest --reporter=json | jq` breaks: JSON goes to a file. Read the file or use `reporters: [['json', { stdout: true }]]`. https://vitest.dev/guide/migration

## Browser mode changes

- `browser.api` is ignored: move it to `test.api`. `browser.isolate` is deprecated. https://vitest.dev/guide/migration
- Locators are exact by default. `browser.locators.exact: false` restores the old match. https://vitest.dev/guide/migration
- `toHaveTextContent` no longer accepts a RegExp: use `toMatchTextContent`. https://vitest.dev/guide/migration
- `render` from `vitest-browser-vue` and `vitest-browser-svelte` is async. https://vitest.dev/guide/migration
- Custom commands receive `{ selector }: SerializedLocator`. https://vitest.dev/guide/migration
- `toMatchScreenshot` has its own `browser.expect.toMatchScreenshot.screenshotDirectory`. https://vitest.dev/guide/migration
- Coverage: set `perFile` on each glob threshold, and review `include` patterns now matched relative to the root. https://vitest.dev/guide/migration

## From Jest

| Jest habit | Vitest |
| :-- | :-- |
| `jest.fn`, `jest.mock` | `vi.fn`, `vi.mock` |
| Globals available | Import from `vitest`, or `globals: true` |
| `mockReset` gives an empty function | Resets to the `vi.fn(impl)` implementation |
| `jest.mock('./p', () => 'hello')` | `vi.mock('./p', () => ({ default: 'hello' }))` |
| `__mocks__` applied automatically | Call `vi.mock` (in `setupFiles` for every file) |
| `jest.requireActual` | `await vi.importActual(...)` |
| `jest.replaceProperty` | `vi.stubEnv` or `vi.spyOn` |
| `jest.setTimeout(5_000)` | `vi.setConfig({ testTimeout: 5_000 })` |
| `done` callback | async test or `new Promise(done => ...)` |
| Hooks run in list order | Stack order, or `sequence: { hooks: 'list' }` |
| `JEST_WORKER_ID` | `VITEST_POOL_ID` |
| `jest-snapshot` in matchers | `import { Snapshots } from 'vitest'` |
| `test.each` | `test.for` |

https://vitest.dev/guide/migration/jest , https://vitest.dev/guide/learn/writing-tests

- With globals off, Testing Library does not auto-clean the DOM. https://vitest.dev/guide/migration/jest
- Mocks do not reach external libraries importing the module: add them to `server.deps.inline`. https://vitest.dev/guide/migration/jest
- `mock.mock` survives `mockClear` as the same object. https://vitest.dev/guide/migration/jest
- Legacy fake timers are not supported. https://vitest.dev/guide/migration/jest
- Vue CLI snapshots: add `snapshotSerializers: ['jest-serializer-vue']`. https://vitest.dev/guide/migration/jest

## From Mocha, Chai and Sinon

- `before` / `after` become `beforeAll` / `afterAll`. Chai and sinon-chai style assertions work without plugins. https://vitest.dev/guide/migration/mocha
- `sinon.stub(obj, 'm')` becomes `vi.spyOn(obj, 'm')`, `.returns(42)` becomes `mockReturnValue(42)`, `callsFake` becomes `mockImplementation`, `clock.tick` becomes `vi.advanceTimersByTime`. `sinon.mock` has no equivalent. https://vitest.dev/guide/migration/mocha
- Vitest runs files in parallel: shared state or ports between files may break. https://vitest.dev/guide/migration/mocha

## Upgrade hygiene

- TypeScript definitions may change between minors. Pin the minor with a semver range and upgrade by hand. https://vitest.dev/releases
- Pin exact versions when relying on pre-releases or experimental features. https://vitest.dev/releases
