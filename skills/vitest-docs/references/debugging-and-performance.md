# Debugging, common errors and performance

## Contents

- Debugging a test
- Common errors
- Dependencies and CommonJS
- Measuring time
- Speeding up a suite
- Disabling isolation safely

## Debugging a test

- Narrow first: `vitest src/user.test.js -t "name"`, `--bail 1`. A test passing alone but failing in the suite points to shared state. https://vitest.dev/guide/learn/debugging-tests
- With breakpoints add `--test-timeout=0` and `--no-file-parallelism`. https://vitest.dev/guide/debugging
- VS Code: run `vitest` in a JavaScript Debug Terminal. Elsewhere: `vitest --inspect-brk --no-file-parallelism` then `chrome://inspect`. https://vitest.dev/guide/debugging
- Browser mode: `vitest --inspect-brk --browser --no-file-parallelism`. https://vitest.dev/guide/debugging
- `--detectAsyncLeaks` reports timers and handles still alive after a file. Much slower: debugging only. https://vitest.dev/config/detectasyncleaks
- `--shuffle --run` twice: different results reveal order-dependent tests. https://vitest.dev/guide/recipes/disable-isolation

## Common errors

- "Cannot find module" with tsconfig paths: Vite ignores tsconfig by default, add `vite-tsconfig-paths`. https://vitest.dev/guide/common-errors
- "Failed to terminate worker" (Node `fetch` under `threads`) and native segfaults: switch to `pool: 'forks'`. https://vitest.dev/guide/common-errors
- `process.env.TZ` set in a setup file, test or `env` has no effect under `threads` / `vmThreads`. Set it in the shell, the config top level or `globalSetup`. https://vitest.dev/guide/common-errors
- Custom `package.json` conditions are ignored. In the node environment set `ssr.resolve.conditions`. `resolve.conditions` is for DOM and browser. https://vitest.dev/guide/common-errors
- jsdom lacks layout (`offsetTop`) and some APIs. https://vitest.dev/guide/comparisons Browser mode runs tests in a real browser. https://vitest.dev/guide/browser/why

## Dependencies and CommonJS

- "Unexpected token 'export'", "Cannot use import statement outside a module" or `Unknown file extension ".css"` come from an externalized package loaded by Node. Fix the package first. Otherwise inline the whole chain: https://vitest.dev/guide/common-errors

```ts
test: { server: { deps: { inline: ['wrapper-package', 'broken-package'] } } }
```
https://vitest.dev/guide/common-errors

- `require()` goes straight to Node: no plugins, aliases, transforms or mocks apply. Importing and requiring one file can evaluate it twice. https://vitest.dev/guide/common-errors

## Measuring time

- The `Duration` line splits time by phase (environment, import, transform, setup, tests) as percentages of the summed phase time, not of wall clock. https://vitest.dev/guide/improving-performance
- `vitest doctor` measures alternative configs, including `isolate: false`, validated by running the suite twice with a shuffled file order. https://vitest.dev/guide/cli
- Slow imports: `--experimental.importDurations.print`. Import specific entry points (`date-fns/format`) instead of barrels. A barrel import can pull in many unneeded transformed files (the page shows ~85% fewer without the barrel). https://vitest.dev/guide/profiling-test-performance
- Profile workers with `test.execArgv: ['--cpu-prof', '--cpu-prof-dir=test-runner-profile']` and `fileParallelism: false`. https://vitest.dev/guide/profiling-test-performance

## Speeding up a suite

- DOM environments cost per file. `happy-dom` is cheaper than `jsdom`. https://vitest.dev/guide/improving-performance
- Try `--pool=threads` on a large suite. Keep `forks` when native modules or `process.chdir` are involved (`threads` cannot use `process.chdir()`). https://vitest.dev/guide/common-errors
- `--no-isolate` greatly speeds node-environment suites that clean up after themselves. https://vitest.dev/guide/improving-performance
- `globalSetup` files run once in the main process before any worker is created: the place for one-time work. https://vitest.dev/guide/lifecycle
- Keep setup files light: they run before every test file, even with isolation disabled. https://vitest.dev/guide/lifecycle
- On many-core machines the single Vite server limits throughput. Shard locally and merge blob reports. https://vitest.dev/guide/improving-performance
- `NODE_COMPILE_CACHE=node_modules/.cache/node-compile-cache` helps only if the folder survives between runs. https://vitest.dev/guide/improving-performance

## Disabling isolation safely

- Disable isolation per project, not globally. A file is safe only if it does not mutate module-level state, call `vi.stubGlobal` / `vi.stubEnv`, patch prototypes, add `process` listeners or rely on a fresh module for `vi.mock`. https://vitest.dev/guide/recipes/disable-isolation
- `isolate` only governs `threads` and `forks`. The vm pools always isolate. https://vitest.dev/guide/recipes/disable-isolation
