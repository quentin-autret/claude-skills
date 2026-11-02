# Mocking with vi

## Contents

- Cleanup between tests
- Mock functions and spies
- Module mocking
- Class mocks
- Conditional mocks with vi.when
- Timers and dates
- Globals and env
- Network and file system

## Cleanup between tests

- Config defaults: `clearMocks: true` (history cleared before each test, implementations kept), `mockReset`, `restoreMocks`, `unstubEnvs`, `unstubGlobals` all `false`. https://vitest.dev/config/clearmocks
- Default: set `restoreMocks: true` in the config instead of `afterEach(() => vi.restoreAllMocks())`. https://vitest.dev/guide/learn/mock-functions
- Ladder: `mockClear` clears history. `mockReset` also resets the implementation to the one given to `vi.fn(impl)` (empty for `vi.fn()`). `mockRestore` also restores the original descriptor of a `vi.spyOn` target. https://vitest.dev/api/mock
- `vi.restoreAllMocks()` only restores `vi.spyOn` spies. It does not clear history or touch automocked modules. After a restore, call `vi.spyOn` again to spy anew. https://vitest.dev/api/vi
- The auto-cleanup options are unsafe with `test.concurrent`: one finishing test clears the mocks of all running tests. https://vitest.dev/config/restoremocks
- Scoped cleanup: `using spy = vi.spyOn(console, 'log').mockImplementation(() => {})` restores at block exit (TypeScript 5.2+, Node 24+). Fallback: `onTestFinished(() => spy.mockRestore())`. https://vitest.dev/guide/recipes/explicit-resources

## Mock functions and spies

- `mock.calls` keeps argument references. Mutating an object after the call changes what the assertion sees. Clone in the implementation (`structuredClone(arg)`). https://vitest.dev/api/mock
- `mock.results` records a returned promise as `type: 'return'` even if it rejects. Use `mock.settledResults` for resolved values. https://vitest.dev/api/mock
- `mockThrow` / `mockThrowOnce` exist. `withImplementation(fn, cb)` overrides temporarily and must be awaited for an async callback. https://vitest.dev/api/mock
- Getters and setters: `vi.spyOn(obj, 'name', 'get').mockReturnValue('Max')`. https://vitest.dev/guide/mocking/classes
- `vi.mocked(fn)` is a type helper only (`{ partial: true }`, `{ deep: true }`). https://vitest.dev/api/vi
- `vi.mockObject(obj)` deep-mocks an object. With `{ spy: true }` it replaces methods in place. https://vitest.dev/api/vi
- Types come from `vitest`: `import type { Mock } from 'vitest'`, `Mock<(name: string) => number>`. https://vitest.dev/guide/migration/jest

## Module mocking

- Default form: pass `import('./x.js')`, not a string. TypeScript checks the path and types `importOriginal` and the factory. https://vitest.dev/guide/mocking/modules . With tsconfig `paths` aliases, use the relative path inside `import()`, or the types do not resolve. https://vitest.dev/api/vi

```ts
vi.mock(import('./some-path.js'), async (importOriginal) => {
  const mod = await importOriginal()
  return { ...mod, mocked: vi.fn() }
})
```
https://vitest.dev/guide/mocking/modules

- `vi.mock`, `vi.unmock` and `vi.hoisted` are hoisted above imports and must sit at module top level: anywhere else they throw. https://vitest.dev/guide/migration
- The factory cannot use outer variables. Declare them with `vi.hoisted(() => ({ fn: vi.fn() }))`, or use `vi.doMock` (not hoisted, affects later dynamic imports only). https://vitest.dev/api/vi
- Import `vi` straight from `vitest` (or use globals) in that file. A re-exported `vi` is not detected for hoisting. https://vitest.dev/api/vi
- Only `import` is mocked, never `require()`. https://vitest.dev/api/vi
- A factory must return every export explicitly, including `default`: `vi.mock('./p', () => ({ default: 'hello' }))`. https://vitest.dev/guide/migration/jest . It may be async and runs once, its result cached until `vi.unmock` / `vi.doUnmock`. https://vitest.dev/api/vi
- Modules imported by a setup file are already cached and will not be mocked. A `vi.mock` in a setup file applies to every test file. https://vitest.dev/api/vi , https://vitest.dev/guide/mocking/modules
- Without a factory Vitest uses `__mocks__` (root folder for packages, next to the file otherwise) or automocks. Nothing is mocked unless `vi.mock` is called. https://vitest.dev/api/vi
- `vi.mock(path, { spy: true })` keeps real implementations and tracks calls. For Node built-ins it patches in place for every consumer in the worker. https://vitest.dev/guide/mocking/modules
- A function called from the same file cannot be mocked. Split the file or inject the dependency. https://vitest.dev/guide/mocking/modules
- To patch one export, prefer `vi.spyOn(namespace, 'fn')` with `import * as ns`. It works with the module runner of Node-based environments, not in browser mode. https://vitest.dev/guide/mocking/modules
- `vi.resetModules()` clears the module cache but not the mock registry. `vi.importActual` replaces `jest.requireActual` and is async. https://vitest.dev/api/vi , https://vitest.dev/guide/migration/jest
- Mocks do not reach external libraries that import the module. Add the library to `server.deps.inline`. https://vitest.dev/guide/migration/jest

## Class mocks

- Mock classes with `class` syntax: `vi.fn(class { feed = vi.fn() })`. Arrow functions cannot be called with `new` (error "<anonymous> is not a constructor"). https://vitest.dev/api/vi , https://vitest.dev/guide/mocking/classes

```ts
vi.mock(import('./dog.js'), () => {
  const Dog = vi.fn(class {
    feed = vi.fn()
  })
  return { Dog }
})
```
https://vitest.dev/guide/mocking/classes

- `mockReturnValue`, `mockResolvedValue` and friends throw when the mock is called with `new`. Use `mockImplementation(class { ... })`. https://vitest.dev/api/mock
- Class fields give each instance its own mock. Prototype methods are shared by all instances. https://vitest.dev/guide/mocking/classes
- `vi.fn(Dog)` keeps the prototype and `instanceof`, but `dog.speak` is the real method, not a spy. Spy one instance with `vi.spyOn(instance, 'speak')`. https://vitest.dev/guide/mocking/classes , https://vitest.dev/guide/migration

## Conditional mocks with vi.when

- Default for argument-specific behavior instead of `if` inside `mockImplementation`. Arguments match with deep equality and asymmetric matchers. https://vitest.dev/guide/recipes/conditional-mocking

```ts
vi.when(db.findById)
  .calledWith(1).thenResolve({ id: 1, name: 'Ella' })
  .calledWith(2).thenResolve({ id: 2, name: 'Gracie' })
```
https://vitest.dev/guide/recipes/conditional-mocking

- Different `calledWith` behaviors match first registered first: put specific matchers before broad ones. Actions on one behavior are consumed last registered first: register the fallback first. https://vitest.dev/guide/recipes/conditional-mocking
- New args that already match an existing behavior (after `calledWith(expect.any(String))`) merge into it. https://vitest.dev/guide/recipes/conditional-mocking
- Unmatched calls pass through by default. Use `onUnmatched: 'throw'` to catch them. https://vitest.dev/api/vi
- `expect(w).toHaveBeenExhausted()` checks every behavior was consumed. https://vitest.dev/guide/recipes/conditional-mocking

## Timers and dates

- `vi.useFakeTimers()` in `beforeEach`, `vi.useRealTimers()` in `afterEach`. Fake timers also fake `Date` and `Temporal` when present. https://vitest.dev/guide/mocking/dates , https://vitest.dev/guide/migration
- `vi.setSystemTime()` works without fake timers, mocking only `Date.*` and `Temporal.Now.*`. https://vitest.dev/api/vi
- `nextTick` and `queueMicrotask` are not faked by default. Faking `nextTick` hangs under `pool: 'forks'`. https://vitest.dev/config/faketimers
- `vi.runAllTimers()` throws after 10 000 tries (endless `setInterval`, limit set by `fakeTimers.loopLimit`). Use `advanceTimersByTime(ms)` or `runOnlyPendingTimers`. https://vitest.dev/api/vi
- Use the `*Async` variants when timer callbacks schedule promises. https://vitest.dev/api/vi
- `vi.setTimerTickMode('nextTimerAsync' | 'interval')` advances automatically. Default is manual. https://vitest.dev/guide/mocking/timers
- `fakeTimers.fakeNodeBuiltins` also fakes `node:timers` imports. https://vitest.dev/config/faketimers
- Replace `jest.setTimeout(5_000)` with `vi.setConfig({ testTimeout: 5_000 })`. https://vitest.dev/guide/migration/jest

## Globals and env

- `vi.stubGlobal(name, value)` and `vi.stubEnv(name, value)` are undone by `vi.unstubAllGlobals()` / `vi.unstubAllEnvs()` or the `unstub*` options. Direct assignment cannot be undone. https://vitest.dev/api/vi
- `vi.stubEnv` sets `process.env` and `import.meta.env`. `undefined` removes the variable. https://vitest.dev/api/vi

## Network and file system

- Mock HTTP with Mock Service Worker in a setup file: `server.listen({ onUnhandledRequest: 'error' })` in `beforeAll`, `resetHandlers()` in `afterEach`, `close()` in `afterAll`. https://vitest.dev/guide/mocking/requests
- No built-in fs mock. Use `memfs` in `__mocks__/fs.cjs` and `__mocks__/fs/promises.cjs`, `vi.mock('node:fs')` and `vi.mock('node:fs/promises')`, `vol.reset()` in `beforeEach`. https://vitest.dev/guide/mocking/file-system
