# Using and writing plugins

## Contents

- Using plugins
- Plugin shape and naming
- Hook filters
- Virtual modules
- Hooks in dev versus build
- Vite-specific hooks
- Ordering and conditional application
- Rolldown and Rollup compatibility
- Assets, bundle metadata, paths
- Old patterns

## Using plugins

- In `plugins`, falsy entries are ignored, nested arrays (presets) are flattened, promises are awaited. https://vite.dev/config/shared-options#plugins
- Much of what needs a plugin with Rollup is built in. Registry: https://registry.vite.dev/plugins . https://vite.dev/guide/using-plugins#finding-plugins
- Adjust a third-party plugin by spreading it and adding Vite-only properties: https://vite.dev/guide/using-plugins#enforcing-plugin-ordering , https://vite.dev/guide/api-plugin#rolldown-plugin-compatibility

```js
import image from '@rollup/plugin-image'
export default defineConfig({ plugins: [{ ...image(), enforce: 'pre', apply: 'build' }] })
```

- Plugins in a `config` hook are too late: user plugins are resolved before it runs, so injecting plugins there has no effect. https://vite.dev/guide/api-plugin#config

## Plugin shape and naming

- Vite plugins extend the Rolldown plugin interface and can live inline in `vite.config`. `name` is required. https://vite.dev/guide/api-plugin#authoring-a-plugin , https://vite.dev/guide/api-plugin#importing-a-virtual-file
- Package names: `rolldown-plugin-*` (keywords `rolldown-plugin` and `vite-plugin`) without Vite-only hooks, `vite-plugin-*` otherwise. https://vite.dev/guide/api-plugin#conventions

## Hook filters

Use the object form `{ filter, handler }`. Hook filters reduce the communication overhead between the Rust and JavaScript runtimes by avoiding unnecessary hook invocations. https://vite.dev/guide/api-plugin#hook-filters

```js
export default function myPlugin() {
  return {
    name: 'transform-my-ext',
    transform: {
      filter: { id: /\.my-ext$/ },
      handler(code, id) {
        return { code: compileToJs(code), map: null, moduleType: 'js' }
      },
    },
  }
}
```

- Filters are supported by Vite 6.3.0+ and Rollup 4.38.0+. A plugin that supports older versions repeats the check in the handler. https://vite.dev/guide/api-plugin#hook-filters
- `exactRegex` and `prefixRegex` come from `@rolldown/pluginutils` (also `rolldown/filter`). For user include/exclude options, `createFilter` from `@rollup/pluginutils`. https://vite.dev/guide/api-plugin#hook-filters , https://vite.dev/guide/api-plugin#filtering-include-exclude-pattern
- `withFilter(plugin, { transform: { code: '@' } })` from `vite` adds a filter to an existing plugin. https://vite.dev/guide/migration#javascript-transforms-by-oxc
- Return `moduleType: 'js'` when turning non-JS content into JS: Rolldown infers the type from the extension. https://vite.dev/guide/migration#module-type-support-and-auto-detection

## Virtual modules

```js
import { exactRegex } from '@rolldown/pluginutils'

export default function myPlugin() {
  const id = 'virtual:my-module'
  const resolvedId = '\0' + id
  return {
    name: 'my-plugin',
    resolveId: { filter: { id: exactRegex(id) }, handler: () => resolvedId },
    load: { filter: { id: exactRegex(resolvedId) }, handler: () => `export const msg = "hi"` },
  }
}
```

- Prefix the resolved id with `\0`. In dev the browser sees `/@id/__x00__virtual:my-module`. Hooks get the decoded id. https://vite.dev/guide/api-plugin#importing-a-virtual-file

## Hooks in dev versus build

- In dev, `options` and `buildStart` run once at server start, `resolveId`, `load`, `transform` per request, `buildEnd` and `closeBundle` at close. https://vite.dev/guide/api-plugin#rolldown-hooks
- `moduleParsed` never runs in dev. Output generation hooks run only in build, except `closeBundle`. https://vite.dev/guide/api-plugin#rolldown-hooks
- `buildStart`, `buildEnd` and `watchChange` run for the client environment only in dev, unless `perEnvironmentStartEndDuringDev: true` / `perEnvironmentWatchChangeDuringDev: true`. https://vite.dev/guide/api-environment-plugins#per-environment-state-in-plugins
- Detect Rolldown-powered Vite with `this.meta.rolldownVersion` (absent on Rollup-powered Vite). `this.meta.viteVersion` gives the Vite version. https://vite.dev/guide/api-plugin#plugin-context-meta

## Vite-specific hooks

- `config(config, { mode, command, isSsrBuild, isPreview })`: return a partial config to deep-merge. Mutate only when merging cannot express it. https://vite.dev/guide/api-plugin#config
- `configResolved(resolved)`: store the final config for other hooks. `command` is `'serve'` in dev. https://vite.dev/guide/api-plugin#configresolved
- `configureServer(server)`: middlewares added directly run before Vite's internals. Return a function to add them after. It never runs in build, so guard any stored `server`. https://vite.dev/guide/api-plugin#configureserver

```js
configureServer(server) {
  return () => {
    server.middlewares.use((req, res, next) => { /* after Vite's middlewares */ next() })
  }
}
```

- `configurePreviewServer(server)`: same, for `vite preview`. https://vite.dev/guide/api-plugin#configurepreviewserver
- `closeServer({ reason })` (`'restart'` or `'close'`) disposes what `configureServer` created. `closePreviewServer()` has no reason. https://vite.dev/guide/api-plugin#closeserver
- `transformIndexHtml(html, ctx)`: return a string, tag descriptors `{ tag, attrs, children, injectTo }` (default `injectTo: 'head-prepend'`), or `{ html, tags }`. `order: 'pre'` runs before HTML processing, so an injected script goes through the plugin pipeline. Not called when a framework handles entry files itself. https://vite.dev/guide/api-plugin#transformindexhtml
- `hotUpdate({ type, file, timestamp, modules, read, server })`: per environment, also on `create` and `delete`. Use `read()`, not `fs.readFile`, because the event can fire before the editor finishes writing. Return a narrowed module list, or `[]` to take over: https://vite.dev/guide/api-environment-plugins#the-hotupdate-hook

```js
hotUpdate({ modules, timestamp }) {
  if (this.environment.name !== 'client') return
  const invalidated = new Set()
  for (const mod of modules) {
    this.environment.moduleGraph.invalidateModule(mod, invalidated, timestamp, true)
  }
  this.environment.hot.send({ type: 'full-reload' })
  return []
}
```

- Or return `[]` and send a custom event, handled by `import.meta.hot.on('special-update', cb)` in the client. https://vite.dev/guide/api-plugin#handlehotupdate

## Ordering and conditional application

- Order: alias, user `enforce: 'pre'`, Vite core, user plugins without `enforce`, Vite build plugins, user `enforce: 'post'`, Vite post-build (minify, manifest, reporting). A hook-level `order` still applies inside that. https://vite.dev/guide/api-plugin#plugin-ordering
- `apply: 'build' | 'serve'`, or a function: https://vite.dev/guide/api-plugin#conditional-application

```js
apply(config, { command }) { return command === 'build' && !config.build.ssr }
```

- Per environment: `applyToEnvironment(env)` returns `true`, `false` or a replacement plugin. It runs after `configResolved`. https://vite.dev/guide/api-environment-plugins#per-environment-plugins-using-the-applytoenvironment-hook
- One plugin instance serves every environment: key state by `this.environment` (for example a `Map<Environment, State>` or a `WeakMap`). https://vite.dev/guide/api-environment-plugins#per-environment-state-in-plugins

## Rolldown and Rollup compatibility

- A Rollup or Rolldown plugin works in Vite if it does not use `moduleParsed`, does not rely on Rolldown-only options such as `transform.inject`, and does not couple bundle-phase and output-phase hooks. https://vite.dev/guide/api-plugin#rolldown-plugin-compatibility
- Build-only plugins can sit in `build.rolldownOptions.plugins` (like `enforce: 'post'` plus `apply: 'build'`). https://vite.dev/guide/api-plugin#rolldown-plugin-compatibility
- In `generateBundle`, emit with `this.emitFile()` rather than assigning `bundle[name]`, and clone with `structuredClone({ ...bundle })`. https://vite.dev/guide/migration#advanced

## Assets, bundle metadata, paths

- `this.emitFile({ type: 'asset', name, source })` returns a `referenceId`. In JS, reference it as `import.meta.ROLLDOWN_FILE_URL_<referenceId>`. In CSS or HTML, use the token `__VITE_ASSET__<referenceId>__`, with any query or hash right after it. https://vite.dev/guide/api-plugin#referencing-emitted-assets

```js
return `export default import.meta.ROLLDOWN_FILE_URL_${referenceId} + '#frag'`
```

- Chunks and assets carry `viteMetadata.importedCss` and `importedAssets` (Sets) in `renderChunk`, `generateBundle`, `writeBundle`. https://vite.dev/guide/api-plugin#output-bundle-metadata
- Resolved ids use POSIX separators, even on Windows. Compare with `normalizePath` from `vite` (`normalizePath('foo\\bar')` gives `'foo/bar'`). https://vite.dev/guide/api-plugin#path-normalization

## Old patterns

| Old | Current |
| :-- | :-- |
| `handleHotUpdate({ server, modules })` + `server.ws.send` | `hotUpdate({ modules })` + `this.environment.hot.send` |
| `options.ssr` in hooks | `this.environment.config.consumer === 'server'` |
| `server.moduleGraph` cached from `configureServer` | `this.environment.moduleGraph` |
| `transformWithEsbuild` (deprecated) | `transformWithOxc` |
| `resolve.alias[].customResolver` | Plugin with `resolveId` and `enforce: 'pre'` |

Sources: https://vite.dev/guide/migration , https://vite.dev/guide/api-javascript

`handleHotUpdate` and `options.ssr` still work. Their deprecation is planned for a future major, and the docs do not recommend moving away yet. To detect usage, set `future.removePluginHookHandleHotUpdate: 'warn'` and `future.removePluginHookSsrArgument: 'warn'`. https://vite.dev/changes/hotupdate-hook , https://vite.dev/changes/this-environment-in-hooks
