# ESM, code transformation, test environments, watch plugins (Jest 30.5)

## Contents

- Native ESM
- Mocking in ESM
- `require()` of ESM (Node 24.9+)
- Divergences from Node
- Transformers
- Static assets
- Custom test environments
- Watch plugins

## Native ESM

- ESM support is experimental, like the Node `vm` modules API under it. https://jestjs.io/docs/ecmascript-modules
- Enable it with `transform: {}` (or a transformer emitting ESM) and Node's `--experimental-vm-modules` flag. On Windows use `cross-env`. https://jestjs.io/docs/ecmascript-modules

```bash
NODE_OPTIONS="$NODE_OPTIONS --experimental-vm-modules" npx jest
```

https://jestjs.io/docs/ecmascript-modules

- Files follow Node rules (`"type": "module"`, `.mjs`). Other extensions go in `extensionsToTreatAsEsm`, for example `['.ts']`. https://jestjs.io/docs/configuration#extensionstotreatasesm-arraystring
- The `jest` object is not injected: import it from `@jest/globals` or use `import.meta.jest`. https://jestjs.io/docs/ecmascript-modules#differences-between-esm-and-commonjs

## Mocking in ESM

- Static imports run before any code, so `jest.mock` is not hoisted. Register `jest.unstable_mockModule(path, factory)` (factory required), then load the module under test, and every module that imports the mock, with `await import()`. https://jestjs.io/docs/ecmascript-modules#module-mocking-in-esm

```js
import {jest} from '@jest/globals';
jest.unstable_mockModule('node:child_process', () => ({execSync: jest.fn()}));
const {execSync} = await import('node:child_process');
```

https://jestjs.io/docs/ecmascript-modules#module-mocking-in-esm

- A second `unstable_mockModule` on an already mocked path does not replace the first factory. Call `jest.unstable_unmockModule` first. https://jestjs.io/docs/ecmascript-modules#module-unmocking-in-esm
- CJS targets keep `jest.mock`. https://jestjs.io/docs/ecmascript-modules#mocking-cjs-modules
- On Node 24.9+, automocks and `__mocks__` files also apply to ESM with synchronously evaluable graphs. Graphs with top-level await, or an async-only resolver or transformer, still need `unstable_mockModule`. https://jestjs.io/docs/ecmascript-modules#module-mocking-in-esm

## `require()` of ESM (Node 24.9+)

- CJS can `require()` ESM. A graph with top-level await throws `ERR_REQUIRE_ASYNC_MODULE` (use `await import()`), as does an async-only transformer or resolver. Older Node throws `ERR_REQUIRE_ESM`. https://jestjs.io/docs/ecmascript-modules#require-of-esm
- Packages resolve through the `require` and `module-sync` conditions. An entry exposed only under `import` fails with `ERR_PACKAGE_PATH_NOT_EXPORTED`. https://jestjs.io/docs/ecmascript-modules#require-of-esm
- `jest.mock` does not apply when the resolved file is ESM. `jest.unstable_mockModule` covers the required file and its ESM dependencies. https://jestjs.io/docs/ecmascript-modules#require-of-esm

## Divergences from Node

- Writes to `require.cache` are ignored. JSON imports without `with {type: 'json'}` only warn. Stack traces show paths, not `file://` URLs. `import source` throws `ERR_SOURCE_PHASE_NOT_DEFINED`. `require()` of a module in a loading graph throws `ERR_REQUIRE_CYCLE_MODULE`. https://jestjs.io/docs/ecmascript-modules#divergences-from-node

## Transformers

- Default transformer: `babel-jest` on `/\.[jt]sx?$/`. It loads the Babel config and injects the plugin that hoists `jest.mock`. `excludeJestPreset: true` removes that hoisting. https://jestjs.io/docs/code-transformation#defaults
- Adding another entry to `transform` drops the default: list `babel-jest` explicitly. https://jestjs.io/docs/code-transformation#defaults

```json
"transform": {
  "\\.[jt]sx?$": "babel-jest",
  "\\.css$": "some-css-transformer"
}
```

https://jestjs.io/docs/code-transformation#examples

- `babel-jest` does not type-check, `ts-jest` does. https://jestjs.io/docs/code-transformation#typescript-with-type-checking
- A custom transformer implements `getCacheKey` (helper `@jest/create-cache-key-function`), returns a source map, and exports `createTransformer` to receive options. Develop it with `--no-cache`. https://jestjs.io/docs/code-transformation#writing-custom-transformers
- `require` calls sync `process`. `import` can use `processAsync`. Anything loaded through `require` (`createRequire` included) needs sync `process`. https://jestjs.io/docs/code-transformation#writing-custom-transformers
- `supportsStaticESM` and the other `supports*` flags choose ESM or CJS output, not sync or async. `supportsStaticESM` is true only with `--experimental-vm-modules` and an extension in `extensionsToTreatAsEsm`. https://jestjs.io/docs/code-transformation#writing-custom-transformers
- `node_modules` is not transformed unless `transformIgnorePatterns` changes. https://jestjs.io/docs/code-transformation#writing-custom-transformers

```js
// fileTransformer.js
const path = require('path');
module.exports = {
  process(sourceText, sourcePath, options) {
    return {code: `module.exports = ${JSON.stringify(path.basename(sourcePath))};`};
  },
};
```

https://jestjs.io/docs/code-transformation#examples

## Static assets

- Map assets to stubs with `moduleNameMapper`, CSS Modules to `identity-obj-proxy` (`styles.foobar === 'foobar'`). https://jestjs.io/docs/webpack#mocking-css-modules

```js
module.exports = {
  moduleNameMapper: {
    '\\.(css|less)$': 'identity-obj-proxy',
    '\\.(jpg|jpeg|png|gif|svg|woff2)$': '<rootDir>/__mocks__/fileMock.js',
  },
};
```

https://jestjs.io/docs/webpack#mocking-css-modules

- Webpack equivalents: `resolve.modules` = `moduleDirectories`, `resolve.extensions` = `moduleFileExtensions`, `resolve.roots` = `modulePaths`, `resolve.alias` = `moduleNameMapper`. https://jestjs.io/docs/webpack#configuring-jest-to-find-our-files

## Custom test environments

- `jsdom` needs the separate `jest-environment-jsdom` package. https://jestjs.io/docs/tutorial-jquery
- Each test file gets its own environment instance. https://jestjs.io/docs/test-environment#environments-for-specific-files
- Extend `TestEnvironment` from `jest-environment-node`, call `super.setup()` first and `super.teardown()` last. The constructor receives `(config, context)`, with `context.testPath` and `context.docblockPragmas`. https://jestjs.io/docs/test-environment#extending-built-in-environments
- `@jest/environment-jsdom-abstract` builds a jsdom environment on another jsdom version: `super(config, context, jsdom)`. https://jestjs.io/docs/test-environment#extending-built-in-environments
- `handleTestEvent` may return a Promise, awaited except for `start_describe_definition`, `finish_describe_definition`, `add_hook`, `add_test` and `error`. https://jestjs.io/docs/test-environment#extending-built-in-environments

## Watch plugins

- A plugin implements `apply(jestHooks)`, `getUsageInfo()` returning `{key, prompt}`, and `run(globalConfig, updateConfigAndRun)` resolving `true` to rerun. https://jestjs.io/docs/watch-plugins#watch-plugin-interface
- When `run` calls `updateConfigAndRun`, it must not resolve truthy, or tests run twice. https://jestjs.io/docs/watch-plugins#runglobalconfig-updateconfigandrun
- Keys `c`, `i`, `q`, `u`, `w` cannot be overridden. https://jestjs.io/docs/watch-plugins#choosing-a-good-key
