# Config file, env variables and modes

## Contents

- Config file loading
- Conditional and async config
- Env files are not loaded while the config runs
- `mergeConfig`
- `import.meta.env` in client code
- `.env` files and priority
- Modes versus `NODE_ENV`
- `define`
- `base` and `BASE_URL`
- Resolution options
- Typing env variables

## Config file loading

- Vite resolves `vite.config.{js,ts,mjs,...}` in the project root. `--config <file>` is relative to the cwd. With an alternative root (`vite serve sub/dir`), the config is looked up inside that root. https://vite.dev/config/ , https://vite.dev/guide/#specifying-alternative-root
- Default loader `bundle`: Rolldown bundles the config to a temp file in `node_modules/.vite-temp`. `--configLoader native` runs it directly (TypeScript needs Node 22.18+), `runner` is experimental. https://vite.dev/config/ , https://vite.dev/guide/cli
- `ERR_REQUIRE_ESM` / "This package is ESM only" from the config: make the config ESM with `"type": "module"` in the nearest `package.json`, or rename it `vite.config.mjs` / `.mts`. https://vite.dev/guide/troubleshooting#this-package-is-esm-only
- Debugging the config in VS Code: run `vite --configLoader native` so breakpoints map to source. With the bundle loader, add `**/node_modules/.vite-temp/**` to `resolveSourceMapLocations`. https://vite.dev/config/#debugging-the-config-file-in-vs-code

## Conditional and async config

```js
export default defineConfig(async ({ command, mode, isSsrBuild, isPreview }) => {
  if (command === 'serve') return { /* dev and preview */ }
  return { /* build */ }
})
```

- `command` is `'serve'` for `vite`, `vite dev`, `vite serve` and also for preview in `resolveConfig`. It is `'build'` for `vite build`. https://vite.dev/config/#conditional-config , https://vite.dev/guide/api-javascript#resolveconfig
- `isSsrBuild` and `isPreview` can be `undefined` in other tools: compare with `=== true`. https://vite.dev/config/#conditional-config
- Plugins that need the final config read it in `configResolved`, not from the user config. https://vite.dev/guide/api-plugin#configresolved

## Env files are not loaded while the config runs

`.env*` files are loaded after the config is evaluated, because which files to load depends on `root`, `envDir` and the final `mode`. `process.env.VITE_X` from a `.env` file is `undefined` inside `vite.config`. Use `loadEnv`: https://vite.dev/config/#using-environment-variables-in-config

```js
import { defineConfig, loadEnv } from 'vite'

export default defineConfig(({ mode }) => {
  const env = loadEnv(mode, process.cwd(), '') // '' = all vars, not only VITE_
  return {
    define: { __APP_ENV__: JSON.stringify(env.APP_ENV) },
    server: { port: env.APP_PORT ? Number(env.APP_PORT) : 5173 },
  }
})
```

- `loadEnv(mode, envDir, prefixes = 'VITE_')` merges `.env` files with matching vars already in `process.env`. https://vite.dev/guide/api-javascript#loadenv
- To read `NODE_ENV` in the config, set it on the command line, not in a `.env.[mode]` file. https://vite.dev/guide/env-and-mode#node-env-and-modes

## `mergeConfig`

- `mergeConfig(defaults, overrides, isRoot = true)` deep-merges. Pass `isRoot: false` when merging nested objects such as two `build` blocks. https://vite.dev/guide/api-javascript#mergeconfig
- `null` and `undefined` in `overrides` are skipped, so it cannot clear a default: edit the result instead. https://vite.dev/guide/api-javascript#mergeconfig
- It takes objects only. Call a function config first:

```js
export default defineConfig((env) => mergeConfig(configAsCallback(env), configAsObject))
```

## `import.meta.env` in client code

- Built-ins: `MODE`, `BASE_URL`, `PROD`, `DEV`, `SSR`. They are globals in dev and statically replaced in build, so `if (import.meta.env.DEV) {}` is tree-shaken. https://vite.dev/guide/env-and-mode#built-in-constants
- Only `VITE_`-prefixed variables reach client code, and always as strings. `VITE_PORT=123` gives `"123"`. Convert numbers and booleans yourself. https://vite.dev/guide/env-and-mode#env-variables
- `VITE_*` values are inlined in the bundle. Never put API keys or secrets there. Keep secrets on a backend or a serverless function. https://vite.dev/guide/env-and-mode#env-variables
- `envPrefix` changes the prefix. `envPrefix: ''` throws because it would expose every variable. Expose one unprefixed variable with `define` instead: https://vite.dev/config/shared-options#envprefix

```js
define: { 'import.meta.env.ENV_VARIABLE': JSON.stringify(process.env.ENV_VARIABLE) }
```

- In HTML, `%VITE_API_URL%` and `%MODE%` are replaced. Unknown names such as `%NON_EXISTENT%` stay as-is (in JS they become `undefined`). No conditionals: write a `transformIndexHtml` plugin. https://vite.dev/guide/env-and-mode#html-constant-replacement
- In library mode `import.meta.env.*` is replaced but `process.env.*` is not. Force it with `define: { 'process.env.NODE_ENV': '"production"' }` or use `esm-env`. https://vite.dev/guide/build#css-support

## `.env` files and priority

- Files in `envDir` (default `root`, `false` disables): `.env`, `.env.local`, `.env.[mode]`, `.env.[mode].local`. Mode files win over generic ones. https://vite.dev/guide/env-and-mode#env-files , https://vite.dev/config/shared-options#envdir
- Variables already set in the environment when Vite starts win over every `.env` file: `VITE_KEY=1 vite build`. https://vite.dev/guide/env-and-mode#env-files
- `.env` files load once at startup. Restart after editing. https://vite.dev/guide/env-and-mode#env-files
- Bun preloads `.env` into `process.env`, which then outranks Vite's mode files. https://vite.dev/guide/env-and-mode#env-files
- Git-ignore `*.local`. https://vite.dev/guide/env-and-mode#env-files
- Values are expanded by `dotenv-expand`. Escape a literal `$` as `\$`. Do not reference a variable defined later in the file: Vite accepts it but shells and `docker compose` do not. https://vite.dev/guide/env-and-mode#env-files

```
KEY=123
NEW_KEY2=test\$foo  # test$foo
NEW_KEY3=test$KEY   # test123
```

## Modes versus `NODE_ENV`

- Dev runs in mode `development`, `vite build` in `production`. `--mode staging` loads `.env.staging`. A `mode` in the config overrides the default, `--mode` overrides the config. https://vite.dev/guide/env-and-mode#modes , https://vite.dev/config/shared-options#mode
- Mode and `NODE_ENV` are independent. `vite build --mode development` keeps `NODE_ENV=production`. `NODE_ENV=development vite build` keeps mode `production`. https://vite.dev/guide/env-and-mode#node-env-and-modes
- `PROD`/`DEV` follow `NODE_ENV`, `MODE` follows `--mode`. For a development-flavored build, put `NODE_ENV=development` in `.env.testing` and run `vite build --mode testing`. https://vite.dev/guide/env-and-mode#node-env-and-modes
- `createServer` and `build` in the same Node process both read `process.env.NODE_ENV`. Set `process.env.NODE_ENV` or the `mode` of both to `development`, or run them in separate processes. https://vite.dev/guide/api-javascript#createserver

## `define`

- Implemented with Oxc define. Each value is a string holding JSON or a single identifier. Non-string values are `JSON.stringify`-ed for you. https://vite.dev/config/shared-options#define
- Each occurrence gets its own copy of an object value. https://vite.dev/guide/migration#advanced

## `base` and `BASE_URL`

- Deploying under `/my/app/`: set `base: '/my/app/'` (or `--base`). Imported asset URLs, CSS `url()` and HTML references are rewritten. https://vite.dev/guide/build#public-base-path
- For URLs built at runtime use `import.meta.env.BASE_URL`, written exactly like that: `import.meta.env['BASE_URL']` is not replaced. https://vite.dev/guide/build#public-base-path
- Unknown deploy path: `base: './'` (needs `import.meta` support). A full URL is accepted, its origin is ignored in dev. https://vite.dev/guide/build#relative-base , https://vite.dev/config/shared-options#base
- With a non-relative `base`, prefix every `server.proxy` key with it. https://vite.dev/config/server-options#server-proxy
- Different hosts for JS, assets and public files: `experimental.renderBuiltUrl(filename, { hostType, type, hostId })` returns a URL, `{ relative: true }` or `{ runtime: '...' }`. A `runtime` string is emitted as-is, so encode it yourself. https://vite.dev/guide/build#advanced-base-options

## Resolution options

- `resolve.alias`: first matching rule wins. Use absolute paths for file targets, relative ones are not resolved. RegExp `find` supports `$1`. https://vite.dev/config/shared-options#resolve-alias
- `resolve.tsconfigPaths: true` opts into tsconfig `paths` (off by default, has a cost). `paths` applies only to files in the tsconfig `include`, so add `"src/**/*.css"` for CSS imports. Not supported in `.less`. Prefer `resolve.alias` or `package.json` `imports`. https://vite.dev/config/shared-options#resolve-tsconfigpaths , https://vite.dev/guide/features#paths
- The `tsconfig` option is discouraged: it overrides per-file tsconfig discovery. https://vite.dev/config/shared-options#tsconfig
- `resolve.conditions` default `['module', 'browser', 'development|production']`, the last one following `NODE_ENV`. `resolve.mainFields` loses to `exports`. https://vite.dev/config/shared-options#resolve-conditions
- `resolve.extensions` default `['.mjs', '.js', '.mts', '.ts', '.jsx', '.tsx', '.json']`. Do not add `.vue` and similar: it hurts IDE and type support. https://vite.dev/config/shared-options#resolve-extensions

## Typing env variables

```ts
// src/vite-env.d.ts, no import statements in this file
interface ViteTypeOptions {
  strictImportMetaEnv: unknown // unknown keys become a type error
}
interface ImportMetaEnv {
  readonly VITE_APP_TITLE: string
}
interface ImportMeta {
  readonly env: ImportMetaEnv
}
```

An `import` in that file turns it into a module and the augmentation stops working. https://vite.dev/guide/env-and-mode#intellisense-for-typescript
