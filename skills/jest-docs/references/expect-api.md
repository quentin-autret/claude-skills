# expect API: matchers, asymmetric matchers, custom matchers (Jest 30.5)

## Contents

- Equality and identity
- Numbers, strings, collections, errors
- Mock call matchers
- Asymmetric matchers
- Promises and assertion counts
- Custom matchers
- Custom equality testers
- Snapshot matchers and serializers
- Matcher state

## Equality and identity

- `.toBe` calls `Object.is` to compare values, which the docs call better for testing than `===`. https://jestjs.io/docs/expect#tobevalue
- `.toBe` reports a deep diff on failure. For a huge unhelpful diff write `expect(Object.is(received, expected)).toBe(true)`. https://jestjs.io/docs/expect#tobevalue
- `.toEqual` ignores `undefined` properties, `undefined` array items, array sparseness and object type. `.toStrictEqual` checks them: `{a: undefined, b: 2}` is not `{b: 2}`, `[, 1]` is not `[undefined, 1]`, a class instance is not an equal literal. https://jestjs.io/docs/expect#tostrictequalvalue
- `.toEqual` does not deep-compare errors, it only looks at `message`. Use `.toThrow` for errors. https://jestjs.io/docs/expect#toequalvalue
- For objects with their own comparator, assert the comparator: `expect(received.equals(expected)).toBe(true)` for a `Buffer`. https://jestjs.io/docs/expect#toequalvalue
- Since Jest 30, non-enumerable properties are excluded from object matchers. https://jestjs.io/docs/upgrading-to-jest30#non-enumerable-properties
- `.toMatchObject` accepts extra properties on the received object. With arrays the lengths must match, each element may carry extra properties. https://jestjs.io/docs/expect#tomatchobjectobject
- Prefer `.toBeNull()` over `.toBe(null)` and `.toBeDefined()` over `.not.toBe(undefined)`. https://jestjs.io/docs/expect#tobenull

## Numbers, strings, collections, errors

- Floats use `.toBeCloseTo(n, numDigits)`, never `.toBe`. Default `numDigits` is 2, criterion `Math.abs(expected - received) < 0.005`. It does not support big integers. https://jestjs.io/docs/expect#tobeclosetonumber-numdigits
- `.toContain` compares array items with `===`. Use `.toContainEqual` for structural match. `.toContain` also takes strings, sets, node lists and HTML collections. https://jestjs.io/docs/expect#tocontainitem
- `.toHaveProperty` takes a dot path or an array. A key containing a dot needs the array form: `['ceiling.height']`. https://jestjs.io/docs/expect#tohavepropertykeypath-value
- `.toThrow` needs the call wrapped in a function. Argument: a regexp matches the message, a string checks a substring, an error object compares the message, a class checks `instanceof`. https://jestjs.io/docs/expect#tothrowerror

```js
expect(() => compileAndroidCode()).toThrow(/^you are using the wrong JDK!$/);
```

https://jestjs.io/docs/using-matchers#exceptions

## Mock call matchers

- `.toHaveBeenCalledWith` compares arguments with the `.toEqual` algorithm, not the strict one. https://jestjs.io/docs/expect#tohavebeencalledwitharg1-arg2-
- `.toHaveBeenNthCalledWith` and `.toHaveNthReturnedWith` count from 1. https://jestjs.io/docs/expect#tohavebeennthcalledwithnthcall-arg1-arg2-
- `.toHaveReturnedTimes` does not count calls that threw. `.toHaveLastReturnedWith` and `.toHaveNthReturnedWith` fail when that call threw. https://jestjs.io/docs/expect#tohavereturnedtimesnumber
- TypeScript: the `CalledWith` family infers parameter types, so wrong argument types are compile errors. https://jestjs.io/docs/upgrading-to-jest30#improved-type-inference-for-calledwith

## Asymmetric matchers

- `expect.anything()` does not match `null` or `undefined`. https://jestjs.io/docs/expect#expectanything
- `expect.arrayContaining` is a subset check: the received array may contain extra items. https://jestjs.io/docs/expect#expectarraycontainingarray
- `expect.arrayOf(value)` asserts every element matches. `expect.not.arrayOf(value)` matches when not all elements match. https://jestjs.io/docs/expect#expectarrayofvalue
- `expect.closeTo(n, numDigits?)` is the float check inside objects and arrays. https://jestjs.io/docs/expect#expectclosetonumber-numdigits
- `expect.not.stringContaining` and `expect.not.stringMatching` also match non-strings. https://jestjs.io/docs/expect#expectnotstringcontainingstring

```js
expect(onPress).toHaveBeenCalledWith(
  expect.objectContaining({x: expect.any(Number), y: expect.any(Number)}),
);
expect([{id: 1, name: 'Alice'}]).toEqual(
  expect.arrayOf(expect.objectContaining({id: expect.any(Number), name: expect.any(String)})),
);
expect({sum: 0.1 + 0.2}).toEqual({sum: expect.closeTo(0.3, 5)});
```

https://jestjs.io/docs/expect#asymmetric-matchers

## Promises and assertion counts

- Return or `await` assertions using `.resolves` and `.rejects`. https://jestjs.io/docs/expect#resolves

```js
await expect(Promise.reject(new Error('octopus'))).rejects.toThrow('octopus');
```

https://jestjs.io/docs/expect#resolves

- `expect.assertions(n)` or `expect.hasAssertions()` proves that assertions inside callbacks ran. https://jestjs.io/docs/expect#expectassertionsnumber

## Custom matchers

- Register with `expect.extend` in a `setupFilesAfterEnv` file so every test gets them. https://jestjs.io/docs/expect#expectextendmatchers
- Return `{pass, message}` where `message` is a zero-argument function. When `pass` is true, `message` describes the failure of the `.not` form. https://jestjs.io/docs/expect#custom-matchers-api
- `this` exposes `isNot`, `promise` (`'rejects'`, `'resolves'` or `''`), `equals`, `expand`, `utils` and `customTesters`. https://jestjs.io/docs/expect#thisisnot
- A custom matcher also works as an asymmetric matcher: `expect.toBeWithinRange(1, 10)`. Async matchers return a Promise and must be awaited. https://jestjs.io/docs/expect#async-matchers

```js
expect.extend({
  toBeWithinRange(actual, floor, ceiling) {
    const pass = actual >= floor && actual <= ceiling;
    return {
      pass,
      message: () =>
        `expected ${this.utils.printReceived(actual)} ${pass ? 'not ' : ''}to be within range ${this.utils.printExpected(`${floor} - ${ceiling}`)}`,
    };
  },
});
```

https://jestjs.io/docs/expect#expectextendmatchers

- TypeScript: type matcher arguments as `unknown` and validate at runtime. Declare the types in a `.d.ts` module (it needs `export {}`). https://jestjs.io/docs/expect#expectextendmatchers

```ts
declare module 'expect' {
  interface AsymmetricMatchers {
    toBeWithinRange(floor: number, ceiling: number): void;
  }
  interface Matchers<R> {
    toBeWithinRange(floor: number, ceiling: number): R;
  }
}
export {};
```

https://jestjs.io/docs/expect#expectextendmatchers

## Custom equality testers

- `expect.addEqualityTesters([...])` registers global testers (import `expect` from `@jest/globals`). A tester returns `true`, `false`, or `undefined` to delegate. They apply to all equality comparisons and test authors cannot turn them off per assertion, so use a custom matcher for opt-in behavior. https://jestjs.io/docs/expect#custom-equality-testers-api
- Define testers as regular functions, not arrow functions, to reach `this.equals`. A recursive tester calls `this.equals(a, b, customTesters)` and passes the third argument through. https://jestjs.io/docs/expect#recursive-custom-equality-testers

```js
function areVolumesEqual(a, b) {
  const isAVolume = a instanceof Volume;
  const isBVolume = b instanceof Volume;
  if (isAVolume && isBVolume) return a.equals(b);
  else if (isAVolume === isBVolume) return undefined;
  else return false;
}
expect.addEqualityTesters([areVolumesEqual]);
```

https://jestjs.io/docs/expect#expectaddequalitytesterstesters

- The JavaScript `Volume.equals` example of the docs compares `other.unit` where it means `other.amount`. The TypeScript version is right. https://jestjs.io/docs/expect#expectaddequalitytesterstesters

## Snapshot matchers and serializers

- `toMatchSnapshot(propertyMatchers?, hint?)`: property matchers apply only to object values. `hint` names several snapshots in one test. https://jestjs.io/docs/expect#tomatchsnapshotpropertymatchers-hint
- A custom snapshot matcher calls `toMatchSnapshot.call(this, value, 'name')` from `jest-snapshot`. https://jestjs.io/docs/expect#custom-snapshot-matchers
- Custom inline snapshot matchers cannot take custom arguments. An async one that fails with "Multiple inline snapshots for the same call are not supported" sets `this.error = new Error()` before any `await`. https://jestjs.io/docs/expect#async
- `this.dontThrow = () => {}` in a custom snapshot matcher bails out on the first mismatch. https://jestjs.io/docs/expect#bail-out
- `expect.addSnapshotSerializer` in the test file makes the dependency explicit. Order: last added first, then `snapshotSerializers` config, then built-ins. https://jestjs.io/docs/expect#expectaddsnapshotserializerserializer
- `SERIALIZABLE_PROPERTIES` on a prototype limits which properties print in failure messages. https://jestjs.io/docs/expect#serializable_properties

## Matcher state

- `expect.getState()` fields such as `currentTestName` and `testPath` can be `undefined` outside a test. https://jestjs.io/docs/expect#expectgetstate
- `expect.setState` merges an object into the state, and the docs say to set only custom properties. TypeScript augments `MatcherState` with `declare module 'expect'`. https://jestjs.io/docs/expect#expectsetstatestate
