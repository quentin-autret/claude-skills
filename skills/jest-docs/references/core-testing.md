# Core testing: setup, async, hooks, globals, snapshots (Jest 30.5)

## Contents

- TypeScript, Babel and ESLint setup
- Asynchronous tests
- Setup and teardown hooks
- Global API (`describe`, `test`, modifiers)
- `test.each` and its typing
- Snapshot testing

## TypeScript, Babel and ESLint setup

- Import the globals in TypeScript tests: `import {describe, expect, jest, test} from '@jest/globals'`. These types ship with Jest and update with it. https://jestjs.io/docs/getting-started#type-definitions
- `@types/jest` is the third-party alternative (globals without import). It can lag behind new Jest features, and its version should match the Jest version as closely as possible. https://jestjs.io/docs/getting-started#type-definitions
- `@babel/preset-typescript` only transpiles: Jest does not type-check the tests. Use `ts-jest` or run `tsc` separately. https://jestjs.io/docs/getting-started#via-babel
- `babel-jest` ships with Jest and transforms files automatically as soon as a Babel config exists. Disable it with `transform: {}`. https://jestjs.io/docs/getting-started#using-babel
- Jest sets `process.env.NODE_ENV` to `'test'` when it is unset. Use `api.env('test')` in `babel.config.js` for test-only compilation, with the target `[['@babel/preset-env', {targets: {node: 'current'}}]]`. https://jestjs.io/docs/getting-started#using-babel
- Vite is not supported (plugin system incompatibilities). `vite-jest` stops working after Vite 2.4.2. The docs point to Vitest instead. https://jestjs.io/docs/getting-started#using-vite
- ESLint: importing from `@jest/globals` avoids `no-undef` with no config. Otherwise add `...globals.jest` to `languageOptions.globals` (flat config) or use `eslint-plugin-jest`. https://jestjs.io/docs/getting-started#using-eslint

## Asynchronous tests

- Return or `await` every promise, `.resolves` and `.rejects` included. Without it the test ends before the promise settles and passes silently. https://jestjs.io/docs/asynchronous#promises
- A callback-style test with no parameter ends before the callback runs. Take a `done` parameter and Jest waits for it. https://jestjs.io/docs/asynchronous#callbacks
- With `done`, wrap `expect` in `try/catch` and call `done(error)`. Otherwise a failing `expect` throws before `done()` and the test only reports a timeout. https://jestjs.io/docs/asynchronous#callbacks

```js
test('the data is peanut butter', done => {
  function callback(error, data) {
    if (error) {
      done(error);
      return;
    }
    try {
      expect(data).toBe('peanut butter');
      done();
    } catch (error) {
      done(error);
    }
  }
  fetchData(callback);
});
```

https://jestjs.io/docs/asynchronous#callbacks

- Jest throws when a test function both takes `done` and returns a promise, `async` functions included. https://jestjs.io/docs/asynchronous#callbacks
- When a rejection is tested with `.catch` or `try/catch`, add `expect.assertions(n)`. Otherwise a fulfilled promise does not fail the test. https://jestjs.io/docs/asynchronous#promises
- `.rejects` and `.resolves` fail on the opposite outcome by themselves. `expect.assertions(n)` stays recommended with them because a missing `return` or `await` is easy. https://jestjs.io/docs/tutorial-async#rejects

```js
test('the fetch fails with an error', async () => {
  await expect(fetchData()).rejects.toMatch('error');
});
```

https://jestjs.io/docs/asynchronous#resolves--rejects

- `async/await` under Babel needs `@babel/preset-env`. https://jestjs.io/docs/tutorial-async#asyncawait

## Setup and teardown hooks

- `beforeEach`, `afterEach`, `beforeAll` and `afterAll` wait for a returned promise or a `done` parameter, like tests. Return the promise: `return initializeCityDatabase();`. https://jestjs.io/docs/setup-teardown#repeating-setup
- Top-level hooks apply to every test of the file. Hooks inside a `describe` apply only to that block. https://jestjs.io/docs/setup-teardown#scoping
- Every `describe` body runs during collection, before any test. Put setup and teardown in hooks rather than directly in a `describe` body. By default, tests then run serially in collection order. https://jestjs.io/docs/setup-teardown#order-of-execution
- `before*` hooks run in declaration order, outer `beforeEach` before inner `beforeEach`. `after*` hooks also run in declaration order, inner scope first. The `jasmine2` runner calls `after*` in reverse order. https://jestjs.io/docs/setup-teardown#order-of-execution
- A test that passes alone with `test.only` but fails in the suite shares state with another test. Reset that state in `beforeEach`. https://jestjs.io/docs/setup-teardown#general-advice
- When data is set up once in `beforeAll`, tests should not modify it. https://jestjs.io/docs/api#beforeallfn-timeout

## Global API

- Default timeout for tests and hooks is 5 seconds. Pass the timeout in ms as the last argument: `test(name, fn, 10_000)`, `beforeEach(fn, 10_000)`. With `.each` it applies per row. https://jestjs.io/docs/api#testname-fn-timeout
- Hooks also wait when `fn` is a generator. https://jestjs.io/docs/api#beforeallfn-timeout
- `describe.skip` still runs the `describe` body. Setup that must be skipped belongs in `beforeAll` or `beforeEach`. https://jestjs.io/docs/api#describeskipname-fn
- `test.todo(name)` takes no callback and throws when given one. Use `test.skip` for written but disabled tests. https://jestjs.io/docs/api#testtodoname
- `test.failing` passes when the body throws and fails when it does not. It needs the default `jest-circus` runner. Variants: `test.failing.each`, `test.only.failing`, `test.skip.failing`. https://jestjs.io/docs/api#testfailingname-fn-timeout
- `test.concurrent` is experimental, its function must be async, and `maxConcurrency` caps it. https://jestjs.io/docs/api#testconcurrentname-fn-timeout
- Aliases: `it` = `test`, `fit` = `test.only`, `xit` and `xtest` = `test.skip`, `fdescribe` = `describe.only`, `xdescribe` = `describe.skip`. https://jestjs.io/docs/api
- Tests are defined synchronously. A `test` created inside `setTimeout` is never collected, and a `test.each` table cannot be filled in `beforeEach` or `beforeAll`. https://jestjs.io/docs/troubleshooting#defining-tests

## `test.each` and its typing

- Forms: array of arrays, array of objects, tagged template. A 1D array of primitives becomes `[[1], [2], [3]]`. https://jestjs.io/docs/api#testeachtablename-fn-timeout
- Title placeholders: `%p` pretty-format, `%s` string, `%d` number, `%i` integer, `%f` float, `%j` JSON, `%o` object, `%#` index of the test case, `%$` case number, `%%` literal percent. https://jestjs.io/docs/api#testeachtablename-fn-timeout
- Object rows use `$variable` and `$variable.path.to.value` in the title (own properties only), `$#` for the index. `$variable` cannot be mixed with printf placeholders except `%%`. https://jestjs.io/docs/api#testeachtablename-fn-timeout

```js
test.each`
  a    | b    | expected
  ${1} | ${1} | ${2}
  ${1} | ${2} | ${3}
`('returns $expected when $a is added to $b', ({a, b, expected}) => {
  expect(a + b).toBe(expected);
});
```

https://jestjs.io/docs/api#testeachtablename-fn-timeout

- TypeScript: prefer an array of objects, inferred correctly inline or as a variable. https://jestjs.io/docs/api#array-of-objects
- An array of arrays declared as a variable needs a tuple type such as `Array<[number, number, string, boolean?]>`, otherwise arguments widen to a union. https://jestjs.io/docs/api#array-of-arrays
- With a template literal whose inputs have different types, arguments are typed as a union. Pass a generic: `test.each<{a: number; b: number; expected: string}>`. The template values are then not type-checked. https://jestjs.io/docs/api#template-literal

## Snapshot testing

- Commit snapshots and review them like code. Keep them short (`eslint-plugin-jest` rule `no-large-snapshots`). Investigate a failure before regenerating. https://jestjs.io/docs/snapshot-testing#1-treat-snapshots-as-code
- On CI, snapshots are not written automatically unless `--updateSnapshot` is passed, because new snapshots would otherwise pass a CI run. https://jestjs.io/docs/snapshot-testing#are-snapshots-written-automatically-on-continuous-integration-ci-systems
- Never run `-u` while unintended failures remain, or the bug gets recorded. Narrow re-recording with `--testNamePattern`. https://jestjs.io/docs/snapshot-testing#updating-snapshots
- `toMatchInlineSnapshot()` with no argument writes the value into the source. When `prettier` is installed Jest writes through it. https://jestjs.io/docs/snapshot-testing#inline-snapshots
- Generated fields (ids, dates) take asymmetric property matchers as the first argument. They are stored as `Any<Date>`, other values are compared exactly. https://jestjs.io/docs/snapshot-testing#property-matchers

```js
expect(user).toMatchSnapshot({
  createdAt: expect.any(Date),
  id: expect.any(Number),
});
```

https://jestjs.io/docs/snapshot-testing#property-matchers

- Random parts inside a plain string are replaced with `.replace()` before snapshotting, or handled by a serializer or a mocked generator. https://jestjs.io/docs/snapshot-testing#property-matchers
- Snapshots contain no platform-specific or time-dependent data. Mock the clock, for example `Date.now = jest.fn(() => 1_482_363_367_071);`. https://jestjs.io/docs/snapshot-testing#2-tests-should-be-deterministic
- Test names state the expected content (`should render Alan Turing`) so a wrong snapshot shows in review. https://jestjs.io/docs/snapshot-testing#3-use-descriptive-snapshot-names
- Any serializable value can be snapshotted (`pretty-format`), not only React output. https://jestjs.io/docs/snapshot-testing#does-snapshot-testing-only-work-with-react-components
- Snapshot merge conflicts are resolved by hand or by rerunning Jest and inspecting the result. https://jestjs.io/docs/snapshot-testing#how-do-i-resolve-conflicts-within-snapshot-files
