# TypeScript, JSX, CSS, assets and imports

## Contents

- TypeScript
- JSX and the `oxc` option
- HTML entries
- CSS
- Static assets and the `public` directory
- `new URL(path, import.meta.url)`
- `import.meta.glob` and dynamic imports
- Web Workers
- WebAssembly and JSON

## TypeScript

- Vite transpiles TypeScript with Oxc and never type-checks. Run `tsc --noEmit` in the build script and `tsc --noEmit --watch` (or `vite-plugin-checker`) beside the dev server. Do not put type checking or linting into the transform pipeline. https://vite.dev/guide/features#transpile-only
- Set `"isolatedModules": true`: `const enum` and implicit type-only imports cannot work per file. Write `import type { T }` and `export type { T }`. https://vite.dev/guide/features#isolatedmodules , https://vite.dev/guide/features#transpile-only
- `tsconfig` `target` is ignored. Dev uses `oxc.target` (default `esnext`), builds use `build.target`. https://vite.dev/guide/features#target
- `useDefineForClassFields` defaults to `true` for ES2022+/ESNext targets. Set `false` when a library relies on the old semantics. https://vite.dev/guide/features#usedefineforclassfields
- `emitDecoratorMetadata` is only partially supported. Oxc cannot lower native decorators: use a Babel or SWC plugin. https://vite.dev/guide/features#emitdecoratormetadata , https://vite.dev/guide/migration#javascript-transforms-by-oxc
- Each file uses its closest `tsconfig.json` (honoring `references`, `include`, `exclude`). Vite config values win over tsconfig ones. https://vite.dev/guide/features#typescript-compiler-options
- `paths` aliases need `resolve.tsconfigPaths: true`. https://vite.dev/guide/features#paths

Client types (`import.meta.env`, `import.meta.hot`, asset imports):

```json
{ "compilerOptions": { "types": ["vite/client"] } }
```

When `types` is set, only listed packages enter the global scope, so list every global lib. The alternative is `/// <reference types="vite/client" />` in `vite-env.d.ts`. To override a default typing (for example `*.svg` as a component), put your declaration in a `.d.ts` referenced before `vite/client`. https://vite.dev/guide/features#client-types

## JSX and the `oxc` option

- `.jsx` and `.tsx` are transformed by Oxc. Configure through `oxc`, which replaces the deprecated `esbuild` option. `oxc: false` disables the transform. https://vite.dev/guide/features#jsx , https://vite.dev/config/shared-options#oxc

```js
export default defineConfig({
  oxc: { jsx: { importSource: 'preact' }, jsxInject: `import React from 'react'` },
})
```

- `oxc` applies to `ts`, `jsx`, `tsx` by default. Widen or narrow with `oxc.include` / `oxc.exclude`. https://vite.dev/config/shared-options#oxc

## HTML entries

- `index.html` sits at the project root, not in `public`, and is part of the module graph. Its URLs are rebased automatically, there is no `%PUBLIC_URL%`. https://vite.dev/guide/#index-html-and-project-root
- Processed elements include `<script type="module" src>`, `<link href>`, `<img src/srcset>`, `<video>`, `<source>`, `<audio>` and `og:image`-style `<meta content>`. Add `vite-ignore` to an element to leave it alone (CDN URLs). https://vite.dev/guide/features#html
- Custom elements or `data-*` attributes holding asset URLs: declare them in `html.additionalAssetSources` (`{ img: { srcAttributes: ['data-src-dark'] } }`). https://vite.dev/config/shared-options#html-additionalassetsources

## CSS

- `import './a.css'` injects a `<style>` with HMR. `@import` is inlined, aliases work, `url()` is rebased. https://vite.dev/guide/features#css
- To get CSS as a string without injecting it, use `?inline`. Default and named imports from a `.css` file (`import s from './a.css'`) were removed. https://vite.dev/guide/features#disabling-css-injection-into-the-page

```js
import styles from './bar.css?inline'
```

- `*.module.css` returns a class map. Named imports need `css.modules.localsConvention` set to a camelCase variant. https://vite.dev/guide/features#css-modules
- Sass, Less and Stylus are built in, but install the compiler (`sass-embedded` is used if installed and recommended for best performance). Stylus gets no alias or `url()` rebasing. In Sass and Less, `url()` starting with a variable is not rebased. https://vite.dev/guide/features#css-pre-processors , https://vite.dev/config/shared-options#css-preprocessoroptions
- `preprocessorOptions.<ext>.additionalData` is prepended to every file: put only variables there (real rules get duplicated) and use absolute or aliased paths. https://vite.dev/config/shared-options#css-preprocessoroptions-extension-additionaldata
- An inline `css.postcss` takes only the array form of `plugins` and stops Vite from searching PostCSS config files. Config files outside the workspace root are not found without an explicit path. https://vite.dev/config/shared-options#css-postcss
- PostCSS still processes CSS. Lightning CSS minifies it by default, using `build.cssTarget`. Full Lightning CSS processing is opt-in with `css.transformer: 'lightningcss'`, and then CSS modules are configured in `css.lightningcss.cssModules`, not `css.modules`. https://vite.dev/guide/features#lightning-css , https://vite.dev/config/shared-options#css-modules

## Static assets and the `public` directory

- `import url from './img.png'` returns `/src/img.png` in dev and `/assets/img.<hash>.png` in build. Files under `build.assetsInlineLimit` (4096 bytes) become base64 data URLs. https://vite.dev/guide/assets#importing-asset-as-url , https://vite.dev/config/build-options#build-assetsinlinelimit
- Git LFS pointer files are never inlined: fetch LFS content before building. https://vite.dev/guide/assets#importing-asset-as-url
- Unknown file types: add patterns to `assetsInclude` (`['**/*.gltf']`). https://vite.dev/config/shared-options#assetsinclude
- Query suffixes:

```js
import workletURL from 'pkg/worklet.js?url'   // URL for a type Vite does not know
import a from './img.svg?no-inline'           // never inline
import b from './img.png?inline'              // always inline
import shader from './shader.glsl?raw'        // string content
```

- An SVG URL put into `url()` from JS must be quoted: `` `url("${imgUrl}")` ``. https://vite.dev/guide/assets#importing-asset-as-url
- `public/` (option `publicDir`, `false` disables) is for files never imported, needing a fixed name (`robots.txt`), or not worth hashing. Reference them by absolute path: `public/icon.png` is `/icon.png`. They are not transformed. Default to importing assets. https://vite.dev/guide/assets#the-public-directory , https://vite.dev/config/shared-options#publicdir

## `new URL(path, import.meta.url)`

```js
const imgUrl = new URL('./img.png', import.meta.url).href
const byName = (name) => new URL(`./dir/${name}.png`, import.meta.url).href // no subdirectories
const notTransformed = new URL(imagePath, import.meta.url).href // dynamic, left as is
```

- The string must be static (or a template literal) for Vite to rewrite it in build. https://vite.dev/guide/assets#new-url-url-import-meta-url
- This pattern does not work in SSR: `import.meta.url` differs between Node and the browser. https://vite.dev/guide/assets#new-url-url-import-meta-url

## `import.meta.glob` and dynamic imports

- `import.meta.glob` is lazy by default (one chunk per file). `eager: true` imports statically. With `eager`, `import: 'default'` or a named export tree-shakes the rest. https://vite.dev/guide/features#glob-import

```js
const pages = import.meta.glob('./pages/*.js', { import: 'default', eager: true })
const svgs = import.meta.glob(['./icons/*.svg', '!./icons/_*.svg'], { query: '?raw', import: 'default' })
```

- Arguments must be literals. Patterns start with `./`, `/` (root) or an alias. `base` cannot use aliases. https://vite.dev/guide/features#glob-import-caveats
- Dynamic import with a variable: the variable spans one directory level, the path starts with `./`, `../` or `#`, ends with an extension, and needs a file prefix in the own directory. https://vite.dev/guide/features#dynamic-import

```js
await import(`./dir/${file}.js`) // valid
// invalid: import(`${foo}.js`), import(`./dir/${foo}`), import(`./${foo}.js`)
```

## Web Workers

- Default pattern, with the `new URL()` written inside `new Worker()` and static options. Otherwise Vite treats it as a plain asset URL. https://vite.dev/guide/features#import-with-constructors

```js
const worker = new Worker(new URL('./worker.js', import.meta.url), { type: 'module' })
```

- Escape hatch: `?worker`, `?sharedworker`, `?worker&inline`, `?worker&url`. https://vite.dev/guide/features#import-with-query-suffixes
- Worker bundles default to `worker.format: 'iife'`. Top-level `plugins` apply to workers in dev only: set `worker.plugins` as a function returning fresh plugin instances for build. Changing `config.worker` in a plugin `config` hook is ignored. https://vite.dev/config/worker-options#worker-format , https://vite.dev/config/worker-options#worker-plugins

## WebAssembly and JSON

- `import { add } from './add.wasm'` works as an async module (needs top-level await). `./add.wasm?init` returns an init function taking an import object. For the `Module`, use `?url` with `WebAssembly.instantiateStreaming(fetch(url))`. https://vite.dev/guide/features#webassembly
- TypeScript: enable `allowArbitraryExtensions` and add `add.d.wasm.ts`. https://vite.dev/guide/features#esm-integration
- In SSR builds, both wasm forms use `node:fs`, so they run in Node-compatible runtimes only. https://vite.dev/guide/features#manual-initialization
- JSON named imports are supported and tree-shake (`json.namedExports` on). `json.stringify: true` emits `JSON.parse(...)`, faster for big files (`'auto'` does it above 10 kB). https://vite.dev/guide/features#json , https://vite.dev/config/shared-options#json-stringify
