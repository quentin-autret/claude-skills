# Mock functions, spies and replaced properties (Jest 30.5)

## Contents

- Clear, reset, restore
- Recorded state
- Implementations and return values
- `whenCalledWith` routing
- `withImplementation`
- `jest.spyOn`
- `jest.replaceProperty`
- TypeScript typing
- Other `jest` object helpers

## Clear, reset, restore

- `mockClear()` empties `mock.calls`, `mock.instances`, `mock.contexts` and `mock.results`. `mockReset()` also replaces the implementation with one returning `undefined`. `mockRestore()` also puts the original implementation back. https://jestjs.io/docs/mock-function-api#mockfnmockclear
- `mockClear()` replaces the `mockFn.mock` object, so a variable holding `mockFn.mock` goes stale. https://jestjs.io/docs/mock-function-api#mockfnmockclear
- `mockRestore()` and `jest.restoreAllMocks()` only restore spies from `jest.spyOn()` and properties from `jest.replaceProperty()`. A hand-assigned `jest.fn()` is restored by hand. https://jestjs.io/docs/jest-object#jestrestoreallmocks
- Config `clearMocks`, `resetMocks` and `restoreMocks` run the matching global call before each test. `resetMocks` does not restore originals, `restoreMocks` does. https://jestjs.io/docs/configuration#resetmocks-boolean
- Default cleanup after `spyOn` or `replaceProperty`: https://jestjs.io/docs/jest-object#jestspyonobject-methodname

```js
afterEach(() => {
  jest.restoreAllMocks();
});
```

## Recorded state

- `mock.results[i].type` is `'return'`, `'throw'` or `'incomplete'`. `'incomplete'` (value `undefined`) appears when results are read from inside the mock or a function it called. https://jestjs.io/docs/mock-function-api#mockfnmockresults
- `mock.contexts` records `this` per call. `mock.lastCall` is the last argument array or `undefined`. https://jestjs.io/docs/mock-function-api#mockfnmockcontexts
- `mockName('add42')` replaces `jest.fn()` in failure output. https://jestjs.io/docs/mock-functions#mock-names

## Implementations and return values

- A mock with an implementation still records calls and instances. https://jestjs.io/docs/mock-function-api#mockfnmockimplementationfn
- `*Once` values queue and fire first, in order. When the queue is empty the default (`jest.fn(impl)`, `mockImplementation`, `mockReturnValue`) applies. https://jestjs.io/docs/mock-function-api#mockfnmockreturnvalueoncevalue

```js
const mockFn = jest
  .fn(() => 'default')
  .mockImplementationOnce(() => 'first call')
  .mockImplementationOnce(() => 'second call');
// 'first call', 'second call', then 'default'
```

https://jestjs.io/docs/mock-function-api#mockfnmockimplementationoncefn

- Async functions use `mockResolvedValue(v)` and `mockRejectedValue(v)` (and `*Once`). https://jestjs.io/docs/mock-function-api#mockfnmockresolvedvaluevalue
- Chained methods use `mockReturnThis()`. https://jestjs.io/docs/mock-function-api#mockfnmockreturnthis
- Inject return values rather than reimplementing logic of code not under test. https://jestjs.io/docs/mock-functions#mock-return-values
- `mockImplementation` can also mock class constructors: the implementation returns the object that `new` yields. https://jestjs.io/docs/mock-function-api#mockfnmockimplementationfn

```js
jest.mock('./SomeClass');
const mockMethod = jest.fn();
SomeClass.mockImplementation(() => ({method: mockMethod}));
```

https://jestjs.io/docs/mock-function-api#mockfnmockimplementationfn

## `whenCalledWith` routing

- `mockFn.whenCalledWith(...args)` routes a return value or implementation by argument list, instead of branching inside `mockImplementation`. It returns a real mock that chains every `mockReturnValue*`, `mockResolvedValue*`, `mockRejectedValue*` and `mockImplementation*`. https://jestjs.io/docs/mock-function-api#mockfnwhencalledwithargs

```js
const fn = jest.fn();
fn.whenCalledWith('apple').mockReturnValue('red');
fn.whenCalledWith(expect.any(Number)).mockReturnValue('numeric');
fn('grape'); // undefined
```

https://jestjs.io/docs/mock-function-api#mockfnwhencalledwithargs

- Each slot takes a literal or an asymmetric matcher, compared structurally. Testers from `expect.addEqualityTesters` are not applied. https://jestjs.io/docs/mock-function-api#matching
- Arity counts: `whenCalledWith('a')` matches `fn('a')` and `fn('a', undefined)`, not `fn('a', 'b')` nor `fn()`. https://jestjs.io/docs/mock-function-api#matching
- Unmatched calls return `undefined` on `jest.fn()` and reach the original method on a spy. https://jestjs.io/docs/mock-function-api#on-spies
- A drained branch (onces consumed, no persistent value) falls through to the base mock. https://jestjs.io/docs/mock-function-api#fall-through
- Each call returns a new branch, even with identical matchers. https://jestjs.io/docs/mock-function-api#repeat-calls-return-fresh-branches
- Precedence: base `*Once` queue, then the earliest matching branch with a queued once, then the last registered matching branch with a persistent value, then the base implementation. https://jestjs.io/docs/mock-function-api#precedence
- `mockReset()` on the parent clears every branch. On a branch it clears that branch only. https://jestjs.io/docs/mock-function-api#resetting

## `withImplementation`

- `mock.withImplementation(fn, callback)` applies `fn` during `callback` only. With an async callback, `await` the call. https://jestjs.io/docs/mock-function-api#mockfnwithimplementationfn-callback

```js
await mock.withImplementation(() => 'inside', async () => { mock(); });
```

https://jestjs.io/docs/mock-function-api#mockfnwithimplementationfn-callback

## `jest.spyOn`

- `spyOn` calls the original method by default, unlike most test libraries. Add `.mockImplementation(...)` to replace it. https://jestjs.io/docs/jest-object#jestspyonobject-methodname
- Accessors take a third argument: `jest.spyOn(obj, 'foo', 'get')` or `'set'`. https://jestjs.io/docs/jest-object#jestspyonobject-methodname-accesstype
- Class methods are spied on the prototype, statics on the class: `jest.spyOn(SoundPlayer.prototype, 'playSoundFile')`, `jest.spyOn(SoundPlayer, 'brand')`. https://jestjs.io/docs/es6-class-mocks#mocking-a-specific-method-of-a-class
- `using spy = jest.spyOn(...)` restores at block exit (TypeScript 5.2+ or the Babel explicit-resource-management plugin). If `Symbol.dispose` is reported missing, polyfill it. https://jestjs.io/docs/jest-object#spied-methods-and-the-using-keyword

```js
if (!Symbol.dispose) {
  Object.defineProperty(Symbol, 'dispose', {get() { return Symbol.for('nodejs.dispose'); }});
}
```

https://jestjs.io/docs/jest-object#spied-methods-and-the-using-keyword

## `jest.replaceProperty`

- `jest.replaceProperty(obj, key, value)` needs an existing property and returns an object with `replaceValue(v)` and `restore()`. https://jestjs.io/docs/jest-object#jestreplacepropertyobject-propertykey-value
- Getters, setters and functions use `jest.spyOn` instead. https://jestjs.io/docs/jest-object#jestreplacepropertyobject-propertykey-value
- Environment variables: replace `process.env` itself, which always exists, and restore with `jest.restoreAllMocks()` in `afterEach`. https://jestjs.io/docs/jest-object#jestreplacepropertyobject-propertykey-value

```js
afterEach(() => {
  jest.restoreAllMocks();
});

test('isLocalhost returns true when HOSTNAME is localhost', () => {
  jest.replaceProperty(process, 'env', {HOSTNAME: 'localhost'});
  expect(utils.isLocalhost()).toBe(true);
});
```

https://jestjs.io/docs/jest-object#jestreplacepropertyobject-propertykey-value

## TypeScript typing

- Import `jest` from `@jest/globals` for typed usage. https://jestjs.io/docs/jest-object
- `jest.fn<typeof add>()` or `jest.fn<(a: string) => void>()` takes a function type. With an implementation the type is inferred. https://jestjs.io/docs/mock-function-api#jestfnimplementation
- `jest.mocked(source)` types a module or object deeply and returns it. `{shallow: true}` types one level. https://jestjs.io/docs/mock-function-api#jestmockedsource-options

```ts
import {jest} from '@jest/globals';
jest.mock('./song');
const mockedSong = jest.mocked(song);
mockedSong.one.more.time.mockReturnValue(12);
```

https://jestjs.io/docs/mock-function-api#jestmockedsource

- Types: `jest.Mock<T>`, `jest.Mocked<S>`, `jest.MockedClass`, `jest.MockedFunction`, `jest.MockedObject`, `jest.Replaced<S>`, `jest.Spied<S>`, `jest.SpiedClass`, `jest.SpiedFunction`, `jest.SpiedGetter`, `jest.SpiedSetter`. https://jestjs.io/docs/mock-function-api#jestmockedsource
- `jest.Spied<Source>` takes the type of a class or function, for example `let spiedDateNow: jest.Spied<typeof Date.now> | undefined`. `jest.SpiedClass<Source>` and `jest.SpiedFunction<Source>` constrain the input. https://jestjs.io/docs/mock-function-api#jestspiedsource

## Other `jest` object helpers

- `jest.setTimeout(ms)` applies to the calling file only. Other scopes use the per-test timeout argument or `testTimeout`. https://jestjs.io/docs/jest-object#jestsettimeouttimeout
- `jest.retryTimes(n, options?)` needs `jest-circus` and sits at file top level or in a `describe`. Options: `logErrorsBeforeRetry`, `waitBeforeRetry` (ms), `retryImmediately`, `entireDescribe`. By default retries run after the other tests of the file. https://jestjs.io/docs/jest-object#jestretrytimesnumretries-options
- `entireDescribe: true` retries the whole block, hooks included, and disables per-test retries below it. Nested budgets multiply. `afterAll` failures do not trigger it. https://jestjs.io/docs/jest-object#jestretrytimesnumretries-options
- `jest.getSeed()` returns the run seed, set by `--seed=<num>` and printed by `--showSeed`. https://jestjs.io/docs/jest-object#jestgetseed
