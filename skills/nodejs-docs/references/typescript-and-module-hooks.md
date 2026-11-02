# TypeScript type stripping and the node:module API

## Contents

- Type stripping
- TypeScript limits
- Customization hooks
- Compile cache
- Other node:module helpers
- Old patterns

## Type stripping

- Node runs `.ts` files by stripping types: on by default since v23.6.0 (v22.18.0), stable since v25.2.0 (backported to v24.12.0). Disable with `--no-strip-types`, detect with `process.features.typescript`. https://nodejs.org/docs/latest/api/typescript.html#type-stripping , https://nodejs.org/docs/latest/api/process.html#processfeaturestypescript
- It replaces types with whitespace: no type checking, `tsconfig.json` ignored, no source maps needed (line numbers stay right). https://nodejs.org/docs/latest/api/typescript.html#type-stripping , https://nodejs.org/docs/latest/api/typescript.html#source-maps
- Recommended tsconfig (TypeScript 5.8+): `"target": "esnext"`, `"module": "nodenext"`, `"rewriteRelativeImportExtensions": true`, `"erasableSyntaxOnly": true`, `"verbatimModuleSyntax": true`, optionally `"noEmit": true`. https://nodejs.org/docs/latest/api/typescript.html#type-stripping
- Write extensions (`import './file.ts'`, also in `require`) and mark type-only imports, or they fail at run time: https://nodejs.org/docs/latest/api/typescript.html#determining-module-system , https://nodejs.org/docs/latest/api/typescript.html#importing-types-without-type-keyword

```ts
import type { Type1, Type2 } from './module.ts';
import { fn, type FnParams } from './fn.ts';
```

- `.ts` follows `"type"` like `.js`. `.mts` is always ESM, `.cts` always CommonJS, `.tsx` is unsupported. https://nodejs.org/docs/latest/api/typescript.html#determining-module-system

## TypeScript limits

- Only erasable syntax works. `enum`, runtime `namespace`, parameter properties and import aliases throw `ERR_UNSUPPORTED_TYPESCRIPT_SYNTAX`, decorators are a parse error. https://nodejs.org/docs/latest/api/typescript.html#typescript-features
- Files under `node_modules` are never stripped: publish JavaScript. https://nodejs.org/docs/latest/api/typescript.html#type-stripping-in-dependencies
- tsconfig `paths` do not work: use `"imports"` with `#` keys. https://nodejs.org/docs/latest/api/typescript.html#paths-aliases
- Not supported in the REPL, `--check` or `node inspect`. Stripping needs WebAssembly. https://nodejs.org/docs/latest/api/typescript.html#non-file-forms-of-input
- For full TypeScript use a third-party runner, for example `node --import=tsx file.ts`. https://nodejs.org/docs/latest/api/typescript.html#full-typescript-support
- `--experimental-transform-types` is removed in v26.0.0. `module.stripTypeScriptTypes()` lost its `transform` and `sourceMap` options in v26.0.0, and its output can change between versions. https://nodejs.org/docs/latest/api/typescript.html , https://nodejs.org/docs/latest/api/module.html#modulestriptypescripttypescode-options

## Customization hooks

- Default to synchronous in-thread `module.registerHooks()` (release candidate since v25.4.0, backported to v24.13.1). Async `module.register()` is deprecated, a runtime warning since v26.0.0. https://nodejs.org/docs/latest/api/module.html#moduleregisterhooksoptions , https://nodejs.org/docs/latest/api/deprecations.html#dep0205-moduleregister
- Register before application code, through `--import` or `--require`. Static imports in the registering file run before `registerHooks()`. https://nodejs.org/docs/latest/api/module.html#registration-of-synchronous-customization-hooks

```js
// register-hooks.js
import { registerHooks } from 'node:module';
registerHooks({
  resolve(specifier, context, nextResolve) { return nextResolve(specifier, context); },
  load(url, context, nextLoad) { return nextLoad(url, context); },
});
```

```bash
node --import ./register-hooks.js ./my-app.js
```

- Hooks chain last-registered-first. Each must call `next*()` or return `shortCircuit: true`. `registerHooks()` returns `{ deregister() }` and supports `Symbol.dispose`. https://nodejs.org/docs/latest/api/module.html#convention-of-hooks-and-chaining
- Async hooks run on another thread: they cannot touch main-thread state, do not affect `createRequire()`, use the `import` condition for CommonJS and may load CommonJS sources more than once. https://nodejs.org/docs/latest/api/module.html#caveats-of-asynchronous-customization-hooks , https://nodejs.org/docs/latest/api/module.html#asynchronous-resolvespecifier-context-nextresolve
- `import.meta.resolve()` is unavailable inside custom loaders. https://nodejs.org/docs/latest/api/esm.html#importmetaresolvespecifier

## Compile cache

- `module.enableCompileCache()` or `NODE_COMPILE_CACHE=dir` (stable since v25.4.0) stores V8 code cache on disk. The first run may be slower, and a cache is only reusable by the same Node version. https://nodejs.org/docs/latest/api/module.html#module-compile-cache

```js
module.enableCompileCache({ directory: '/path/to/cache/storage/dir', portable: true });
```

- `portable: true` survives moving the project. `readOnly: true` serves a cache shipped with the app. `flushCompileCache()` writes before exit. Disable it for precise V8 coverage. https://nodejs.org/docs/latest/api/module.html#portability-of-the-compile-cache , https://nodejs.org/docs/latest/api/module.html#read-only-compile-cache , https://nodejs.org/docs/latest/api/module.html#limitations-of-the-compile-cache

## Other node:module helpers

- `module.findPackageJSON(specifier, base)` (active development): pass `__filename`, not `__dirname`. It ignores resolve hooks and must not decide the module format. https://nodejs.org/docs/latest/api/module.html#modulefindpackagejsonspecifier-base
- Enable source maps with `--enable-source-maps` or `module.setSourceMapsSupport()` (experimental). https://nodejs.org/docs/latest/api/module.html#source-map-support
- `module.builtinModules` lists built-ins, prefix-only ones with `node:`. https://nodejs.org/docs/latest/api/module.html#modulebuiltinmodules

## Old patterns

| Old pattern | Replacement | Source |
| :-- | :-- | :-- |
| `module.register('./hooks.mjs', import.meta.url)` | `module.registerHooks({ resolve, load })` | https://nodejs.org/docs/latest/api/deprecations.html#dep0205-moduleregister |
| `--experimental-transform-types` for enums | rewrite as erasable syntax or use a third-party runner | https://nodejs.org/docs/latest/api/typescript.html#typescript-features |
| `--no-experimental-strip-types` | `--no-strip-types` | https://nodejs.org/docs/latest/api/cli.html#--no-strip-types |
