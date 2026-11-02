# Coverage, reporters, workers and run options (Jest 30.5)

These options live in `jest.config.*`. In a `projects` setup, coverage reporters and thresholds, reporters, workers and the sequencer are root-only and ignored inside a project. https://jestjs.io/docs/configuration#projects-arraystring--projectconfig

## Contents

- Coverage
- Reporters
- Workers and performance
- Runner, sequencer, misc
- Environment variables set by Jest

## Coverage

- `collectCoverage` slows tests. `coverageProvider` is `babel` (default, `/* istanbul ignore next */`) or `v8` (`/* c8 ignore next */`). https://jestjs.io/docs/configuration#collectcoverage-boolean
- `collectCoverageFrom` includes untested files and needs `collectCoverage` or `--coverage`. "Unknown% (0/0)" most likely means the globs match no files. https://jestjs.io/docs/configuration#collectcoveragefrom-array

```js
collectCoverageFrom: ['**/*.{js,jsx}', '!**/node_modules/**', '!**/vendor/**'],
```

https://jestjs.io/docs/configuration#collectcoveragefrom-array

- Negations in `collectCoverageFrom` apply in order: put the `!` pattern after the pattern it carves out. https://jestjs.io/docs/configuration#collectcoveragefrom-array
- `coverageReporters` replaces the defaults `["clover","json","lcov","text"]`: keep `"text"` for console output. Options use the tuple form `['text', {skipFull: true}]`. https://jestjs.io/docs/configuration#coveragereporters-arraystring--string-options
- `coverageThreshold`: positive numbers are minimum percentages, negative numbers the maximum count of uncovered items. Path thresholds apply separately, `global` covers the rest. A path that is not found is an error. https://jestjs.io/docs/configuration#coveragethreshold-object

```js
coverageThreshold: {
  global: {branches: 80, functions: 80, lines: 80, statements: -10},
  './src/api/very-important-module.js': {branches: 100, functions: 100, lines: 100, statements: 100},
},
```

https://jestjs.io/docs/configuration#coveragethreshold-object

- `coveragePathIgnorePatterns` has no effect with `babel-plugin-istanbul`. https://jestjs.io/docs/troubleshooting#coveragepathignorepatterns-seems-to-not-have-any-effect

## Reporters

- `reporters` replaces the default reporter: include `'default'` to keep it. https://jestjs.io/docs/configuration#default-reporter

```js
reporters: ['default', ['jest-junit', {outputDirectory: 'reports', outputName: 'report.xml'}]],
```

https://jestjs.io/docs/configuration#default-reporter

- On GitHub Actions use `[['github-actions', {silent: false}], 'summary']`, without `'default'`. https://jestjs.io/docs/configuration#github-actions-reporter
- `summaryThreshold` (default 20 suites) prints a detailed failure summary: `['default', {summaryThreshold: 10}]`. https://jestjs.io/docs/configuration#summary-reporter
- A custom reporter takes `(globalConfig, reporterOptions, reporterContext)`, and `getLastError()` returning an `Error` forces a non-zero exit. https://jestjs.io/docs/configuration#custom-reporters
- `testFailureExitCode` does not change the exit code of Jest errors such as invalid config. https://jestjs.io/docs/configuration#testfailureexitcode-number

## Workers and performance

- `maxWorkers` default: cores minus 1, half in watch mode. Use `'50%'` on variable-CPU machines. https://jestjs.io/docs/configuration#maxworkers-number--string
- `maxConcurrency` (default 5) limits `test.concurrent` only. https://jestjs.io/docs/configuration#maxconcurrency-number
- `workerIdleMemoryLimit` recycles a worker above a limit (`'50%'`, `'65MB'`). Values `<= 1` are a fraction of memory. Percentages fail on Linux CircleCI. https://jestjs.io/docs/configuration#workeridlememorylimit-numberstring
- `workerThreads` is experimental and loses extra properties on `Error`, `Map` and `Set`. https://jestjs.io/docs/configuration#workerthreads
- `haste.forceNodeFilesystemAPI: true` is suggested: the default `find(1)` crawler walks `node_modules` and `.git` fully. https://jestjs.io/docs/configuration#haste-object
- `waitForUnhandledRejections` waits one event loop turn before reporting a rejection, avoiding false positives at some cost. https://jestjs.io/docs/configuration#waitforunhandledrejections-boolean
- `testTimeout` default is `5000` ms, `openHandlesTimeout` `1000` ms, `workerGracefulExitTimeout` `500` ms (raise it for slow teardown). https://jestjs.io/docs/configuration#testtimeout-number

## Runner, sequencer, misc

- `testRunner` default is `jest-circus/runner`. https://jestjs.io/docs/configuration#testrunner-string
- `runner` options use the tuple form and arrive as the third constructor argument. Custom runners extend `CallbackTestRunner` or `EmittingTestRunner` from `jest-runner`. https://jestjs.io/docs/configuration#runner-string--string-object
- `testSequencer` extends `@jest/test-sequencer` with `sort(tests)` and `shard(tests, {shardIndex, shardCount})`. Sharding runs before sorting. https://jestjs.io/docs/configuration#testsequencer-string
- `errorOnDeprecated: true` makes deprecated APIs throw, useful while upgrading. https://jestjs.io/docs/configuration#errorondeprecated-boolean
- `snapshotFormat` default is `{escapeString: false, printBasicPrototype: false}`. https://jestjs.io/docs/configuration#snapshotformat-object
- `watchPathIgnorePatterns`: only `.git`, `.hg` and `.sl` are ignored by default. https://jestjs.io/docs/configuration#watchpathignorepatterns-arraystring
- `notify` needs the `node-notifier` package. https://jestjs.io/docs/configuration#notify-boolean

## Environment variables set by Jest

- `NODE_ENV` becomes `'test'` only when unset. https://jestjs.io/docs/environment-variables
- `JEST_WORKER_ID` is a 1-based id per worker (set to `1` for all tests when `runInBand` is true). https://jestjs.io/docs/environment-variables
