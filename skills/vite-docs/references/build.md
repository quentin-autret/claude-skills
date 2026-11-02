# Production build and library mode

## Contents

- Browser target and polyfills
- Bundler options and entries
- Minification and output defaults
- Chunks and stale deploys
- Module preload
- Library mode
- Content Security Policy

## Browser target and polyfills

- Default `build.target` is `'baseline-widely-available'`: `['chrome111', 'edge111', 'firefox114', 'safari16.4', 'ios16.4']`. It must be a valid Oxc target. The lowest is `es2015`, and native dynamic import plus `import.meta` stay required. https://vite.dev/config/build-options#build-target , https://vite.dev/guide/build#browser-compatibility
- Vite only lowers syntax, it adds no polyfills. For old browsers use `@vitejs/plugin-legacy` (legacy chunks with polyfills, loaded only where native ESM is missing). It cannot target ES5 or lower. https://vite.dev/guide/build#browser-compatibility , https://vite.dev/guide/migration#advanced
- The build warns when Oxc cannot safely transpile a feature for the target. https://vite.dev/config/build-options#build-target
- `build.cssTarget` defaults to `build.target`. Set it only for unusual browsers (Android WeChat WebView needs `chrome61` to keep `rgba()`). https://vite.dev/config/build-options#build-csstarget

## Bundler options and entries

- Rolldown options live in `build.rolldownOptions` (reference: https://rolldown.rs/reference/). `build.rollupOptions` is a deprecated alias. https://vite.dev/guide/build#customizing-the-build , https://vite.dev/config/build-options#build-rollupoptions
- Prefer the top-level `input` option for entries: it also feeds dev, `build.lib.entry`, `build.ssr` and `optimizeDeps.entries`. `build.rolldownOptions.input`, when set, overrides it for the build only. https://vite.dev/config/shared-options#input , https://vite.dev/config/build-options#build-rolldownoptions

```js
import { resolve } from 'node:path'
export default defineConfig({
  input: {
    main: resolve(import.meta.dirname, 'index.html'),
    nested: resolve(import.meta.dirname, 'nested/index.html'),
  },
})
```

- Multi-page apps: for HTML entries, Vite ignores the key and writes each HTML file at its path relative to the root, as in dev. With a custom `root`, `import.meta.dirname` is still the config folder, so include `root` in `resolve()`. https://vite.dev/guide/build#multi-page-app
- Vite sets Rolldown `preserveEntrySignatures` to `false` for client builds, `'strict'` for libraries and `'allow-extension'` for SSR. https://vite.dev/config/build-options#build-rolldownoptions
- Build-only Rollup or Rolldown plugins can go in `build.rolldownOptions.plugins`. https://vite.dev/guide/api-plugin#rolldown-plugin-compatibility
- Rebuild on change: `vite build --watch` or `build.watch: {}` (Rolldown watcher). https://vite.dev/guide/build#rebuild-on-files-changes
- Programmatic post-processing: `build.write: false` and use the returned output. https://vite.dev/config/build-options#build-write

## Minification and output defaults

- `build.minify` default `'oxc'` for client builds, `false` for SSR builds. `'terser'` needs `terser` installed. `'esbuild'` is deprecated and needs `esbuild` installed. https://vite.dev/config/build-options#build-minify
- `build.cssMinify` default `'lightningcss'`, and `false` when `build.minify` is off. https://vite.dev/config/build-options#build-cssminify
- In library mode with the `es` format, whitespace is not minified, to keep pure annotations for tree-shaking. https://vite.dev/config/build-options#build-minify
- `build.emptyOutDir` is `true` only when `outDir` is inside `root`. Outside, Vite warns and keeps files: set it explicitly. https://vite.dev/config/build-options#build-emptyoutdir
- `build.sourcemap` default `false`. `'hidden'` writes maps without the comment. https://vite.dev/config/build-options#build-sourcemap
- `build.chunkSizeWarningLimit` is 500 kB of uncompressed size. https://vite.dev/config/build-options#build-chunksizewarninglimit
- Applied automatically: CSS code splitting (`build.cssCodeSplit: false` for one file), `<link rel="modulepreload">`, parallel loading of shared async chunks. https://vite.dev/guide/features#build-optimizations
- `build.license: true` writes `.vite/license.md` listing bundled dependency licenses. `{ fileName: 'x.json' }` writes JSON. https://vite.dev/config/build-options#build-license

## Chunks and stale deploys

- Manual chunking: `build.rolldownOptions.output.codeSplitting`. `manualChunks` is removed (object) or deprecated (function). https://vite.dev/guide/build#chunking-strategy , https://vite.dev/guide/migration#removed-object-form-build-rollupoptions-output-manualchunks-and-deprecate-function-form-one
- After a deploy, an open page can import a chunk that no longer exists. Serve the HTML with `Cache-Control: no-cache`, and reload on `vite:preloadError`: https://vite.dev/guide/build#load-error-handling

```js
window.addEventListener('vite:preloadError', (event) => {
  window.location.reload() // event.preventDefault() stops the error from being thrown
})
```

- Other fixes: keep old chunks for a while, or a service worker that caches assets. Check the framework's own solution first. Poor networks cannot be fixed, browsers do not retry dynamic imports. https://vite.dev/guide/troubleshooting#failed-to-fetch-dynamically-imported-module-error
- Ad blockers block chunk names containing words like `ad` or `track`: rename with `build.rolldownOptions.output.chunkFileNames`. https://vite.dev/guide/troubleshooting#failed-to-fetch-dynamically-imported-module-error
- `build.chunkImportMap: true` (experimental) references chunks through an import map so one change does not rehash every importer. It needs `import.meta.resolve` support and does not cover CSS or assets. https://vite.dev/config/build-options#build-chunkimportmap , https://vite.dev/guide/features#chunk-import-map-optimization

## Module preload

- With a non-HTML entry, import the polyfill at the top of the entry (not needed in library mode): https://vite.dev/config/build-options#build-modulepreload

```js
import 'vite/modulepreload-polyfill'
```

- `build.modulePreload.polyfill` replaces the deprecated `build.polyfillModulePreload`. https://vite.dev/config/build-options#build-polyfillmodulepreload
- `modulePreload.resolveDependencies(filename, deps, { hostId, hostType })` (experimental) filters preloaded deps, paths relative to `outDir`. https://vite.dev/config/build-options#build-modulepreload

## Library mode

```js
import { resolve } from 'node:path'
export default defineConfig({
  build: {
    lib: { entry: resolve(import.meta.dirname, 'lib/main.js'), name: 'MyLib', fileName: 'my-lib' },
    rolldownOptions: { external: ['vue'], output: { globals: { vue: 'Vue' } } },
  },
})
```

- Default formats: `['es', 'umd']` for one entry, `['es', 'cjs']` for several. `name` is required for `umd` and `iife`. `fileName` defaults to the `package.json` name. https://vite.dev/config/build-options#build-lib
- Externalize dependencies you do not want bundled (e.g. `vue`, `react`) in `rolldownOptions.external`. https://vite.dev/guide/build#library-mode
- In library mode, `build.assetsInlineLimit` is ignored (assets are always inlined) and `cssCodeSplit` defaults to `false`: CSS goes into one file named by `build.lib.cssFileName`. Export it as `"./style.css"`. https://vite.dev/config/build-options#build-assetsinlinelimit , https://vite.dev/guide/build#css-support
- Without `"type": "module"`, output extensions change: `.js` becomes `.mjs`, `.cjs` becomes `.js`. https://vite.dev/guide/build#css-support
- `process.env.*` is left alone in libraries. Use `define` to force it. https://vite.dev/guide/build#css-support
- Recommended `package.json` for one entry: https://vite.dev/guide/build#library-mode

```json
{
  "type": "module",
  "files": ["dist"],
  "main": "./dist/my-lib.umd.cjs",
  "module": "./dist/my-lib.js",
  "exports": { ".": { "import": "./dist/my-lib.js", "require": "./dist/my-lib.umd.cjs" } }
}
```

- For non-browser libraries, use tsdown or Rolldown directly. https://vite.dev/guide/build#css-support

## Content Security Policy

- `html.cspNonce: 'PLACEHOLDER'` adds nonces to generated script, style and modulepreload tags, plus a `<meta property="csp-nonce">`. Replace the placeholder with a fresh value per request, otherwise the policy is bypassable. https://vite.dev/guide/features#nonce-random
- Inlined assets are `data:` URIs: allow `data:` in `img-src`/`font-src` or set `build.assetsInlineLimit: 0`. Never allow `data:` in `script-src`. https://vite.dev/guide/features#data
