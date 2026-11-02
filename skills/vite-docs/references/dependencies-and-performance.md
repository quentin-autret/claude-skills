# Dependency pre-bundling, HMR troubleshooting and performance

## Contents

- How pre-bundling works
- `optimizeDeps` include and exclude
- Caches and forcing a re-bundle
- Linked packages and monorepos
- CommonJS and Node built-ins
- HMR does not apply
- Speeding up the dev server
- Profiling

## How pre-bundling works

- In dev only, Vite pre-bundles dependencies with Rolldown: CommonJS and UMD become ESM (so `import React, { useState } from 'react'` works), and ESM packages with many internal modules become one file (`lodash-es` would otherwise trigger 600+ requests). Production builds do not use it. https://vite.dev/guide/dep-pre-bundling#the-why
- Without a cache, Vite crawls the source for bare imports. A dependency first seen later (for example created by a plugin transform) triggers a re-bundle and possibly a page reload. https://vite.dev/guide/dep-pre-bundling#automatic-dependency-discovery
- `optimizeDeps.entries` replaces entry inference (default: every `.html`, or `input`). When set, only `node_modules` and `outDir` are ignored, add `!` patterns for the rest. https://vite.dev/config/dep-optimization-options#optimizedeps-entries
- `optimizeDeps.holdUntilCrawlEnd` (default `true`) waits for the crawl before releasing the first result, to avoid reloads. Disable it when the scanner plus `include` already find everything. https://vite.dev/config/dep-optimization-options#optimizedeps-holduntilcrawlend
- Customize the optimizer bundler with `optimizeDeps.rolldownOptions` (`esbuildOptions` is deprecated). It ignores `input`, `logLevel` and output format, sourcemap, dir and banner. Its `plugins` merge with Vite's. https://vite.dev/config/dep-optimization-options#optimizedeps-rolldownoptions

## `optimizeDeps` include and exclude

- Rule: include large or CommonJS dependencies, exclude only small packages that are already valid ESM. https://vite.dev/guide/dep-pre-bundling#customizing-the-behavior
- Never exclude a CommonJS dependency. If an excluded ESM package has a CommonJS dependency, include it with `>`: https://vite.dev/config/dep-optimization-options#optimizedeps-exclude

```js
optimizeDeps: {
  include: ['esm-dep > cjs-dep', 'my-lib/components/**/*.vue'], // glob is experimental
  exclude: ['small-esm-dep'],
}
```

- Imports added by plugins are invisible to the scanner: list them in `include` to avoid a re-bundle right after startup. https://vite.dev/guide/dep-pre-bundling#customizing-the-behavior
- Disable the optimizer fully with `optimizeDeps.noDiscovery: true` and an empty `include`. CommonJS dependencies then must be listed in `include`. `optimizeDeps.disabled` is deprecated. https://vite.dev/config/dep-optimization-options#optimizedeps-nodiscovery , https://vite.dev/config/dep-optimization-options#optimizedeps-disabled
- `optimizeDeps.needsInterop` is only for packages Vite warns about, to avoid cold-start full reloads. https://vite.dev/config/dep-optimization-options#optimizedeps-needsinterop
- `optimizeDeps` applies to the `client` environment and stays top level even with several environments. https://vite.dev/guide/api-environment#environments-configuration

## Caches and forcing a re-bundle

- Filesystem cache in `cacheDir` (default `node_modules/.vite`). It is rebuilt when the lockfile content, the patches folder mtime, relevant config fields or `NODE_ENV` change. https://vite.dev/guide/dep-pre-bundling#file-system-cache
- Force it with `vite --force` or by deleting `node_modules/.vite`. `vite optimize` is deprecated. https://vite.dev/guide/cli
- The browser caches pre-bundled deps with `max-age=31536000,immutable`. To debug a local edit inside a dependency: tick "Disable cache" in DevTools, restart with `--force`, reload. Do not leave "Disable cache" on otherwise, it slows dev down. https://vite.dev/guide/dep-pre-bundling#browser-cache , https://vite.dev/guide/performance#review-your-browser-setup

## Linked packages and monorepos

- Linked packages (resolved outside `node_modules`) are treated as source, not pre-bundled. They must be ESM, otherwise add them to `optimizeDeps.include`. https://vite.dev/guide/dep-pre-bundling#monorepos-and-linked-dependencies
- `npm link` and unlink are not part of the cache hash: run `vite --force` after linking or unlinking. Package manager overrides (pnpm `overrides`, Yarn `resolutions`) are detected and recommended over linking. https://vite.dev/guide/troubleshooting#outdated-pre-bundled-deps-when-linking-to-a-local-package
- A duplicated dependency in a monorepo (two copies of React): `resolve.dedupe`. https://vite.dev/config/shared-options#resolve-dedupe

## CommonJS and Node built-ins

- A default import of a CommonJS module can return the whole `module.exports` object ("Element type is invalid ... got: object", "foo is not a function"). Vite 8 applies one rule in dev and build, based on `.mjs`, `"type": "module"` and `__esModule`. https://vite.dev/guide/troubleshooting#default-import-unexpectedly-returns-an-object , https://vite.dev/guide/migration#consistent-commonjs-interop
- "Module "fs" has been externalized for browser compatibility": Vite never polyfills Node modules. Remove the Node import from browser code, or polyfill it yourself. https://vite.dev/guide/troubleshooting#module-externalized-for-browser-compatibility
- Sloppy-mode code (`with`, writing a property on a primitive) throws because ESM is strict. Patch the dependency (`patch-package`, `pnpm patch`). https://vite.dev/guide/troubleshooting#syntax-error-type-error-happens

## HMR does not apply

- Change detected but no HMR: the import casing differs from the file name (`./Foo.js` for `foo.js`). https://vite.dev/guide/troubleshooting#vite-detects-a-file-change-but-the-hmr-is-not-working
- Full reload instead of HMR: no plugin accepts the update, or the module sits in a circular import. `vite --debug hmr` prints the cycle. https://vite.dev/guide/troubleshooting#a-full-reload-happens-instead-of-hmr
- No change detected at all on WSL2: see `server.watch`. https://vite.dev/guide/troubleshooting#vite-does-not-detect-a-file-change

## Speeding up the dev server

- Write explicit extensions (`import './Component.jsx'`): each implicit extension costs filesystem checks. Narrowing `resolve.extensions` also helps, as long as `node_modules` still resolves. https://vite.dev/guide/performance#reduce-resolve-operations
- Avoid barrel files. Importing one export from `utils/index.js` loads and transforms every re-exported file. Import from the file: `import { slash } from './utils/slash.js'`. https://vite.dev/guide/performance#avoid-barrel-files
- Warm up files that create transform waterfalls. Find them with `vite --debug transform`. https://vite.dev/guide/performance#warm-up-frequently-used-files

```js
server: { warmup: { clientFiles: ['./src/components/BigComponent.vue', './src/utils/big-utils.js'] } }
```

- `server.open` also warms up the entry. https://vite.dev/guide/performance#warm-up-frequently-used-files
- Use plain CSS when possible, and import SVGs as strings or URLs instead of turning them into framework components. https://vite.dev/guide/performance#use-lesser-or-native-tooling
- Test in a browser profile without extensions. https://vite.dev/guide/performance#review-your-browser-setup
- Slow plugins: measure with `vite --debug plugin-transform` or `vite-plugin-inspect`. In your own plugins, keep `config`, `configResolved` and `buildStart` short (dev startup awaits them), import big rarely-used dependencies dynamically, and in `resolveId`, `load` and `transform` check the `code` for a keyword or the `id` for an extension before transforming. https://vite.dev/guide/performance#audit-configured-vite-plugins
- Large builds: `build.reportCompressedSize: false` skips gzip size reporting. https://vite.dev/config/build-options#build-reportcompressedsize

## Profiling

- `vite --profile --open`, load the page, press `p` then `q` in the terminal. `vite build --profile` works too. Open the `.cpuprofile` in speedscope. https://vite.dev/guide/troubleshooting#performance-bottlenecks , https://vite.dev/guide/performance#audit-configured-vite-plugins
- `vite-plugin-inspect` (`localhost:5173/__inspect/`) shows each module's transformation stack and helps identify which plugins are the bottleneck, in dev and build. https://vite.dev/guide/api-plugin#authoring-a-plugin , https://vite.dev/guide/troubleshooting#performance-bottlenecks
