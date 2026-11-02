# ES modules

## Contents

- How modules differ from scripts
- import and export
- Dynamic import and import.meta
- Import attributes, deferred and source imports
- Import maps and loading
- Old patterns

## How modules differ from scripts

- Modules are strict, deferred, evaluated once, and their bindings are not global (unreachable from the console). Top-level `this` is `undefined`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Modules#other_differences_between_modules_and_classic_scripts , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/this
- `import`/`export` in a `<script>` without `type="module"` is a `SyntaxError`. Inline module scripts can import, but nothing can import from them. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Modules#applying_the_module_to_your_html
- `file://` URLs fail with CORS: serve modules over HTTP with a JavaScript MIME type (`text/javascript`). https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Modules#other_differences_between_modules_and_classic_scripts , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Modules#troubleshooting
- Top-level `await` works in modules and blocks modules that depend on it, not siblings. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Modules#top_level_await

```js
const colors = fetch("../data/colors.json").then((r) => r.json());
export default await colors;
```
https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Modules#top_level_await

- `using` is allowed at module top level, unlike a classic script. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Resource_management

## import and export

- Imports are read-only live bindings: the exporter can reassign, the importer cannot (it can mutate an exported object). https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/import
- Import declarations are hoisted, so their side effects run before the module body. Still write them at the top. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/import , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Modules#import_declarations_are_hoisted
- `import { x }` is not destructuring of the default export. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/import
- In browsers `./utils` is fetched as that exact URL: no `.js` or `index.js` is added. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/import
- Duplicate export names or 2 default exports are a `SyntaxError`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/export
- Re-export a default with `export { default } from "m"` or `export { x as default }`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/export
- Names can be strings: `export { variable1 as "string name" }`, aliased on import. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/export
- Never export a function named `then`: `import()` treats the namespace as a thenable. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/import
- Avoid import cycles: merge modules or move shared code into a third. A cycle fails only when a binding is read synchronously before initialization. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Modules#cyclic_imports
- `export using` is a `SyntaxError`. Exporting a variable declared with `using` elsewhere is discouraged: it is disposed when the module finishes executing, so importers get a disposed resource. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/export , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/using

## Dynamic import and import.meta

- Load conditionally with `import()`, which also works in classic scripts but throws in service workers and worklets. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Modules#dynamic_module_loading , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/import
- `import()` caches the namespace. Cache-busting with `?t=${Date.now()}` can leak memory in a long-running application, and the cache cannot be cleared. Only evaluation failures are cached. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/import
- Fetching source and evaluating it by hand (blob URLs, `vm.Module`) is not equivalent to `import()`: fetch destination, CSP and module resolution may not be applied correctly. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/import
- Resolve specifiers with `import.meta.resolve()`, which honors import maps. It is not ECMAScript, but Node.js and Deno implement it. `new URL(x, import.meta.url)` ignores import maps. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/import.meta/resolve
- Replace `__dirname` with `new URL("./file", import.meta.url)`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/import.meta

## Import attributes, deferred and source imports

- Load JSON or CSS with `with { type: ... }`. The type is mandatory, so data never runs as code. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Modules#loading_non-javascript_resources

```js
import colors from "./colors.json" with { type: "json" };
import styles from "./styles.css" with { type: "css" };
```
https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Modules#loading_non-javascript_resources

- JSON modules have only a default export. Attributes also work on `export ... from` and as the second argument of `import()`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/import/with
- `import defer * as ns` (experimental) postpones evaluation until the namespace's properties are accessed. Do not defer what must run first, such as polyfills. Modules with top-level `await` still evaluate early. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/import/defer

```js
import defer * as ts from "typescript";
function compileFile(path) {
  const program = ts.createProgram([path], {});
}
```
https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/import/defer

- Never destructure `await import.defer(...)`: that evaluates at once. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/import/defer
- `import source` and `import.source()` (experimental) give a `WebAssembly.Module` only. Each `WebAssembly.instantiate()` makes a new instance. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/import/source , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/import/source

## Import maps and loading

- Map bare names and path prefixes (key and value both ending in `/`) with an import map, which also hides hashed file names. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Modules#importing_modules_using_import_maps
- Preload with `<link rel="modulepreload">`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Modules#applying_the_module_to_your_html
- For code shared across environments, keep a pure core, feature-detect globals (`typeof window`, `typeof process`), polyfill with `import()`, and use `globalThis`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Modules#authoring_isomorphic_modules

## Old patterns

| Old pattern | Replacement | Source |
| :-- | :-- | :-- |
| `assert { type: "json" }` | `with { type: "json" }` | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/import/with |
| `new URL(spec, import.meta.url)` for bare names | `import.meta.resolve(spec)` | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/import.meta/resolve |
| `__dirname` | `import.meta.url` | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/import.meta |
| `<script defer>` for modules | nothing, modules are deferred | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Modules#other_differences_between_modules_and_classic_scripts |
