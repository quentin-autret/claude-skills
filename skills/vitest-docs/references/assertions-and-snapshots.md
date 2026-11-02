# Assertions, waiting, snapshots and type tests

## Contents

- expect pitfalls
- Async assertions
- Waiting for a condition
- Narrowing and schema matching
- Custom matchers and helpers
- Snapshots
- Type tests

## expect pitfalls

- `expect(actual, 'message')` takes a custom failure message as second argument. https://vitest.dev/api/expect
- `toEqual` ignores `undefined` properties and class types. `toStrictEqual` checks them plus sparse arrays. https://vitest.dev/guide/learn/matchers
- `toEqual` on an `Error` compares `name`, `message`, `cause` and `AggregateError.errors`. https://vitest.dev/api/expect
- `toThrow` needs a wrapper for sync code: `expect(() => fn('')).toThrow('msg')`. A string argument is a substring, a RegExp tests the message. `toThrow('')` matches any message. Use `toThrow(/^$/)` for an empty one. https://vitest.dev/api/expect , https://vitest.dev/guide/migration
- `toContain` uses `===`. Objects in arrays need `toContainEqual`. https://vitest.dev/guide/learn/matchers
- `toHaveProperty('a.b')` parses dots. Wrap a literal dotted key in an array: `toHaveProperty(['P.O'])`. https://vitest.dev/api/expect
- `toBeTypeOf` uses `typeof`, so null and arrays are `'object'`. `toBeNullable` accepts null or undefined. https://vitest.dev/api/expect
- `expect.soft(...)` records a failure and lets the test continue. https://vitest.dev/api/expect
- Spy assertions in Chai style are properties, not calls: `expect(spy).to.have.been.called`. https://vitest.dev/api/expect
- Also available: `toHaveBeenCalledExactlyOnceWith`, `toHaveBeenCalledBefore` / `After`, and `toHaveBeenExhausted()` for `vi.when` chains. https://vitest.dev/api/expect
- Reject weak assertions such as `expect(user).toBeDefined()`. Assert real fields with `toMatchObject`. https://vitest.dev/guide/learn/writing-tests-with-ai

## Async assertions

- Always `await` `.resolves`, `.rejects`, `expect.poll` and `toMatchFileSnapshot`. An unawaited one fails the test. https://vitest.dev/guide/migration , https://vitest.dev/api/expect (poll)
- `.rejects.toThrow()` needs no wrapping function. https://vitest.dev/api/expect
- `expect.assertions(n)` / `expect.hasAssertions()` guard callbacks that might never run. `expect.requireAssertions` enforces it project-wide. https://vitest.dev/guide/learn/async
- Unhandled rejections fail the run even when every assertion passed. Usually a forgotten `await` or a fire-and-forget promise. https://vitest.dev/guide/learn/async
- Under fake timers, a promise that rejects during `vi.advanceTimersByTimeAsync` is reported unhandled even if `.rejects` follows. Create the assertion before advancing, then await it. https://vitest.dev/api/expect

```ts
const assertion = expect(run()).rejects.toThrow('timeout')
await vi.advanceTimersByTimeAsync(1000)
await assertion
```
https://vitest.dev/api/expect

## Waiting for a condition

- Never sleep with `setTimeout`. Defaults for the polling helpers: `timeout` 1000 ms, `interval` 50 ms. https://vitest.dev/guide/recipes/wait-for
- Default: `await expect.poll(() => queue.count()).toBe(0)`. It rejects when the callback or assertion does not settle within `timeout`. The callback receives `{ signal }`. https://vitest.dev/guide/migration

```ts
await expect.poll(async ({ signal }) => {
  const response = await fetch('/api/status', { signal })
  return response.status
}, { timeout: 1000 }).toBe(200)
```
https://vitest.dev/guide/migration

- `expect.poll` does not support snapshot matchers, `.resolves` / `.rejects` or `toThrow`. Use `vi.waitFor` for those. https://vitest.dev/guide/recipes/wait-for
- `vi.waitFor(cb)` retries while `cb` throws and returns its value. `vi.waitUntil(cb)` retries while falsy and fails at once on a throw. https://vitest.dev/api/vi
- With fake timers the helpers advance time by `interval` per attempt. https://vitest.dev/guide/mocking/timers
- In browser mode, prefer locators and `expect.element` over `vi.waitUntil` for DOM queries. https://vitest.dev/guide/recipes/wait-for

## Narrowing and schema matching

- `expect.assert(x)` throws and narrows the TypeScript type. `toBeDefined()` and `toBeTruthy()` do not narrow. Use it instead of `!` or `as`. https://vitest.dev/guide/recipes/type-narrowing

```ts
const user = cache.get('alice'); expect.assert(user); expect(user.name).toBe('Alice')
```
https://vitest.dev/guide/recipes/type-narrowing

- `expect.schemaMatching(schema)` accepts any Standard Schema v1 (Zod, Valibot, ArkType) inside `toEqual`, `toMatchObject`, `toHaveBeenCalledWith` and others. https://vitest.dev/guide/recipes/schema-matching

```ts
expect(repo.save).toHaveBeenCalledWith(expect.schemaMatching(UserSchema))
```
https://vitest.dev/guide/recipes/schema-matching

## Custom matchers and helpers

- Wrap a reusable helper that calls `expect` more than once in `vi.defineHelper(fn)`, so failures point at the call site. https://vitest.dev/guide/recipes/custom-assertions
- In `expect.extend`, do not flip `pass` for `.not`. Vitest inverts it. Return `{ pass, message: () => string }`, plus `actual` / `expected` for a diff. https://vitest.dev/guide/extending-matchers

```ts
expect.extend({
  toBeFoo(received) {
    const { isNot } = this
    return { pass: received === 'foo', message: () => `${received} is${isNot ? ' not' : ''} foo` }
  }
})
declare module 'vitest' { interface Matchers<R, T> { toBeFoo: () => R } }
```
https://vitest.dev/guide/extending-matchers

- Augment `Matchers<R, T>` from `vitest` (with `import 'vitest'` in the .d.ts). Vitest does not read the global `jest.Matchers`. https://vitest.dev/guide/migration
- `this.equals` does not compare Map or Set contents. Pass `[...this.customTesters, this.utils.iterableEquality]`. https://vitest.dev/guide/extending-matchers
- Register matchers and serializers in `setupFiles`. https://vitest.dev/api/expect

## Snapshots

- In CI snapshots are never written: mismatched, missing and obsolete snapshots fail. Update locally with `vitest -u` and review the diff. https://vitest.dev/guide/snapshot
- Dynamic fields: `expect(user).toMatchSnapshot({ id: expect.any(Number) })`. https://vitest.dev/guide/learn/snapshots
- `toMatchFileSnapshot(path)` is async and keeps a real file extension. Never point it into `__snapshots__/`. https://vitest.dev/guide/snapshot
- Format differs from Jest: no `Array [` / `Object {` prefixes (`printBasicPrototype` false), `>` instead of `:` before a custom message in names, `toThrowErrorMatchingSnapshot` prints `[Error: error]`. Expect diffs when migrating. https://vitest.dev/guide/snapshot
- Custom snapshot matchers: call `Snapshots.toMatchSnapshot.call(this, transformed)` from `import { Snapshots } from 'vitest'`. The inline snapshot argument must be last. https://vitest.dev/guide/snapshot
- `toMatchAriaSnapshot` asserts the accessibility tree instead of markup. See browser mode. https://vitest.dev/guide/snapshot

## Type tests

- `*.test-d.ts` files are type tests, checked by `tsc` (or `vue-tsc`). Run `vitest --typecheck`. `typecheck.enabled` defaults to `false`, so normal runs do not type-check. https://vitest.dev/guide/testing-types , https://vitest.dev/config/typecheck
- Prefer a type argument: `expectTypeOf(x).toEqualTypeOf<{ a: string }>()`. https://vitest.dev/guide/testing-types . `toMatchTypeOf` is deprecated, use `toExtend`. https://vitest.dev/api/expect-typeof
- `@ts-expect-error` can hide typos. Include type files in `test.include` so a runtime `ReferenceError` reveals them. https://vitest.dev/guide/testing-types

```ts
expectTypeOf(mount).parameter(0).toExtend<{ name: string }>()
// @ts-expect-error name is a string
assertType(mount({ name: 42 }))
```
https://vitest.dev/guide/testing-types

- Source errors outside test files fail typecheck runs unless `typecheck.ignoreSourceErrors`. https://vitest.dev/config/typecheck
