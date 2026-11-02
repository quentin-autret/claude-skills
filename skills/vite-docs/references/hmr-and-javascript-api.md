# HMR client API and JavaScript API

## Contents

- `import.meta.hot` rules
- accept, dispose, prune, data
- invalidate and decline
- HMR events
- Custom events between a plugin and the client
- `createServer` and `ViteDevServer`
- `build`, `preview`, `createBuilder`
- Config helpers and version exports
- Old patterns

## `import.meta.hot` rules

The manual HMR API is for framework and tool authors. Apps get HMR from framework plugins. https://vite.dev/guide/api-hmr

- Wrap all HMR code in `if (import.meta.hot)` so it is removed from production. https://vite.dev/guide/api-hmr#required-conditional-guard
- Write the call literally as `import.meta.hot.accept(`. Vite detects it by static analysis, whitespace-sensitive. Aliasing `import.meta.hot` to a variable or reformatting the call breaks HMR for the module. https://vite.dev/guide/api-hmr#hot-accept-cb
- Types: `"types": ["vite/client"]` in `tsconfig.json`. https://vite.dev/guide/api-hmr#intellisense-for-typescript

## accept, dispose, prune, data

- A module that accepts updates is an HMR boundary. Its importers are not notified. https://vite.dev/guide/api-hmr#hot-accept-cb
- Vite does not swap the module that importers already hold, and creates no proxy. A boundary that re-exports from a dependency must update those exports itself, declared with `let`. https://vite.dev/guide/api-hmr#hot-accept-cb
- Self-accept. `newModule` is `undefined` after a syntax error: https://vite.dev/guide/api-hmr#hot-accept-cb

```js
export const count = 1
if (import.meta.hot) {
  import.meta.hot.accept((newModule) => {
    if (newModule) console.log('count is now', newModule.count)
  })
}
```

- Accept direct dependencies without reloading the current module. With an array, only the updated entry is non-null, and the array is empty on failure: https://vite.dev/guide/api-hmr#hot-accept-deps-cb

```js
import { foo } from './foo.js'
if (import.meta.hot) {
  import.meta.hot.accept('./foo.js', (newFoo) => newFoo?.foo())
  import.meta.hot.accept(['./foo.js', './bar.js'], ([newFoo, newBar]) => {})
}
```

- `hot.dispose(cb)` cleans side effects of the old copy (timers, listeners). https://vite.dev/guide/api-hmr#hot-dispose-cb
- `hot.prune(cb)` runs when no page imports the module any more. Use it instead of `dispose` when updates already clean up after themselves. https://vite.dev/guide/api-hmr#hot-prune-cb
- `hot.data` persists per module across updates. Mutate its properties. Re-assigning `import.meta.hot.data = {}` is not supported. After a prune, the data is cleared. https://vite.dev/guide/api-hmr#hot-data

```js
import.meta.hot.data.state = 'hello' // ok
```

## invalidate and decline

- A self-accepting module that cannot apply an update calls `invalidate()` inside the `accept` callback, which propagates the update to importers. Always call `accept` first, or the client stops listening to later changes. https://vite.dev/guide/api-hmr#hot-invalidate-message-string

```js
import.meta.hot.accept((mod) => {
  if (cannotHandleUpdate(mod)) import.meta.hot.invalidate('reason shown in console')
})
```

- `hot.decline()` is a no-op kept for compatibility. Use `invalidate()`. https://vite.dev/guide/api-hmr#hot-decline

## HMR events

- `hot.on` / `hot.off`: `vite:beforeUpdate`, `vite:afterUpdate`, `vite:beforeFullReload`, `vite:beforePrune`, `vite:invalidate`, `vite:error`, `vite:ws:disconnect`, `vite:ws:connect`, plus custom plugin events. https://vite.dev/guide/api-hmr#hot-on-event-cb
- `hot.send(event, data)` sends to the dev server. Messages sent before the connection opens are buffered. https://vite.dev/guide/api-hmr#hot-send-event-data
- In production, `vite:preloadError` on `window` reports failed dynamic imports. https://vite.dev/guide/build#load-error-handling

## Custom events between a plugin and the client

```js
// plugin
configureServer(server) {
  server.ws.on('my:from-client', (data, client) => client.send('my:ack', { ok: true }))
}
// client
if (import.meta.hot) {
  import.meta.hot.on('my:ack', (data) => {})
  import.meta.hot.send('my:from-client', { msg: 'hi' })
}
```

- Prefix event names with the plugin name. https://vite.dev/guide/api-plugin#server-to-client
- Type events by augmenting `CustomEventMap` in `'vite/types/customEvent.d.ts'`, with the `.d.ts` in the specifier. https://vite.dev/guide/api-plugin#typescript-for-custom-events

```ts
import 'vite/types/customEvent.d.ts'
declare module 'vite/types/customEvent.d.ts' {
  interface CustomEventMap { 'my:ack': { ok: boolean } }
}
```

## `createServer` and `ViteDevServer`

```ts
import { createServer } from 'vite'

const server = await createServer({
  configFile: false, // skip vite.config resolution
  root: import.meta.dirname,
  server: { port: 1337 },
})
await server.listen()
server.printUrls()
server.bindCLIShortcuts({ print: true })
```

- Takes any user config plus `mode` and `configFile`. https://vite.dev/guide/api-javascript#createserver
- `createServer` and `build` in one process share `process.env.NODE_ENV`. Set `process.env.NODE_ENV` or the `mode` of both to `development`, or use child processes. https://vite.dev/guide/api-javascript#createserver
- Members: `middlewares` (connect app), `httpServer` and `resolvedUrls` (`null` in middleware mode), `watcher`, `ws`, `environments`, `transformIndexHtml`, `reloadModule` (no-op when `hmr` is false), `restart(forceOptimize?)`, `close`, `waitForRequestsIdle`. `environments` is documented on https://vite.dev/guide/api-environment-instances#accessing-the-environments, the rest on https://vite.dev/guide/api-javascript#vitedevserver
- `waitForRequestsIdle(ignoredId?)` is an experimental escape hatch. From a `load` or `transform` hook, pass the current module id or it deadlocks. It holds 1 of the 6 HTTP/1 connections until static imports are processed. https://vite.dev/guide/api-javascript#vitedevserver
- Per-environment work goes through `server.environments.<name>`: `transformRequest(url)`, `warmupRequest(url)`. `fetchModule` is called by the module runner and is not meant to be called manually. https://vite.dev/guide/api-environment-instances#devenvironment-class

## `build`, `preview`, `createBuilder`

```ts
import { build, preview } from 'vite'

try {
  await build({ root: './project', base: '/foo/', build: { rolldownOptions: {} } })
} catch (e) {
  for (const error of e.errors ?? []) console.log(error.code) // BundleError
}

const previewServer = await preview({ preview: { port: 8080, open: true } })
previewServer.printUrls()
```

- `build()` resolves to `RolldownOutput`, an array of them, or a `RolldownWatcher`. It throws a `BundleError` with an `errors` array. https://vite.dev/guide/api-javascript#build , https://vite.dev/guide/migration#build-throws-bundleerror
- `build()` builds client or SSR only. To build every environment, `const builder = await createBuilder(); await builder.buildApp()` (same as `vite build --app`). https://vite.dev/guide/api-environment-frameworks#building-programmatically-with-createbuilder

## Config helpers and version exports

- `resolveConfig(inlineConfig, command, defaultMode, defaultNodeEnv, isPreview)`: `command` is `'serve'` for dev and preview. https://vite.dev/guide/api-javascript#resolveconfig
- `mergeConfig(a, b, isRoot)`: pass `isRoot: false` for nested objects. It cannot unset values and accepts objects only. https://vite.dev/guide/api-javascript#mergeconfig
- `loadEnv(mode, envDir, prefixes)`: loads `.env` files and merges matching `process.env` variables. By default only `VITE_` variables load. Passing `''` as `prefixes` loads all. https://vite.dev/guide/api-javascript#loadenv , https://vite.dev/config/
- `loadConfigFromFile(configEnv, file?, root?)` loads a config with Rolldown and returns `{ path, config, dependencies }` or `null`. https://vite.dev/guide/api-javascript#loadconfigfromfile
- `searchForWorkspaceRoot(cwd)` finds the monorepo root, for `server.fs.allow`. https://vite.dev/guide/api-javascript#searchforworkspaceroot
- `transformWithOxc(code, filename, options?, inMap?)` returns warnings but no `errors`. `preprocessCSS` (experimental) compiles preprocessor files but does not resolve `url()` or `image-set()`. https://vite.dev/guide/api-javascript#transformwithoxc , https://vite.dev/guide/api-javascript#preprocesscss
- `version` and `rolldownVersion` are exported. `esbuildVersion` and `rollupVersion` exist only for backward compatibility. https://vite.dev/guide/api-javascript#version

## Old patterns

| Old | Current |
| :-- | :-- |
| `import.meta.hot.decline()` | `import.meta.hot.invalidate()` inside `accept` |
| `import.meta.hot.data = { ... }` | `import.meta.hot.data.key = value` |
| `transformWithEsbuild` | `transformWithOxc` |
| `RollupOutput` from `build()` | `RolldownOutput` |
| `server.moduleGraph`, `server.transformRequest(url, ssr)` | `environment.moduleGraph`, `environment.transformRequest(url)` |

The server-level methods still work. The docs do not recommend moving yet, only detecting them with the `future.removeServer*` flags. https://vite.dev/changes/per-environment-apis , https://vite.dev/guide/migration#removed-deprecated-features
