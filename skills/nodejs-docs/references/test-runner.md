# node:test runner and mocking

## Contents

- Writing tests
- Skip, todo, only, expected failures
- Running and filtering
- Execution model and run()
- Coverage, reporters, snapshots
- Mocking
- Version-flagged and early-development features
- Old patterns

## Writing tests

- A test fails when it throws, rejects, or calls `done` with an error. Taking `done` and returning a promise also fails. https://nodejs.org/docs/latest/api/test.html
- Always `await t.test(...)`: a test does not wait for its subtests (suites do), and outstanding subtests are cancelled and failed. https://nodejs.org/docs/latest/api/test.html#subtests

```js
test('top level test', async (t) => {
  await t.test('subtest 1', (t) => { assert.strictEqual(1, 1); });
});
```

- Do not let async work outlive the test: a later `setImmediate` subtest fails, and later uncaught exceptions or unhandled rejections are marked failed and reported as top-level diagnostic warnings. https://nodejs.org/docs/latest/api/test.html#extraneous-asynchronous-activity
- `timeout` (default `Infinity`) cannot cancel a test that blocks the thread. `concurrency` defaults to `false`. https://nodejs.org/docs/latest/api/test.html#testname-options-fn
- `after` hooks run even when tests fail. https://nodejs.org/docs/latest/api/test.html#afterfn-options
- `t.plan(n)` counts only `t.assert.*` calls and subtests, not the imported `assert`. `{ wait: true }` waits for late assertions. https://nodejs.org/docs/latest/api/test.html#contextplancountoptions
- Poll instead of sleeping with `await t.waitFor(fn, { interval: 50, timeout: 1000 })` (added v23.7.0, backported to v22.14.0). https://nodejs.org/docs/latest/api/test.html#contextwaitforcondition-options

## Skip, todo, only, expected failures

- `t.skip()` does not stop the test: `return` right after. https://nodejs.org/docs/latest/api/test.html#skipping-tests

```js
test('skip() method', (t) => { t.skip('this is skipped'); return; });
```

- TODO tests run but never fail the exit code. https://nodejs.org/docs/latest/api/test.html#todo-tests
- `only` works only with `--test-only` or isolation off, and every ancestor must be marked `only`. https://nodejs.org/docs/latest/api/test.html#only-tests
- `expectFailure` (added v24.14.0) passes only if the test throws a matching error. `skip` beats `todo`, which beats `expectFailure`. https://nodejs.org/docs/latest/api/test.html#expecting-tests-to-fail

```js
it('x', { expectFailure: { label: 'reason', match: /error message/ } }, () => {});
```

## Running and filtering

- `node --test` finds `**/*.test.{cjs,mjs,js}`, `*-test.*`, `*_test.*`, `test-*.*`, `test.*` and `test/**`, plus TypeScript equivalents. Quote globs so the shell does not expand them. https://nodejs.org/docs/latest/api/test.html#running-tests-from-the-command-line

```bash
node --test "**/*.test.js" "**/*.spec.js"
```

- `--test-name-pattern` and `--test-skip-pattern` take regexes and must both match when combined. https://nodejs.org/docs/latest/api/test.html#filtering-tests-by-name
- `--test-rerun-failures <state-file>` reruns failures keyed by file:line:col, so keep test order deterministic. https://nodejs.org/docs/latest/api/test.html#rerunning-failed-tests

## Execution model and run()

- Each file runs in its own child process by default. Within a file tests share one thread. With `--test-isolation=none`, files load into the runner and global state leaks between them. https://nodejs.org/docs/latest/api/test.html#test-runner-execution-model
- Children inherit Node options except runner-managed ones (`--test`, coverage, reporters, `--watch`, `--config-file`, randomization). https://nodejs.org/docs/latest/api/test.html#child-process-option-inheritance
- `run({ env })` replaces `process.env` instead of merging. With a custom reporter pipeline, set `process.exitCode` on `test:fail` yourself: https://nodejs.org/docs/latest/api/test.html#runoptions

```js
run({ files: [path.resolve('./tests/test.js')] })
  .on('test:fail', () => { process.exitCode = 1; })
  .compose(tap)
  .pipe(process.stdout);
```

- Watch mode (`node --test --watch`) is experimental. https://nodejs.org/docs/latest/api/test.html#watch-mode

## Coverage, reporters, snapshots

- Coverage (`--experimental-test-coverage`) is experimental. It excludes `node_modules`, core and test files. Unmet `--test-coverage-*` thresholds set exit code 1. https://nodejs.org/docs/latest/api/test.html#collecting-code-coverage

```js
/* node:coverage ignore next 3 */
/* node:coverage disable */ /* ... */ /* node:coverage enable */
```

- The default reporter is `spec`, also on non-TTY output since v23. Reporter text may change between versions: consume `TestsStream` events instead of parsing. `lcov` only emits coverage, so pair it with another reporter. https://nodejs.org/docs/latest/api/test.html#test-reporters

```bash
node --test --experimental-test-coverage --test-reporter=lcov --test-reporter-destination=lcov.info
```

- Snapshots (`t.assert.snapshot()`) are stable. Generate with `--test-update-snapshots` and commit the `.snapshot` files. A missing file fails the first run. https://nodejs.org/docs/latest/api/test.html#snapshot-testing

## Mocking

- Use `t.mock` rather than the global `mock`: mocks are restored when the test ends. https://nodejs.org/docs/latest/api/test.html#mocking

```js
test('spies on a method', (t) => {
  t.mock.method(number, 'add');
  number.add(1, 2);
  assert.strictEqual(number.add.mock.callCount(), 1);
});
```

- Mock timers with `t.mock.timers.enable({ apis: ['setTimeout'] })`, then `tick(ms)`. Default `apis` include `Date`, sharing one clock. Destructured imports such as `import { setTimeout } from 'node:timers'` are not mocked. https://nodejs.org/docs/latest/api/test.html#timers , https://nodejs.org/docs/latest/api/test.html#class-mocktimers
- `mock.module()` is early development and needs `--experimental-test-module-mocks`. References taken before mocking keep the original, and `cache` defaults to `false`. Pass `exports` (with `exports.default`). https://nodejs.org/docs/latest/api/test.html#mockmodulespecifier-options
- `mock.property()` (added v24.3.0): `mockImplementationOnce` counts gets and sets alike. https://nodejs.org/docs/latest/api/test.html#caveat

## Version-flagged and early-development features

| Feature | Status | Source |
| :-- | :-- | :-- |
| `--test-global-setup` (`globalSetup`/`globalTeardown`) | early development, v24.0.0. A throwing setup skips tests and teardown | https://nodejs.org/docs/latest/api/test.html#global-setup-and-teardown |
| `--test-randomize`, `--test-random-seed=<n>` | early development, added in v26.1.0. Awaited `t.test` loops keep their order | https://nodejs.org/docs/latest/api/test.html#randomizing-tests-execution-order |
| `tags` and `--experimental-test-tag-filter='(unit or smoke) and not slow'` | early development, added in v26.2.0 | https://nodejs.org/docs/latest/api/test.html#test-tags |
| `t.assert.callCount`, `called`, `lastCalledWith` | added in v26.12.0 (`fileSnapshot` is older, v23.7.0) | https://nodejs.org/docs/latest/api/test.html#contextassertcallcountfn-times-message |
| `context.workerId`, `context.attempt` | added in v25.8.0, v25.0.0 | https://nodejs.org/docs/latest/api/test.html |

- Tag filters surprise: untagged tests are included by `not flaky` but excluded by `*` or any positive match. https://nodejs.org/docs/latest/api/test.html#test-tags

## Old patterns

| Old pattern | Replacement | Source |
| :-- | :-- | :-- |
| `mock.module(s, { defaultExport, namedExports })` | `{ exports: { default, ... } }` | https://nodejs.org/docs/latest/api/test.html#mockmodulespecifier-options |
| global `mock` without `mock.reset()` | `t.mock` | https://nodejs.org/docs/latest/api/test.html#mockreset |
| `setTimeout` sleeps in tests | `t.waitFor()` or mock timers | https://nodejs.org/docs/latest/api/test.html#contextwaitforcondition-options |
| `assert.CallTracker` | removed (End-of-Life), count calls with `fn.mock.callCount()` | https://nodejs.org/docs/latest/api/deprecations.html#DEP0173 |
