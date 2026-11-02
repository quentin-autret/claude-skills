---
name: jest-docs
description: Writes, fixes and configures Jest 30.5 tests from the official Jest documentation. Use when writing or debugging Jest unit tests, mocking modules or classes with jest.mock, jest.fn or jest.spyOn, asserting with expect and asymmetric matchers, writing custom matchers, managing snapshots, controlling time with fake timers, editing jest.config (transform, moduleNameMapper, projects, coverage, reporters), running native ESM tests with unstable_mockModule, tuning the Jest CLI on CI, or upgrading from Jest 29 to Jest 30 (removed alias matchers, testPathPatterns, genMockFromModule). Triggers on Jest, jest.config, jest.mock, jest.fn, spyOn, expect, toHaveBeenCalledWith, snapshot, toMatchSnapshot, fake timers, useFakeTimers, ESM, experimental-vm-modules, babel-jest, ts-jest, Jest 30 upgrade. Not for Vitest, Mocha or other runners.
metadata:
  version: "30.5"
  source: https://jestjs.io/docs/getting-started
  read: "2026-11-02"
---

# Jest 30.5

## Overview

This skill holds the Jest 30.5 rules that are easy to get wrong: async tests that pass silently, mocks that are not restored, hoisting that ESM breaks, matchers removed in Jest 30, config options silently ignored. Every claim carries the page URL it comes from. The reference files hold the detail per domain.

## Version covered

Jest 30.5, latest stable docs version. Minimum Node 18, minimum TypeScript 5.4 with Jest types. https://jestjs.io/docs/upgrading-to-jest30#compatibility

## Quick start

- Scaffold a config with `npm init jest@latest`. `jest --init` is removed. https://jestjs.io/docs/upgrading-to-jest30#removed---init-command
- Jest discovers `jest.config.js|ts|mjs|mts|cjs|cts|json`. A JS or TS config can wrap its export in `defineConfig` from `jest`. https://jestjs.io/docs/configuration

```js
const {defineConfig} = require('jest');
module.exports = defineConfig({ /* options */ });
```

https://jestjs.io/docs/configuration

- In TypeScript, import the globals rather than relying on `@types/jest`. https://jestjs.io/docs/getting-started#type-definitions

```ts
import {describe, expect, jest, test} from '@jest/globals';
```

- The default environment is `node`. DOM tests install `jest-environment-jsdom` and set `testEnvironment: 'jsdom'`, or a per-file `@jest-environment jsdom` docblock. https://jestjs.io/docs/configuration#testenvironment-node--jsdom--string
- `babel-jest` and `@babel/preset-typescript` do not type-check. Use `ts-jest` or run `tsc` separately. https://jestjs.io/docs/code-transformation#typescript-with-type-checking

## Rules and pitfalls

### Asynchronous tests

- Return or `await` every promise, `.resolves` and `.rejects` assertions included. Otherwise the test passes before the promise settles. https://jestjs.io/docs/asynchronous#promises
- An `async` test can combine `await` with `.rejects`. https://jestjs.io/docs/asynchronous#resolves--rejects

```js
test('the fetch fails with an error', async () => {
  await expect(fetchData()).rejects.toMatch('error');
});
```

- A test that tests a rejection with `.catch` adds `expect.assertions(n)`. https://jestjs.io/docs/asynchronous#promises
- A test function never both takes `done` and returns a promise: Jest throws. https://jestjs.io/docs/asynchronous#callbacks
- Setup goes in `beforeEach` and `beforeAll`, never directly in a `describe` body: every `describe` body runs before any test. https://jestjs.io/docs/setup-teardown#order-of-execution
- Default timeout is 5 seconds. Pass it as the last argument, `test(name, fn, 10_000)`. https://jestjs.io/docs/api#testname-fn-timeout

### Matchers

- `toEqual` ignores `undefined` properties, sparseness and class type. `toStrictEqual` checks them. https://jestjs.io/docs/expect#toequalvalue
- `toBe` uses `Object.is`. Floats use `toBeCloseTo`, nested floats `expect.closeTo`. https://jestjs.io/docs/expect#tobeclosetonumber-numdigits
- `toThrow` needs the call wrapped in a function. A string argument is a substring match. https://jestjs.io/docs/expect#tothrowerror
- `toContain` uses `===`, `toContainEqual` compares structure. https://jestjs.io/docs/expect#tocontainitem
- `expect.anything()` does not match `null` or `undefined`. https://jestjs.io/docs/expect#expectanything
- Custom matchers are added with `expect.extend`, which can move to a `setupFilesAfterEnv` script to enable them for all tests. https://jestjs.io/docs/expect#expectextendmatchers

### Mocks

- `jest.spyOn` calls the original method unless `.mockImplementation()` is added. https://jestjs.io/docs/jest-object#jestspyonobject-methodname
- Restore spies and replaced properties after each test. `jest.restoreAllMocks()` does not restore a hand-assigned `jest.fn()`. https://jestjs.io/docs/jest-object#jestrestoreallmocks

```js
afterEach(() => {
  jest.restoreAllMocks();
});
```

- `resetMocks` empties implementations, `restoreMocks` also restores originals. https://jestjs.io/docs/configuration#resetmocks-boolean
- `jest.mock` is hoisted by `babel-jest`. Factory variables need a `mock` prefix and must be initialized when the factory runs. https://jestjs.io/docs/es6-class-mocks#calling-jestmock-with-the-module-factory-parameter
- A factory for a module with a default export sets `__esModule: true`. https://jestjs.io/docs/jest-object#jestmockmodulename-factory-options
- `jest.mock` paths are case-sensitive. https://jestjs.io/docs/upgrading-to-jest30#jestmock-only-works-with-case-sensitive-module-path
- Node core modules (`fs`, `path`) need an explicit `jest.mock('fs')`, even with `__mocks__/fs.js`. https://jestjs.io/docs/manual-mocks#mocking-node-modules
- A module imported by a `setupFilesAfterEnv` file cannot be mocked. https://jestjs.io/docs/jest-object#jestmockmodulename-factory-options
- Route return values by arguments with `mockFn.whenCalledWith(...)` rather than branching in `mockImplementation`. https://jestjs.io/docs/mock-function-api#mockfnwhencalledwithargs

```js
const fn = jest.fn();
fn.whenCalledWith('apple').mockReturnValue('red');
fn.whenCalledWith(expect.any(Number)).mockReturnValue('numeric');
```

### Fake timers

- `jest.useFakeTimers()` lasts for the file. Restore with `jest.useRealTimers()` in `afterEach`. https://jestjs.io/docs/jest-object#jestusefaketimersfaketimersconfig
- When promises and timers mix, use the async variants (`await jest.advanceTimersByTimeAsync(ms)`, `await jest.runAllTimersAsync()`). They let promise callbacks run between timers. https://jestjs.io/docs/jest-object#jestrunalltimersasync
- Recursive timers use `jest.runOnlyPendingTimers()`. `runAllTimers` aborts after 100000 timers. https://jestjs.io/docs/timer-mocks#run-pending-timers

### Native ESM

- Run with `NODE_OPTIONS="$NODE_OPTIONS --experimental-vm-modules" npx jest` and `transform: {}`. https://jestjs.io/docs/ecmascript-modules
- Import `jest` from `@jest/globals`. Mock with `jest.unstable_mockModule`, then load the code under test with `await import()`. https://jestjs.io/docs/ecmascript-modules#module-mocking-in-esm

```js
import {jest} from '@jest/globals';
jest.unstable_mockModule('node:child_process', () => ({execSync: jest.fn()}));
const {execSync} = await import('node:child_process');
```

- A second `unstable_mockModule` on the same path is flagged "Don't override": in the docs example the first factory is still the one used. The example calls `jest.unstable_unmockModule` before loading the original. https://jestjs.io/docs/ecmascript-modules#module-unmocking-in-esm

### Configuration

- Setting `transform` drops the default `babel-jest`: list it explicitly beside other transformers. https://jestjs.io/docs/code-transformation#defaults
- `transformIgnorePatterns` skips a file matching any pattern: keep all package exceptions in one pattern. https://jestjs.io/docs/configuration#transformignorepatterns-arraystring
- `moduleNameMapper`: patterns are checked in order, so list the most specific first. https://jestjs.io/docs/configuration#modulenamemapper-objectstring-string--arraystring
- Use the `<rootDir>` token in path patterns so they do not depend on the machine's root directory. https://jestjs.io/docs/configuration#coveragepathignorepatterns-arraystring
- In `projects`, run-wide options (`reporters`, `coverageThreshold`, `maxWorkers` and others) are ignored in a project config: set them at root. https://jestjs.io/docs/configuration#projects-arraystring--projectconfig
- Custom `reporters` override the default reporter: keep `'default'` in the list. https://jestjs.io/docs/configuration#default-reporter

### Snapshots and CI

- Pass `--ci` on CI so a new snapshot fails the test instead of being written. https://jestjs.io/docs/cli#--ci
- Generated fields take property matchers: `toMatchSnapshot({id: expect.any(Number)})`. https://jestjs.io/docs/snapshot-testing#property-matchers
- A run that does not exit has open handles: find them with `--detectOpenHandles`, do not hide them with `--forceExit`. https://jestjs.io/docs/cli#--forceexit

## Reference files

| File | Read it for |
| ---- | ----------- |
| [references/core-testing.md](references/core-testing.md) | Setup with TypeScript and Babel, async tests, hooks order, `describe` and `test` modifiers, `test.each` and its typing, snapshots |
| [references/expect-api.md](references/expect-api.md) | Matcher semantics, asymmetric matchers, custom matchers and their typing, equality testers, snapshot matchers |
| [references/mock-functions.md](references/mock-functions.md) | `jest.fn`, clear, reset and restore, `whenCalledWith`, `spyOn`, `replaceProperty`, mock typing, `retryTimes` |
| [references/module-mocking.md](references/module-mocking.md) | `jest.mock` hoisting and factories, partial mocks, `__mocks__`, class mocks, module registry, jsdom gaps |
| [references/fake-timers.md](references/fake-timers.md) | Fake timer APIs, async variants, clock control, `fakeTimers` config, legacy mode |
| [references/configuration.md](references/configuration.md) | Config file, path tokens, test matching, `projects`, module resolution, setup files, environments, transforms |
| [references/run-options.md](references/run-options.md) | Coverage, reporters, workers, sequencer, environment variables set by Jest |
| [references/cli-and-troubleshooting.md](references/cli-and-troubleshooting.md) | CLI flags, test selection, sharding, seeds, open handles, cache, debugging |
| [references/esm-and-transforms.md](references/esm-and-transforms.md) | Native ESM and its mocking, custom transformers, static assets, custom environments, watch plugins |
| [references/upgrading.md](references/upgrading.md) | Jest 29 to 30 changes in full, older Jest 29 changes, migration from other runners |
| [references/integrations.md](references/integrations.md) | React Native, Puppeteer, MongoDB, DynamoDB, platform packages |

## Old patterns

| Old usage | Replacement | Source |
| --------- | ----------- | ------ |
| `toBeCalled`, `toBeCalledTimes`, `toBeCalledWith` | `toHaveBeenCalled`, `toHaveBeenCalledTimes`, `toHaveBeenCalledWith` | https://jestjs.io/docs/upgrading-to-jest30#removal-of-alias-matcher-functions |
| `lastCalledWith`, `nthCalledWith` | `toHaveBeenLastCalledWith`, `toHaveBeenNthCalledWith` | https://jestjs.io/docs/upgrading-to-jest30#removal-of-alias-matcher-functions |
| `toReturn`, `toReturnTimes`, `toReturnWith` | `toHaveReturned`, `toHaveReturnedTimes`, `toHaveReturnedWith` | https://jestjs.io/docs/upgrading-to-jest30#removal-of-alias-matcher-functions |
| `lastReturnedWith`, `nthReturnedWith` | `toHaveLastReturnedWith`, `toHaveNthReturnedWith` | https://jestjs.io/docs/upgrading-to-jest30#removal-of-alias-matcher-functions |
| `toThrowError` | `toThrow` | https://jestjs.io/docs/upgrading-to-jest30#removal-of-alias-matcher-functions |
| `jest.genMockFromModule` | `jest.createMockFromModule` | https://jestjs.io/docs/upgrading-to-jest30#jestgenmockfrommodule-removed |
| `jest.SpyInstance` type | `jest.Spied` | https://jestjs.io/docs/upgrading-to-jest30#removed-mock-function-types |
| `--testPathPattern` | `--testPathPatterns` | https://jestjs.io/docs/upgrading-to-jest30#--testpathpattern-was-renamed-to---testpathpatterns |
| `jest --init` | `npm init jest@latest` | https://jestjs.io/docs/upgrading-to-jest30#removed---init-command |
| Bare flag without value (`--maxWorkers`) | Explicit value (`--maxWorkers=50%`) | https://jestjs.io/docs/upgrading-to-jest30#other-cli-changes |
| `--filter` module returning an array | Return `{filtered: [...]}` | https://jestjs.io/docs/upgrading-to-jest30#other-cli-changes |
| Deep imports such as `jest-runner/build/testWorker` | Public package APIs | https://jestjs.io/docs/upgrading-to-jest30#esm-module-support-and-internal-restructuring |
| `jest.mocked(obj, true)` | `jest.mocked(obj)`, `{shallow: true}` for one level | https://jestjs.io/docs/upgrading-to-jest29#jestmocked |
| `extraGlobals` | `sandboxInjectedGlobals` | https://jestjs.io/docs/configuration#sandboxinjectedglobals-arraystring |
| `react-test-renderer` | `@testing-library/react-native`, awaited `render` | https://jestjs.io/docs/tutorial-react-native#snapshot-test |
| `jest.mock` in native ESM | `jest.unstable_mockModule` plus `await import()` | https://jestjs.io/docs/ecmascript-modules#module-mocking-in-esm |
