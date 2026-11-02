# Config, projects, environments and CLI

## Contents

- Config file resolution
- include, globals, environment
- Pools, workers, isolation
- Projects
- CLI, filtering and watch
- Reporters

## Config file resolution

- Without `--config`, Vitest picks `vitest.config.*` first, then `vite.config.*`, in the root. JSON is not supported. Parent directories are not searched. https://vitest.dev/config , https://vitest.dev/guide/migration
- A `vitest.config.*` replaces `vite.config.*` entirely: no plugin or alias from the Vite file applies. To keep it, merge: https://vitest.dev/config

```ts
import { defineConfig, mergeConfig } from 'vitest/config'
import viteConfig from './vite.config'
export default mergeConfig(viteConfig, defineConfig({ test: { /* ... */ } }))
```
https://vitest.dev/config

- Or add a `test` key to `vite.config.ts` with `/// <reference types="vitest/config" />` on top. https://vitest.dev/config
- Vite options (`define`, `resolve.alias`, `plugins`) go at the top level, not inside `test`. https://vitest.dev/config
- Extend defaults rather than replace them: `exclude: [...configDefaults.exclude, 'packages/template/*']`. https://vitest.dev/config
- `VITEST_SKIP_INSTALL_CHECKS=1` disables prompts to install optional packages (CI, scripts). https://vitest.dev/config

## include, globals, environment

- Default `include`: `['**/*.{test,spec}.?(c|m)[jt]s?(x)']`. Setting it replaces the default. Spread `configDefaults.include` to extend. https://vitest.dev/config/include
- `globals` defaults to `false`: import `test`, `expect`, `vi` from `vitest`. With `globals: true` add `"types": ["vitest/globals"]` to tsconfig (and `./node_modules` back to a custom `typeRoots`). https://vitest.dev/config/globals
- Some libraries, such as `@testing-library/react`, rely on globals being present to perform their automatic cleanup. https://vitest.dev/config/globals
- Default environment is `node`. `jsdom` and `happy-dom` do not come with Vitest and must be installed separately (https://vitest.dev/guide/features). Per file: `// @vitest-environment jsdom` at the top. Per folder: use projects (https://vitest.dev/guide/projects). https://vitest.dev/guide/environment , https://vitest.dev/config/environment
- "document is not defined" means the test ran in the `node` environment. https://vitest.dev/guide/learn/debugging-tests
- `browser` is not an environment. Use a project with `browser.enabled`. https://vitest.dev/guide/environment
- Only `VITE_` variables from `.env` files load. Load all with `env: loadEnv(mode, process.cwd(), '')` in a function config. https://vitest.dev/guide/features
- The `env` option reaches tests only, not `globalSetup`. https://vitest.dev/config/env
- Vite does not read `tsconfig.json` by default, so imports relying on `baseUrl` fail with "module cannot be found". Add `vite-tsconfig-paths`. https://vitest.dev/guide/common-errors

## Pools, workers, isolation

- Default pool `forks`. `threads` can be faster but cannot `process.chdir()`, ignores `process.env.TZ`, and native modules (Prisma, bcrypt, canvas) may segfault. https://vitest.dev/config/pool
- The vm pools are faster but leak memory, are unstable with ESM, and give Node objects foreign prototypes so `toStrictEqual` fails. On large suites prefer `vmForks` to `vmThreads`. https://vitest.dev/config/pool
- `isolate: true` by default. Disable it per project, only for files that clean their state. https://vitest.dev/config/isolate , https://vitest.dev/guide/improving-performance
- `fileParallelism: false` runs files one at a time (forces `maxWorkers: 1`). Use it for files that share an exclusive resource such as a fixed port or a database, ideally in a separate project. https://vitest.dev/config/fileparallelism , https://vitest.dev/guide/recipes/parallel-sequential
- `maxWorkers` takes a number or `'50%'`. Default: all cores, half in watch mode. https://vitest.dev/config/maxworkers
- `VITEST_POOL_ID` identifies a worker, from 1 to the worker count. https://vitest.dev/guide/migration
- `vitest doctor` runs the suite under alternative configs and recommends one. https://vitest.dev/guide/cli

## Projects

- Use `test.projects`. `workspace` is deprecated. https://vitest.dev/guide/projects
- Entries: folder globs (`'packages/*'`), config files, or inline objects. Name inline projects, names must be unique. https://vitest.dev/guide/projects
- The root config is not a project unless listed. It holds global options (`reporters`, `coverage`). https://vitest.dev/guide/projects
- Inline projects inherit the root config (`extends: true` by default). Arrays such as `setupFiles` are concatenated. `name`, `projects` and the root `globalSetup` are never inherited. `extends: false` opts out. https://vitest.dev/guide/projects
- Projects referenced as files or folders inherit nothing. Share with `mergeConfig(sharedConfig, defineProject({...}))`. https://vitest.dev/guide/projects
- Do not `mergeConfig` the root config when it declares `projects`: the project becomes a container of nested projects. Merge a `vitest.shared` file instead. https://vitest.dev/guide/migration
- Use `defineProject` in project files. `coverage` and `reporters` are not supported there (root config only). https://vitest.dev/guide/projects
- `process.cwd()` stays where Vitest started, not the project root. https://vitest.dev/guide/projects
- Filter with `--project e2e`, `unit*` or `'!e2e'`. Debug with `DEBUG=vitest:projects vitest`. https://vitest.dev/guide/projects
- Run a sequential project after the parallel one with `sequence.groupOrder`. https://vitest.dev/guide/recipes/parallel-sequential

```ts
projects: [
  { test: { name: 'parallel', exclude: ['**.sequential.test.ts'], sequence: { groupOrder: 0 } } },
  { test: { name: 'sequential', include: ['**.sequential.test.ts'], fileParallelism: false, sequence: { groupOrder: 1 } } },
]
```
https://vitest.dev/guide/recipes/parallel-sequential

## CLI, filtering and watch

- `vitest` watches in a terminal and runs once in CI or non-TTY. Scripts, agents and hooks must use `vitest run`. lint-staged needs `vitest related --run`. https://vitest.dev/guide/cli , https://vitest.dev/guide/learn/writing-tests-with-ai
- Positional args are path substrings. `-t` is a regex on the full name joined by ` > ` (`-t 'math > adds'`). https://vitest.dev/guide/filtering
- Line filter: `vitest basic/foo.test.ts:10`, full file name required, no ranges. https://vitest.dev/guide/filtering
- `-t`, `--tags-filter`, `.only` and `.skip` apply after each file loads. Add a file filter in large repos. https://vitest.dev/guide/filtering
- Affected tests: `--changed origin/main` follows the import graph from Git. `--since-last-run` compares file modification times with its last recorded run and needs no Git. Run a full `vitest run` before finishing. https://vitest.dev/guide/recipes/affected-tests
- `--last-failed` reruns failures. `vitest related <files>` follows static imports only. https://vitest.dev/guide/cli
- Watch only follows imports. For files read at runtime use `watchTriggerPatterns`. https://vitest.dev/guide/recipes/watch-templates
- Defaults: `testTimeout` 5000 (15000 in browser mode), `hookTimeout` 10000 (30000 in browser mode), `slowTestThreshold` 300. `--silent=passed-only` keeps logs from failing tests only. https://vitest.dev/guide/cli , https://vitest.dev/config/testtimeout , https://vitest.dev/config/hooktimeout , https://vitest.dev/config/slowtestthreshold
- Bun: run `bun run test`, never `bun test` (Bun's own runner). https://vitest.dev/guide
- Add `.vitest/` to `.gitignore`: reports and artifacts land there. https://vitest.dev/guide

## Reporters

- Default `default`. Under a detected AI agent: `minimal` (alias `agent`, failures only). `github-actions` is added on GitHub Actions. https://vitest.dev/guide/reporters
- Setting `reporters` replaces the list. Keep defaults: `reporters: ['json', ...configDefaults.reporters]`. https://vitest.dev/guide/reporters
- `json`, `junit` and `html` write to `.vitest/` (`json/output.json`, `junit/output.xml`, `index.html`), not stdout. Set `outputFile` to relocate `json` and `junit` (`html` uses its `outputDir` option). https://vitest.dev/guide/reporters
- Sharded CI: `vitest run --reporter=blob --shard=1/3` per machine, then `vitest run --merge-reports --reporter=junit`. Shards split files, not tests. Set `VITEST_BLOB_LABEL` per OS. https://vitest.dev/guide/improving-performance , https://vitest.dev/guide/cli
