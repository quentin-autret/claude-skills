# ES modules and CommonJS

## Contents

- Choosing the module system
- ESM specifics
- import.meta
- Importing CommonJS from ESM
- require(esm)
- CommonJS specifics
- Old patterns

## Choosing the module system

- Always set `"type"` in `package.json`, even for an all-CommonJS package. A typeless `.js` file is tried as CommonJS, then retried as ESM if ESM syntax appears: a startup cost, and not future-proof. https://nodejs.org/docs/latest/api/packages.html#determining-module-system
- `.mjs` is always ESM and `.cjs` always CommonJS, whatever `"type"` says. The nearest `package.json` above the file decides `.js`. https://nodejs.org/docs/latest/api/packages.html#package-json-and-file-extensions , https://nodejs.org/docs/latest/api/packages.html#type
- Syntax detection treats static `import`/`export`, `import.meta` and top-level `await` as ESM. `import()` alone does not, since it is valid in CommonJS. https://nodejs.org/docs/latest/api/packages.html#syntax-detection
- `require()` always uses the CommonJS loader, `import()` the ESM loader. `import()` works from both and loads both. https://nodejs.org/docs/latest/api/modules.html#enabling

## ESM specifics

- Relative and absolute specifiers need the full extension and no directory index: `import './startup/index.js'`. https://nodejs.org/docs/latest/api/esm.html#mandatory-file-extensions
- Modules are cached by URL: percent-encode `#` and `?`, and a different query string loads a new instance. Convert paths with `url.pathToFileURL()`. `https:` imports are unsupported. https://nodejs.org/docs/latest/api/esm.html#urls
- JSON needs `with { type: 'json' }` and exposes only `default`. Import attributes replaced the legacy `assert` keyword in v21.0.0. https://nodejs.org/docs/latest/api/esm.html#import-attributes , https://nodejs.org/docs/latest/api/esm.html#json-modules

```js
import fooData from './foo.json' with { type: 'json' };
const { default: barData } = await import('./bar.json', { with: { type: 'json' } });
```

- ESM has no `require`, `module`, `__filename`, `__dirname` or `require.resolve`. Build a `require` with `module.createRequire(import.meta.url)`. Native addons are not supported by `import` (only behind `--experimental-addon-modules`): load them with `createRequire` or `process.dlopen`. https://nodejs.org/docs/latest/api/esm.html#differences-between-es-modules-and-commonjs
- A top-level `await` that never settles exits with code 13. https://nodejs.org/docs/latest/api/esm.html#top-level-await
- Unknown file extensions fail with `ERR_UNKNOWN_FILE_EXTENSION`. https://nodejs.org/docs/latest/api/packages.html#module-resolution-and-loading
- Built-in modules have named exports that update only through `module.syncBuiltinESMExports()`. https://nodejs.org/docs/latest/api/esm.html#built-in-modules
- Wasm modules load without a flag since v24.5.0. Source phase imports (`import source mod from './lib.wasm'`) are a release candidate. https://nodejs.org/docs/latest/api/esm.html#wasm-modules
- Text modules (`with { type: 'text' }`) need `--experimental-import-text`. https://nodejs.org/docs/latest/api/esm.html#text-modules

## import.meta

- Replace `__dirname` / `__filename` with `import.meta.dirname` / `import.meta.filename` (stable since v24.0.0, only for `file:` modules). https://nodejs.org/docs/latest/api/esm.html#importmetadirname
- Replace `require.main === module` with `import.meta.main` (added v24.2.0, early development). https://nodejs.org/docs/latest/api/esm.html#importmetamain

```js
if (import.meta.main) main();
```

- `import.meta.resolve(specifier)` is synchronous and returns a URL string. https://nodejs.org/docs/latest/api/esm.html#importmetaresolvespecifier
- Read assets with `readFileSync(new URL('./data.proto', import.meta.url))`. https://nodejs.org/docs/latest/api/esm.html#importmetaurl

## Importing CommonJS from ESM

- `module.exports` is the default export. Named exports come from static analysis, copied once: later additions and reassignments are not seen. Prefer the default import when detection is unreliable. https://nodejs.org/docs/latest/api/esm.html#commonjs-namespaces

```js
import cjs from 'cjs'; // === module.exports
```

## require(esm)

- `require()` of ESM is stable (no longer experimental in v25.4.0, on by default since v22.12.0). It works only when the whole graph is synchronous: any top-level `await` throws `ERR_REQUIRE_ASYNC_MODULE`, then use `import()`. It returns the namespace, with the default under `.default`. https://nodejs.org/docs/latest/api/modules.html#loading-ecmascript-modules-using-require
- Give CommonJS consumers a single value by exporting `'module.exports'`, attaching named exports to it: https://nodejs.org/docs/latest/api/modules.html#loading-ecmascript-modules-using-require

```js
export function distance(a, b) { /* ... */ }
export default class Point { static distance = distance; }
export { Point as 'module.exports' };
```

- Diagnose with `--trace-require-module` and `--experimental-print-required-tla`. Detect support with `process.features.require_module`. https://nodejs.org/docs/latest/api/modules.html#loading-ecmascript-modules-using-require
- Do not depend on `__esModule` in hand-written CommonJS: it is for transpiler output. https://nodejs.org/docs/latest/api/modules.html#loading-ecmascript-modules-using-require
- An immediate CommonJS/ESM cycle through `require()` throws `ERR_REQUIRE_CYCLE_MODULE`: require lazily inside a function. https://nodejs.org/docs/latest/api/errors.html#err_require_cycle_module

## CommonJS specifics

- Assign `module.exports` synchronously, never in a callback. Reassigning `exports` exports nothing. https://nodejs.org/docs/latest/api/modules.html#moduleexports
- The cache is keyed by resolved filename: `./foo` and `./FOO` on a case-insensitive file system are 2 instances, and copies in different `node_modules` are separate. Export a function to run code again. https://nodejs.org/docs/latest/api/modules.html#module-caching-caveats
- Cycles return a partly filled `exports`. https://nodejs.org/docs/latest/api/modules.html#cycles
- `node:` imports bypass `require.cache`. Newer built-ins require the prefix: `node:sqlite`, `node:test`, `node:test/reporters`, `node:sea`, `node:ffi`. https://nodejs.org/docs/latest/api/modules.html#built-in-modules-with-mandatory-node-prefix
- `require.main` is `undefined` when the entry is ESM. https://nodejs.org/docs/latest/api/modules.html#accessing-the-main-module
- `NODE_PATH` and global folders (`$HOME/.node_modules`) are historic: install locally. https://nodejs.org/docs/latest/api/modules.html#loading-from-the-global-folders

## Old patterns

| Old pattern | Replacement | Source |
| :-- | :-- | :-- |
| `import x from './x.json' assert { type: 'json' }` | `with { type: 'json' }` | https://nodejs.org/docs/latest/api/esm.html#import-attributes |
| `__dirname` in ESM | `import.meta.dirname` | https://nodejs.org/docs/latest/api/esm.html#importmetadirname |
| `require.main === module` in ESM | `import.meta.main` | https://nodejs.org/docs/latest/api/esm.html#importmetamain |
| `module.parent` | `require.main` and `module.children` | https://nodejs.org/docs/latest/api/deprecations.html#DEP0144 |
| `Module.createRequireFromPath()` (end-of-life) | `module.createRequire()` | https://nodejs.org/docs/latest/api/deprecations.html#DEP0130 |
| `require.extensions['.x'] = ...` | compile ahead of time | https://nodejs.org/docs/latest/api/modules.html#requireextensions |
| folder `index.js` / `main` resolution | `"exports"` / `"imports"` | https://nodejs.org/docs/latest/api/modules.html#folders-as-modules |
| `this.x = ...` at CommonJS top level | `exports.x = ...` | https://nodejs.org/docs/latest/api/deprecations.html#DEP0092 |
| dynamic import to avoid `ERR_REQUIRE_ESM` | plain `require()` of sync ESM | https://nodejs.org/docs/latest/api/errors.html#err_require_esm |
