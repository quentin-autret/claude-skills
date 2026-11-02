# Migration from Vite 7 to Vite 8

## Contents

- Before upgrading
- Toolchain swap: Rolldown and Oxc
- Renamed and deprecated options
- Behavior changes that break code
- Advanced breaking changes
- Upcoming changes and the `future` option

## Before upgrading

- Coming from Vite 6: apply the Vite 7 guide (https://v7.vite.dev/guide/migration) first, then this one. https://vite.dev/guide/migration#migration-from-v6
- Gradual path: `rolldown-vite` is Vite 7 plus Rolldown and nothing else. Use it as an intermediate step, then undo the dependency change and set `vite` to `^8.0.0`. https://vite.dev/guide/migration#gradual-migration

```json
"vite": "^8.0.0"
```

replacing `"vite": "npm:rolldown-vite@7.2.2"`.

- Node.js 20.19+ or 22.12+ is required. https://vite.dev/guide/

## Toolchain swap: Rolldown and Oxc

Vite 8 uses Rolldown and Oxc based tools instead of esbuild and Rollup: Rolldown for dependency optimization, Oxc for JS transforms and JS minification. esbuild is no longer used directly. A compatibility layer preserves existing configurations. https://vite.dev/guide/migration#rolldown , https://vite.dev/guide/why#a-unified-toolchain

- Default `build.target` (`'baseline-widely-available'`) moved up: Chrome and Edge 111, Firefox 114, Safari 16.4. https://vite.dev/guide/migration#default-browser-target-change
- JS minifier is Oxc, CSS minifier is Lightning CSS. Lightning CSS lowers syntax better, so CSS can grow slightly. https://vite.dev/guide/migration#javascript-minification-by-oxc , https://vite.dev/guide/migration#css-minification-by-lightning-css
- `esbuild` is now an optional dependency. Anything that still uses it (`build.minify: 'esbuild'`, `build.cssMinify: 'esbuild'`, `transformWithEsbuild`) needs `esbuild` as a devDependency. https://vite.dev/guide/migration#esbuild-fallbacks

## Renamed and deprecated options

All deprecated options still work through auto-conversion but will be removed. Migrate now.

| Old | Current |
| :-- | :-- |
| `build.rollupOptions` | `build.rolldownOptions` |
| `worker.rollupOptions` | `worker.rolldownOptions` |
| `esbuild` option | `oxc` option |
| `esbuild.jsx: 'automatic'`, `jsxImportSource` | `oxc.jsx: { runtime: 'automatic', importSource }` |
| `esbuild.jsx: 'transform'`, `jsxFactory`, `jsxFragment` | `oxc.jsx: { runtime: 'classic', pragma, pragmaFrag }` |
| `esbuild.jsxDev`, `jsxSideEffects` | `oxc.jsx.development`, `oxc.jsx.pure` |
| `esbuild.jsxInject`, `include`, `exclude`, `define` | `oxc.*` with the same names |
| `esbuild.banner` / `footer` | A small plugin with a `transform` hook |
| `esbuild.minify*` | `build.rolldownOptions.output.minify` |
| `esbuild.drop` | `build.rolldownOptions.output.minify.compress.drop*` |
| `optimizeDeps.esbuildOptions` | `optimizeDeps.rolldownOptions` |
| `build.minify: 'esbuild'` | Default `'oxc'` |
| `transformWithEsbuild` | `transformWithOxc` |
| `output.manualChunks` (object form removed, function deprecated) | `build.rolldownOptions.output.codeSplitting` |
| `build.rollupOptions.watch.chokidar` (removed) | `build.rolldownOptions.watch.watcher` |
| `build.commonjsOptions`, `build.dynamicImportVarsOptions.warnOnError` | No-ops, delete them |
| `resolve.alias[].customResolver` | Plugin with `resolveId` and `enforce: 'pre'` |
| `build.polyfillModulePreload` | `build.modulePreload.polyfill` |
| `server.hmr.{protocol,host,port,path,clientPort,timeout,server}` | `server.ws.*` |
| `parseAst` / `parseAstAsync` | `parseSync` / `parse` |
| `import.meta.hot.accept(url)` (removed) | `import.meta.hot.accept(id)` |

Sources: https://vite.dev/guide/migration#other-related-deprecations , https://vite.dev/guide/migration#javascript-transforms-by-oxc , https://vite.dev/guide/migration#dependency-optimizer-now-uses-rolldown , https://vite.dev/config/build-options#build-polyfillmodulepreload , https://vite.dev/config/server-options#server-hmr , https://vite.dev/guide/migration#removed-deprecated-features

esbuild optimizer options map to Rolldown ones: `treeShaking` to `treeshake`, `define` to `transform.define`, `loader` to `moduleTypes`, `preserveSymlinks` to `!resolve.symlinks`, `conditions` to `resolve.conditionNames`, `keepNames` to `output.keepNames`. Print the result to check it:

```js
configResolved(config) { console.log(config.optimizeDeps.rolldownOptions) }
```

https://vite.dev/guide/migration#dependency-optimizer-now-uses-rolldown

### Not supported by Oxc

- `esbuild.supported` has no equivalent. https://vite.dev/guide/migration#javascript-transforms-by-oxc
- Lowering native decorators. Run Babel (`@rolldown/plugin-babel` with `@babel/plugin-proposal-decorators`) or SWC (`@rollup/plugin-swc`) only on files that contain `@`:

```js
import { defineConfig, withFilter } from 'vite'
import swc from '@rollup/plugin-swc'

export default defineConfig({
  plugins: [
    withFilter(
      swc({ swc: { jsc: { parser: { decorators: true, decoratorsBeforeExport: true },
        transform: { decoratorVersion: '2023-11' } } } }),
      { transform: { code: '@' } },
    ),
  ],
})
```

https://vite.dev/guide/migration#javascript-transforms-by-oxc

- Property mangling (`mangleProps`, `reserveProps`, `mangleQuoted`, `mangleCache`). https://vite.dev/guide/migration#javascript-minification-by-oxc
- If minified output breaks, the esbuild and Oxc minifiers make different assumptions about the source: compare them and report. https://vite.dev/guide/migration#javascript-minification-by-oxc

## Behavior changes that break code

- **CommonJS default import is now consistent across dev and build.** `import x from 'cjs'` gives `module.exports` when the importer is `.mjs`/`.mts`, or its nearest `package.json` has `"type": "module"`, or the importee does not set `module.exports.__esModule = true`. Otherwise it gives `module.exports.default`. Symptoms: "Element type is invalid ... got: object", "foo is not a function". Temporary escape hatch: `legacy.inconsistentCjsInterop: true` (deprecated). Report the package upstream. https://vite.dev/guide/migration#consistent-commonjs-interop , https://vite.dev/guide/troubleshooting#default-import-unexpectedly-returns-an-object
- **No more format sniffing.** With both `browser` and `module` fields, Vite follows `resolve.mainFields` order and no longer picks the ESM file by content. Fix with `resolve.alias` or a package patch. https://vite.dev/guide/migration#removed-module-resolution-using-format-sniffing
- **`require()` of externalized modules is kept**, not rewritten to `import`. To convert:

```js
import { defineConfig, esmExternalRequirePlugin } from 'vite'
export default defineConfig({
  plugins: [esmExternalRequirePlugin({ external: ['react', 'vue', /^node:/] })],
})
```

https://vite.dev/guide/migration#require-calls-for-externalized-modules

- **`import.meta.url` in UMD/IIFE output is `undefined`** (no polyfill). Restore it with `define` plus `build.rolldownOptions.output.intro`. https://vite.dev/guide/migration#import-meta-url-in-umd-iife
- **`build()` throws `BundleError`**: read `e.errors` (array of Rolldown errors with `code`). https://vite.dev/guide/migration#build-throws-bundleerror
- **Plugins that turn non-JS into JS must return `moduleType: 'js'`**, because Rolldown infers the module type from the id extension. https://vite.dev/guide/migration#module-type-support-and-auto-detection

```js
return { code: `export default ${JSON.stringify(content)}`, moduleType: 'js' }
```

## Advanced breaking changes

These affect a minority of projects. https://vite.dev/guide/migration#advanced

- Extglobs are unsupported. TypeScript legacy `namespace` is partially supported.
- `define` gives each use its own copy of an object, references are not shared.
- In `generateBundle`/`writeBundle`, assigning `bundle[name]` is unsupported (use `this.emitFile()`), the `bundle` object is not shared across hooks, and `structuredClone(bundle)` throws (use `structuredClone({ ...bundle })`).
- Rollup "parallel" hooks run sequentially. `"use strict"` is not always injected. Comments are stripped before `renderChunk`.
- `@vitejs/plugin-legacy` cannot target ES5 or lower.
- A `build.target` that lists the same browser twice now errors.
- Removed: `output.format: 'system'` and `'amd'`, hooks `shouldTransformCachedModule`, `resolveImportMeta`, `renderDynamicImport`, `resolveFileUrl`.

## Upcoming changes and the `future` option

- The `future` config option opts into upcoming breaking changes early. Its type is `Record<string, 'warn' | undefined>`, and setting a flag to `'warn'` identifies your usage. https://vite.dev/config/shared-options#future , https://vite.dev/changes/ , https://vite.dev/changes/per-environment-apis
- Planned for the next major: `this.environment` in hooks, the `hotUpdate` hook, SSR through `ModuleRunner`. Still under consideration: per-environment server APIs, shared plugins during build. https://vite.dev/changes/
- Flags: `removePluginHookSsrArgument`, `removePluginHookHandleHotUpdate`, `removeSsrLoadModule`, `removeServerModuleGraph`, `removeServerReloadModule`, `removeServerPluginContainer`, `removeServerHot`, `removeServerTransformRequest`, `removeServerWarmupRequest`. The pages only ask you to set a flag to `"warn"` to detect usages. `removePluginHookSsrArgument`: https://vite.dev/changes/this-environment-in-hooks . `removePluginHookHandleHotUpdate`: https://vite.dev/changes/hotupdate-hook . `removeSsrLoadModule`: https://vite.dev/changes/ssr-using-modulerunner . The `removeServer*` flags: https://vite.dev/changes/per-environment-apis

```js
export default defineConfig({
  future: { removeSsrLoadModule: 'warn', removePluginHookHandleHotUpdate: 'warn' },
})
```
