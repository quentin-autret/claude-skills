# Writing tests: test API, fixtures, hooks, tags

## Contents

- Test signature and options
- Parameterized tests
- Concurrency
- Fixtures and the test context
- Hooks and lifecycle
- Setup files and global setup
- Tags
- Benchmarks
- In-source tests

## Test signature and options

- `test(name, body?, timeout?)` or `test(name, options, body?)`. A trailing timeout cannot be combined with an options object: put `timeout` inside it, `test('x', { skip: true, timeout: 10_000 }, fn)`. https://vitest.dev/api/test
- Options set to `undefined` are ignored and do not override chain, suite or tag values. https://vitest.dev/api/test
- The Jest `done` callback is not supported. Use an async function or return a promise. https://vitest.dev/api/test
- `test.only` fails the run in CI (`allowOnly` defaults to `!process.env.CI`). Filter by file or line instead of leaving `.only`. https://vitest.dev/config/allowonly https://vitest.dev/guide/filtering
- `test.skipIf(cond)` / `test.runIf(cond)` always set `skip`, so a falsy condition overrides a `skip` inherited from tags. https://vitest.dev/api/test
- Skip at runtime with the context: `({ skip }) => { skip(condition, 'note') }`. https://vitest.dev/guide/test-context
- `test.fails('repro #1234', ...)` pins a known bug: it passes while the code still fails. https://vitest.dev/api/test
- `retry` takes a number or `{ count, delay, condition }`. A function `condition` only works in test files, not in the config (config is serialized to workers). `repeats` reruns a test to debug flakiness. https://vitest.dev/api/test
- `meta` merges only top-level keys from suites and tags. Keep it flat. https://vitest.dev/api/test
- Suite options (`timeout`, `retry`, `tags`, `shuffle`, `concurrent`) apply to all nested tests: `describe('slow', { timeout: 10_000 }, () => {})`. https://vitest.dev/api/describe

## Parameterized tests

- Default: `test.for`. It does not spread array rows and passes the Test Context as second argument. `test.each` (Jest style) spreads rows and gives no context. Same split for `describe.for` / `describe.each`. https://vitest.dev/guide/learn/writing-tests

```ts
test.for([[1, 1, 2], [1, 2, 3]])('add(%i, %i) -> %i', ([a, b, expected]) => {
  expect(a + b).toBe(expected)
})
test.for([{ a: 1, b: 1, expected: 2 }])('add($a, $b) -> $expected', ({ a, b, expected }) => {})
```
https://vitest.dev/guide/learn/writing-tests

- Title tokens: `%s %d %i %f %j %o`, `%#` (0-based index), `%$` (1-based), `$prop` / `$a.val` for objects, `$0 $1` for arrays. Long values are truncated at `taskTitleValueFormatTruncate`. https://vitest.dev/api/test

## Concurrency

- Files run in parallel in separate workers. Tests inside a file run sequentially unless marked `test.concurrent`, `describe.concurrent` or `sequence.concurrent: true`. Opt out with `{ concurrent: false }`. https://vitest.dev/guide/parallelism
- Concurrent tests only help when they await I/O or timers. Synchronous ones still run one after another. Bound: `maxConcurrency` (default 5). https://vitest.dev/guide/parallelism https://vitest.dev/config/maxconcurrency
- In concurrent tests take `expect`, `onTestFinished` and `onTestFailed` from the context. The global ones cannot track which concurrent test is running, so snapshots and assertions are attributed wrongly. https://vitest.dev/api/test https://vitest.dev/api/hooks

```ts
test.concurrent.for([[1, 1], [1, 2]])('add(%i, %i)', ([a, b], { expect }) => {
  expect(a + b).toMatchSnapshot()
})
```
https://vitest.dev/api/test

## Fixtures and the test context

- Built-in context: `task`, `expect`, `skip`, `annotate`, `signal`, `bench`, `onTestFailed`, `onTestFinished`. https://vitest.dev/guide/test-context
- `signal` aborts on timeout, Ctrl+C or `--bail`. Pass it to `fetch`, streams, `child_process.spawn` and your own helpers. https://vitest.dev/guide/recipes/cancellable
- Default fixture API: the builder `test.extend('name', value | fn)`, typed by inference. The object syntax with `use()` still works but needs manual generics. https://vitest.dev/guide/test-context

```ts
export const test = baseTest
  .extend('config', { port: 3000, host: 'localhost' })
  .extend('server', async ({ config }) => `http://${config.host}:${config.port}`)
  .extend('tempFile', async ({}, { onCleanup }) => {
    const p = `/tmp/test-${Date.now()}.txt`
    await fs.writeFile(p, 'x')
    onCleanup(async () => { await fs.unlink(p) })
    return p
  })
```
https://vitest.dev/guide/test-context

- Fixtures are lazy: they run only when destructured. Always destructure (`({ db }) =>`). `(context) => context.db` defeats detection. https://vitest.dev/guide/test-context
- `onCleanup` can be called once per fixture (a second call throws). Split multi-resource fixtures. https://vitest.dev/guide/test-context
- Options: `{ auto: true }`, `{ scope: 'test' | 'file' | 'worker' }`, `{ injected: true }` (value from project `provide`). https://vitest.dev/guide/test-context
- A fixture without scope is test-scoped and cannot be used by `file` or `worker` fixtures. Those cannot read `task`, `expect` or `skip`. https://vitest.dev/guide/test-context
- `worker` scope behaves like `file` unless isolation is disabled. In the vm pools it always falls back to per file. https://vitest.dev/guide/test-context
- `test.override('name', value)` overrides a fixture for a suite and its children. It cannot add fixtures or change `scope` / `auto`. https://vitest.dev/guide/test-context
- Global `beforeAll` / `afterAll` / `aroundAll` do not see custom fixtures. Call them on the extended `test` (`test.beforeEach`, `test.aroundEach`). Suite-level ones read only file and worker fixtures. https://vitest.dev/guide/test-context

## Hooks and lifecycle

- `beforeEach` / `beforeAll` can return a cleanup function, which runs after all `afterEach` / `afterAll`. Prefer it to a separate after hook. https://vitest.dev/api/hooks

```ts
beforeEach(async () => {
  await database.connect()
  return async () => { await database.disconnect() }
})
```
https://vitest.dev/api/hooks

- A hook returning any other value is misread as a teardown. Use braces: `beforeEach(() => { setActivePinia(createTestingPinia()) })`. https://vitest.dev/guide/migration/jest
- Hook order is a stack (after hooks reversed, `sequence.hooks: 'stack'`). Default hook timeout 10 s. https://vitest.dev/api/hooks
- `aroundEach(async (runTest, ctx) => ...)` / `aroundAll` wrap a test in a context (transaction, AsyncLocalStorage, span). You must call `runTest()`, else the test fails. Use them only for such contexts. https://vitest.dev/api/hooks

```ts
export const test = baseTest.extend('db', { scope: 'file' }, async ({}, { onCleanup }) => {
  const db = await createTestDatabase(); onCleanup(() => db.close()); return db })
test.aroundEach(async (runTest, { db }) => { await db.transaction(runTest) })
```
https://vitest.dev/guide/recipes/db-transaction

- `onTestFinished(fn)` / `onTestFailed(fn)` register from inside a test and run after `afterEach`. Good for helpers that own their cleanup (`getTestDb()`). https://vitest.dev/api/hooks

## Setup files and global setup

- `setupFiles` run in the worker before each test file, before collection. Put `expect.extend` and polyfills there. Keep them light and remember editing one reruns everything. https://vitest.dev/guide/lifecycle https://vitest.dev/guide/learn/setup-teardown
- With `isolate: false` the setup file still re-executes per file: guard heavy work with a global flag, but register hooks every time. https://vitest.dev/config/setupfiles
- `globalSetup` runs once in the main process, only if a test is queued. Its variables are invisible to tests: pass data with `project.provide(key, value)` and read with `inject(key)`. https://vitest.dev/config/globalsetup

```ts
export default function setup(project: TestProject) { project.provide('wsPort', 3000) }
declare module 'vitest' { export interface ProvidedContext { wsPort: number } }
```
https://vitest.dev/config/globalsetup

- In watch mode global setup does not rerun. Reset state with `project.onTestsRerun(...)` (do not destructure `project`). https://vitest.dev/config/globalsetup

## Tags

- Tags must be declared in `test.tags` (`{ name, timeout, retry, priority }`). An undeclared tag throws unless `strictTags: false`. https://vitest.dev/guide/test-tags
- Use tags for cross-file categories sharing options (`flaky`, `db`). Use projects when the pool or environment differs. https://vitest.dev/guide/test-tags
- The JSDoc `@module-tag x` applies to every test in the file. Use `{ tags: ['x'] }` per test. https://vitest.dev/guide/test-tags
- Filter: `vitest --tags-filter="(unit || e2e) && !slow"`. `--list-tags` lists them. Type them by augmenting `TestTags`. https://vitest.dev/guide/test-tags

## Benchmarks

- `bench` is a test-context fixture, not an import: `test('sort', async ({ bench }) => { await bench('sort', () => { [3, 1, 2].sort() }).run() })`. Compare with `await bench.compare(bench('a', fn), bench('b', fn))`. https://vitest.dev/guide/benchmarking
- The fixture exists only in files matching `benchmark.include` (`*.bench.*`, `*.benchmark.*`). Plain `vitest` skips them unless `benchmark.enabled: true`. `vitest bench` runs only them. https://vitest.dev/guide/benchmarking
- Assert on `bench.compare()` results: `expect(result.get('a')).toBeFasterThan(result.get('b'), { delta: 0.1 })`, with the `{ retry: 3 }` test option. Store baselines with `writeResult` and replay with `bench.from()`, generated in one environment. https://vitest.dev/guide/benchmarking
- Consume results (dead-code elimination). Copy hot imports to a local const: module-runner getters distort loops. https://vitest.dev/guide/benchmarking

## In-source tests

- Write `if (import.meta.vitest) { const { it, expect } = import.meta.vitest }`, set `includeSource`, and `define: { 'import.meta.vitest': 'undefined' }` for production builds. Types: `"types": ["vitest/importMeta"]`. https://vitest.dev/guide/in-source
