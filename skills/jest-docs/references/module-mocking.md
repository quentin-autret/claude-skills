# Module mocking: jest.mock, manual mocks, class mocks (Jest 30.5)

## Contents

- `jest.mock` and hoisting
- Factories and partial mocks
- Manual mocks in `__mocks__`
- Generated automocks
- ES6 class mocks
- Module registry
- jsdom gaps

## `jest.mock` and hoisting

- `jest.mock(path)` without a factory automocks: functions return `undefined`. https://jestjs.io/docs/jest-object#jestmockmodulename-factory-options
- A module mocked with `jest.mock` is mocked only in the file that calls it. https://jestjs.io/docs/jest-object#jestmockmodulename-factory-options
- `jest.mock` paths are case-sensitive: `jest.mock('./FILENAME.js')` fails when only `filename.js` exists. https://jestjs.io/docs/upgrading-to-jest30#jestmock-only-works-with-case-sensitive-module-path
- `babel-jest` hoists `jest.mock`, `unmock`, `enableAutomock` and `disableAutomock` above imports. `jest.doMock`, `dontMock`, `autoMockOn` and `autoMockOff` are not hoisted. https://jestjs.io/docs/jest-object#jestdomockmodulename-factory-options
- Hoisting does not happen with native ESM: static imports are evaluated first. ESM mocking uses `jest.unstable_mockModule` (factory required) with `require` or a dynamic `import()` loaded after it. https://jestjs.io/docs/ecmascript-modules
- `jest.mock(path)` sits in the same scope as the `require` or `import` it targets. https://jestjs.io/docs/manual-mocks#examples
- A module imported by a `setupFilesAfterEnv` file cannot be mocked, nor anything it imports. https://jestjs.io/docs/jest-object#jestmockmodulename-factory-options
- `{virtual: true}` as third argument mocks a module that exists nowhere on disk. https://jestjs.io/docs/jest-object#jestmockmodulename-factory-options
- `jest.unmock` never returns a mock. `jest.deepUnmock` also covers dependencies. https://jestjs.io/docs/jest-object#jestdeepunmockmodulename
- `jest.setMock(name, exports)` takes exports, not a factory. The docs recommend `jest.mock()` instead. https://jestjs.io/docs/jest-object#jestsetmockmodulename-moduleexports

## Factories and partial mocks

- A factory for an ES module with a default export sets `__esModule: true` by hand. https://jestjs.io/docs/jest-object#jestmockmodulename-factory-options
- A hoisted factory cannot reference outside variables unless their name starts with `mock`, and the variable must be initialized when the factory runs (Temporal Dead Zone). Returning a `const mockX` directly from the factory throws `ReferenceError`. Wrap it in a function. https://jestjs.io/docs/es6-class-mocks#calling-jestmock-with-the-module-factory-parameter
- Partial mock: spread `jest.requireActual(...)` and override members. https://jestjs.io/docs/mock-functions#mocking-partials

```js
jest.mock('../foo-bar-baz', () => {
  const originalModule = jest.requireActual('../foo-bar-baz');
  return {
    __esModule: true,
    ...originalModule,
    default: jest.fn(() => 'mocked baz'),
    foo: 'mocked foo',
  };
});
```

https://jestjs.io/docs/mock-functions#mocking-partials

- `jest.requireActual(name)` in a test file fetches real parts of a mocked module, for example the real `Response` of a mocked `node-fetch` (otherwise `TypeError: response.text is not a function`). https://jestjs.io/docs/bypassing-module-mocks

```js
jest.mock('node-fetch');
import fetch from 'node-fetch';
const {Response} = jest.requireActual('node-fetch');
```

https://jestjs.io/docs/bypassing-module-mocks

- `jest.requireMock(name)` returns the mock and bypasses all checks. https://jestjs.io/docs/jest-object#jestrequiremockmodulename
- TypeScript: `jest.mock<typeof import('../m')>(...)` and `jest.requireActual<typeof import('../m')>(...)` take a type argument. https://jestjs.io/docs/jest-object#jestmockmodulename-factory-options

## Manual mocks in `__mocks__`

- A user-module mock lives in `__mocks__/` next to the module (`models/__mocks__/user.js` for `models/user.js`) and needs an explicit `jest.mock('./models/user')`. https://jestjs.io/docs/manual-mocks#mocking-user-modules
- The folder name is case-sensitive: `__MOCKS__` breaks on some systems. https://jestjs.io/docs/manual-mocks#mocking-user-modules
- A `node_modules` package mock lives in `__mocks__` next to `node_modules` (or in one of the `roots`) and applies with no `jest.mock` call. Scoped packages mirror the name: `__mocks__/@scope/project-name.js`. https://jestjs.io/docs/manual-mocks#mocking-node-modules
- Node core modules (`fs`, `path`) are not mocked by default: `jest.mock('fs')` is required even with `__mocks__/fs.js`. https://jestjs.io/docs/manual-mocks#mocking-node-modules
- With `automock: true` the manual mock applies without `jest.mock`. Opt out with `jest.unmock(name)`. https://jestjs.io/docs/manual-mocks#examples
- Build manual mocks from `jest.createMockFromModule` (or `jest.requireActual`) and override, so they track the real module. https://jestjs.io/docs/manual-mocks#examples

```js
// __mocks__/fs.js
const fs = jest.createMockFromModule('fs');
fs.readdirSync = readdirSync; // custom override
module.exports = fs;
```

https://jestjs.io/docs/manual-mocks#examples

- Helpers exported by a `__mocks__` file are imported from the original module path, never through `__mocks__`. https://jestjs.io/docs/es6-class-mocks#spying-on-methods-of-our-class

## Generated automocks

- `jest.createMockFromModule(name)`: functions (async too) become mocks with no formal parameters returning `undefined`, classes keep their interface with mocked members, objects are deep-cloned with mocked values, arrays become empty, primitives keep their value. https://jestjs.io/docs/jest-object#jestcreatemockfrommodulemodulename
- TypeScript: `jest.createMockFromModule<typeof import('../utils')>('../utils')`. https://jestjs.io/docs/jest-object#jestcreatemockfrommodulemodulename
- `jest.onGenerateMock((modulePath, moduleMock) => moduleMock)` edits every generated automock, callbacks chaining in order. It does not run for `__mocks__` files or explicit factories. https://jestjs.io/docs/jest-object#jestongeneratemockcb

## ES6 class mocks

- An automatic class mock turns methods into mocks returning `undefined`. Arrow-function class properties are not mocked (not on the prototype). https://jestjs.io/docs/es6-class-mocks#automatic-mock
- A constructor factory returns a regular function, not an arrow. `new` on an arrow throws `TypeError: ... is not a constructor` unless transpiled to ES5. https://jestjs.io/docs/es6-class-mocks#-module-factory-function-must-return-a-function
- A named class export needs a factory returning an object keyed by the export name. https://jestjs.io/docs/es6-class-mocks#mocking-non-default-class-exports
- Each `new` creates a new object. Share one `jest.fn()` across instances to assert on method calls. https://jestjs.io/docs/es6-class-mocks#spying-on-methods-of-our-class

```js
const mockPlaySoundFile = jest.fn();
jest.mock('./sound-player', () => {
  return jest.fn().mockImplementation(() => {
    return {playSoundFile: mockPlaySoundFile};
  });
});
```

https://jestjs.io/docs/es6-class-mocks#spying-on-methods-of-our-class

- A manual mock written as a plain class injects behavior but cannot track calls. Wrap it in `jest.fn().mockImplementation(...)`. https://jestjs.io/docs/es6-class-mocks#manual-mock-that-is-another-es6-class
- Change the class mock between tests with `mockImplementation` in `beforeAll` rather than a factory, and call `mockClear()` on the class and shared method mocks in `beforeEach`. https://jestjs.io/docs/es6-class-mocks#cleaning-up-between-tests

## Module registry

- Mock one module differently per test with `jest.resetModules()` in `beforeEach` plus `jest.doMock`, then load the module with `require` or dynamic `import()` (with `__esModule: true`). https://jestjs.io/docs/jest-object#jestdomockmodulename-factory-options
- `jest.isolateModules(fn)` gives a sandbox registry for modules loaded inside `fn`. `jest.isolateModulesAsync` is the awaited form. https://jestjs.io/docs/jest-object#jestisolatemodulesfn
- Each test file already has its own registry. https://jestjs.io/docs/configuration#resetmodules-boolean

## jsdom gaps

- APIs missing from jsdom (for example `window.matchMedia`) are mocked with `Object.defineProperty(window, ...)`. When the tested file calls them at import time, put the mock in a separate file imported before it. https://jestjs.io/docs/manual-mocks#mocking-methods-which-are-not-implemented-in-jsdom
- For a module with import-time DOM side effects, set `document.body.innerHTML` first, then `require` the module inside the test. https://jestjs.io/docs/tutorial-jquery
