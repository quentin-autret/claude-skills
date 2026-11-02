# CLI and troubleshooting (Jest 30.5)

## Contents

- General behavior
- Selecting tests
- CI, sharding, workers
- Order and flakiness
- Open handles and hanging runs
- Cache
- Debugging and slow runs

## General behavior

- CLI options override the config file. https://jestjs.io/docs/cli#options
- Camel case and dashed forms are equivalent: `--collect-coverage` = `--collectCoverage`. https://jestjs.io/docs/cli#camelcase--dashed-args-support
- Flags that need a value throw when it is missing (`--maxWorkers=50%`, `--selectProjects <name>`). https://jestjs.io/docs/upgrading-to-jest30#other-cli-changes
- `--json` sends every other output to stderr. https://jestjs.io/docs/cli#--json
- `--reporters` on the CLI cannot carry reporter options: `jest --reporters="default" --reporters="jest-junit"`. https://jestjs.io/docs/cli#--reporters
- `FORCE_COLOR=true|false` overrides color detection. https://jestjs.io/docs/cli#--colors

## Selecting tests

- `--testPathPatterns` takes several patterns, space separated or repeated. https://jestjs.io/docs/upgrading-to-jest30#--testpathpattern-was-renamed-to---testpathpatterns

```bash
jest --testPathPatterns "unit/.*" "integration/.*"
```

https://jestjs.io/docs/upgrading-to-jest30#--testpathpattern-was-renamed-to---testpathpatterns

- `-t` (`--testNamePattern`) matches the full name, `describe` titles included. https://jestjs.io/docs/cli#--testnamepatternregex
- `--runTestsByPath` takes exact paths and skips regex matching (faster with many patterns). A partial path finds nothing. https://jestjs.io/docs/cli#--runtestsbypath
- `--findRelatedTests <files>` runs the minimal set for pre-commit hooks and combines with `--coverage`. https://jestjs.io/docs/cli#--findrelatedtests-spaceseparatedlistofsourcefiles
- `--onlyChanged` (`-o`) needs git or hg and a static dependency graph. `--watch` behaves like `-o`, `--watchAll` runs everything. https://jestjs.io/docs/cli#--onlychanged
- `--changedSince <base>` tests only local changes when the branch diverged. https://jestjs.io/docs/cli#--changedsince
- `--collectTests` lists suites and tests without running them. Load errors exit non-zero. https://jestjs.io/docs/cli#--collecttests
- `--filter` modules return `{filtered: [...]}`. https://jestjs.io/docs/cli#--filterfile

```js
module.exports = testPaths => ({filtered: testPaths.filter(p => p.endsWith('.spec.js'))});
```

https://jestjs.io/docs/cli#--filterfile

- On Windows, path regexes use `/` or escape `\` as `\\`. https://jestjs.io/docs/cli#jest-regexfortestfiles
- With several projects, `--selectProjects` and `--ignoreProjects` need a `displayName` on each. https://jestjs.io/docs/cli#--selectprojects-project1--projectn

## CI, sharding, workers

- Pass `--ci` on CI: a new snapshot fails instead of being written. https://jestjs.io/docs/cli#--ci
- `--shard=1/3` splits the suite (1-based). A custom `testSequencer` must implement `shard`. https://jestjs.io/docs/cli#--shard
- `--maxWorkers=50%` suits variable-CPU machines. https://jestjs.io/docs/cli#--maxworkersnumstring
- `--maxConcurrency` only affects `test.concurrent`. https://jestjs.io/docs/cli#--maxconcurrencynum
- `--workerThreads` is experimental. https://jestjs.io/docs/cli#--workerthreads

## Order and flakiness

- `--randomize` shuffles tests inside each file from a seed (`jest-circus` only). Reproduce an order with `--seed`. https://jestjs.io/docs/cli#--randomize

```bash
jest --randomize --seed 1234
```

https://jestjs.io/docs/cli#--randomize

- The seed also shuffles suite order without `--randomize`. https://jestjs.io/docs/cli#--randomize
- `--testLocationInResults`: `column` is 1-indexed with `jest-circus`, 0-indexed with `jest-jasmine2`. https://jestjs.io/docs/cli#--testlocationinresults
- `--waitForUnhandledRejections` avoids rejections reported as false positives or on the wrong test. https://jestjs.io/docs/cli#--waitforunhandledrejections

## Open handles and hanging runs

- A run that does not exit has open handles or pending timers. Find them with `--detectOpenHandles` (implies `--runInBand`, slow, debug only), then tear them down. `--forceExit` is an escape hatch, not a fix. https://jestjs.io/docs/cli#--forceexit
- `--openHandlesTimeout` (default 1000 ms, `0` disables) warns when the process stays alive. https://jestjs.io/docs/cli#--openhandlestimeoutmilliseconds
- `--workerGracefulExitTimeout` (default 500 ms) force-kills lingering workers. https://jestjs.io/docs/cli#--workergracefulexittimeoutnumber
- "Async callback was not invoked within timeout" usually comes from conflicting Promise implementations. Keep one, or set `globalThis.Promise = jest.requireActual('promise')`. https://jestjs.io/docs/troubleshooting#unresolved-promises

## Cache

- The cache is on by default. `--no-cache` makes Jest at least 2x slower, keep it for cache bugs and transformer development. `--clearCache` deletes the cache folder (`--showConfig` shows it). https://jestjs.io/docs/cli#--cache
- After editing the Babel config and seeing odd behavior, run `jest --clearCache`. https://jestjs.io/docs/webpack#using-with-webpack
- If transform or Babel changes are not picked up, retry with `--no-cache`. A custom transformer should add a `getCacheKey` function. https://jestjs.io/docs/troubleshooting#caching-issues

## Debugging and slow runs

- Attach a debugger with `node --inspect-brk node_modules/.bin/jest --runInBand` (Windows: `node --inspect-brk ./node_modules/jest/bin/jest.js --runInBand`). https://jestjs.io/docs/troubleshooting#tests-are-failing-and-you-dont-know-why
- Slow on Docker or CI: `--runInBand`, `--maxWorkers=4` or `--shard`. https://jestjs.io/docs/troubleshooting#tests-are-extremely-slow-on-docker-andor-continuous-integration-ci-server
- Watchman problems: `--no-watchman` or `watchman: false`. https://jestjs.io/docs/troubleshooting#watchman-issues
- `coveragePathIgnorePatterns` has no effect with `babel-plugin-istanbul`. Jest already wraps Istanbul. https://jestjs.io/docs/troubleshooting#coveragepathignorepatterns-seems-to-not-have-any-effect
- Files imported by `setupFilesAfterEnv` scripts are not mocked. https://jestjs.io/docs/cli#--setupfilesafterenv-path1--pathn
