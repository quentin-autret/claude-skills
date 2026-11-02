# Configuration: jest.config and its options (Jest 30.5)

## Contents

- Config file
- Paths and test matching
- Projects (monorepo)
- Module resolution
- Setup files and globals
- Test environment options
- Transforms

## Config file

- Default: a dedicated `jest.config.js|ts|mjs|mts|cjs|cts|json`, found automatically (`--config` gives another path). The object must be JSON-serializable. https://jestjs.io/docs/configuration
- `defineConfig` from `jest` can wrap the object, and `mergeConfig` extends options from another config file. https://jestjs.io/docs/configuration

```js
const {defineConfig} = require('jest');
module.exports = defineConfig({ /* options */ });
```

https://jestjs.io/docs/configuration

- Extend defaults through `defaults` from `jest-config`: `moduleDirectories: [...defaults.moduleDirectories, 'bower_components']`. https://jestjs.io/docs/configuration
- `jest.config.ts` needs `ts-node` by default. A docblock selects `ts-node` or `esbuild-register` (installed) and passes options. https://jestjs.io/docs/configuration

```ts
/** @jest-config-loader ts-node */
/** @jest-config-loader-options {"transpileOnly": true} */
```

https://jestjs.io/docs/configuration

## Paths and test matching

- Use `<rootDir>` in `coveragePathIgnorePatterns`, `testPathIgnorePatterns` and `transformIgnorePatterns` (patterns match the full path), or they can accidentally ignore all files where the root directory differs. `<rootDir>` also refers to `rootDir` in file paths such as `setupFiles` and `moduleNameMapper`. https://jestjs.io/docs/configuration#testpathignorepatterns-arraystring
- `rootDir` defaults to the folder of the config file or `package.json`. `roots` is what Jest uses to find tests, sources and `__mocks__` of node modules. https://jestjs.io/docs/configuration#roots-arraystring
- Default `testMatch`: `["**/__tests__/**/*.?([mc])[jt]s?(x)", "**/?(*.)+(spec|test).?([mc])[jt]s?(x)"]`. `testMatch` and `testRegex` cannot both be set. https://jestjs.io/docs/configuration#testmatch-arraystring
- `testRegex` matches the absolute path, so a matching folder name turns every file in it into a test. https://jestjs.io/docs/configuration#testregex-string--arraystring
- Negations in `testMatch` and `collectCoverageFrom` apply in order. Put the `!` pattern after the pattern it carves out. https://jestjs.io/docs/configuration#collectcoveragefrom-array

## Projects (monorepo)

- `projects` takes paths, globs or inline configs, and can mix runners. Give each project a `displayName`. https://jestjs.io/docs/configuration#projects-arraystring--projectconfig

```js
projects: [
  {displayName: 'test'},
  {displayName: 'lint', runner: 'jest-runner-eslint', testMatch: ['<rootDir>/**/*.js']},
],
```

https://jestjs.io/docs/configuration#projects-arraystring--projectconfig

- Root-only options, ignored when set in a project config: `bail`, `coverageReporters`, `coverageThreshold`, `maxConcurrency`, `maxWorkers`, `notify`, `notifyMode`, `randomize`, `reporters`, `showSeed`, `testFailureExitCode`, `testResultsProcessor`, `testSequencer`, `watchPlugins`, `watchman`, `workerGracefulExitTimeout`, `workerIdleMemoryLimit`, `workerThreads`, plus `ci`, `passWithNoTests`, `updateSnapshot`, `watch`. https://jestjs.io/docs/configuration#projects-arraystring--projectconfig
- Root options are copied into each project, `<rootDir>` resolving to the project root. https://jestjs.io/docs/configuration#projects-arraystring--projectconfig
- A project `globalSetup` or `globalTeardown` runs only when a test of that project runs. https://jestjs.io/docs/configuration#globalsetup-string

## Module resolution

- `moduleNameMapper`: the first matching pattern wins, so put specific rules first. Anchor with `^` and `$` (unanchored `relay` also catches `react-relay`). Backreferences and fallback arrays work. https://jestjs.io/docs/configuration#modulenamemapper-objectstring-string--arraystring

```js
moduleNameMapper: {
  'module_name_(.*)': '<rootDir>/substituted_module_$1.js',
  'assets/(.*)': ['<rootDir>/images/$1', '<rootDir>/photos/$1'],
},
```

https://jestjs.io/docs/configuration#modulenamemapper-objectstring-string--arraystring

- Mapped modules are unmocked even with `automock`. https://jestjs.io/docs/configuration#modulenamemapper-objectstring-string--arraystring
- `moduleDirectories` keeps `'node_modules'` and avoids `'.'`, which breaks scoped packages. Prefer `moduleNameMapper`. https://jestjs.io/docs/configuration#moduledirectories-arraystring
- `moduleFileExtensions` is searched left to right: put `ts` and `tsx` first in TypeScript projects. https://jestjs.io/docs/configuration#modulefileextensions-arraystring
- `resolver` exports `(path, options)` or `{sync, async}`. Calling `options.defaultResolver` keeps its cache and error handling (`jest-resolve` relies on `unrs-resolver`). https://jestjs.io/docs/configuration#resolver-string

```js
module.exports = (path, options) =>
  options.defaultResolver(path, {...options, mainFields: ['react-native', 'main']});
```

https://jestjs.io/docs/configuration#resolver-string

## Setup files and globals

- `setupFiles` run before the test framework is installed in the environment. `setupFilesAfterEnv` run right after it, so Jest globals, the `jest` object and `expect` are accessible: the place for `expect.extend` and global hooks. https://jestjs.io/docs/configuration#setupfilesafterenv-array
- A CJS setup file may export an async function, an ESM one may use top-level await. https://jestjs.io/docs/configuration#setupfiles-array
- `globalSetup` and `globalTeardown` export a function of `(globalConfig, projectConfig)` and run once per run. Globals they set are visible only to `globalTeardown`, not to tests. `node_modules` is never transformed for them. https://jestjs.io/docs/configuration#globalsetup-string
- `globals` must be JSON-serializable (no functions, use `setupFiles`). https://jestjs.io/docs/configuration#globals-object
- `injectGlobals: false` requires importing `expect`, `test`, `jest` from `@jest/globals` (`jest-circus` only). https://jestjs.io/docs/configuration#injectglobals-boolean
- `sandboxInjectedGlobals: ['Math']` speeds up access to globals inside the test `vm`. No effect with native ESM. https://jestjs.io/docs/configuration#sandboxinjectedglobals-arraystring

## Test environment options

- `testEnvironment` default is `node`. Use `jsdom` for browser code, per file with the `@jest-environment jsdom` docblock. https://jestjs.io/docs/configuration#testenvironment-node--jsdom--string
- jsdom options: `url` (default `http://localhost`), `userAgent`, any jsdom option such as `html`. https://jestjs.io/docs/configuration#jsdom-environment-options
- `customExportConditions` defaults: jsdom `['browser']`, node `['node', 'node-addons']`. https://jestjs.io/docs/configuration#custom-export-conditions
- `globalsCleanup` (`'on'`, `'soft'` default, `'off'`): at the end of a file Jest removes globals it added. In `'soft'` a later access warns `JEST-01`, meaning code holds a global past its file. It is set once per worker by its first environment. https://jestjs.io/docs/configuration#cleaning-up-globals

## Transforms

- Default `transform` is `{"\\.[jt]sx?$": "babel-jest"}`. To use `babel-jest` alongside other transformers, include it explicitly. Options use the tuple form `['babel-jest', {rootMode: 'upward'}]`. https://jestjs.io/docs/configuration#transform-objectstring-pathtotransformer--pathtotransformer-object
- `transformIgnorePatterns` default is `["/node_modules/", "\\.pnp\\.[^\\\/]+$"]`. A file matching any pattern is skipped, so keep all exceptions in one pattern. https://jestjs.io/docs/configuration#transformignorepatterns-arraystring
- With pnpm, match the `.pnpm` folder and use `@` after the package name, `+` for scope slashes. https://jestjs.io/docs/configuration#transformignorepatterns-arraystring

```js
transformIgnorePatterns: ['<rootDir>/node_modules/.pnpm/(?!(package-a|@scope\\+pkg-b)@)'],
```

https://jestjs.io/docs/configuration#transformignorepatterns-arraystring
