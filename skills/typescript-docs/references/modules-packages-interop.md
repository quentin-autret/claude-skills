# Modules: packages, paths and ESM/CJS interop

## Contents

- Extension substitution and sibling declarations
- `paths` and `baseUrl`
- package.json `exports`, `imports`, `types`, `typesVersions`
- ESM and CJS interop
- Interop for library authors
- Import and export syntax

## Extension substitution and sibling declarations

- A runtime lookup of `/mod.js` tries `.ts`, `.tsx`, `.d.ts`, `.js`, `.jsx`. `/mod.mjs` tries `.mts`, `.d.mts`, `.mjs`. `/mod.cjs` tries `.cts`, `.d.cts`, `.cjs`. This applies even when the specifier says `.js`. https://www.typescriptlang.org/docs/handbook/modules/reference.html#file-extension-substitution
- TypeScript stops at the first TS or declaration file and assumes it describes the sibling JS exactly. Every JS file needs a sibling declaration with the same name and matching extension. Hand-written, copied or renamed declarations cause false errors for users: check with `@arethetypeswrong/cli`. https://www.typescriptlang.org/docs/handbook/modules/theory.html#the-role-of-declaration-files
- Bare specifiers look for types in `node_modules/<pkg>` then `node_modules/@types/<pkg>` in every ancestor, and only then for JS. A JS-only resolution is `any`. https://www.typescriptlang.org/docs/handbook/modules/reference.html#node_modules-package-lookups

## `paths` and `baseUrl`

- `paths` never changes emitted specifiers. An alias can type check and crash in Node.js. Use it only to mirror a mapping a bundler or runtime already applies. https://www.typescriptlang.org/docs/handbook/modules/reference.html#paths-does-not-affect-emit
- Published libraries must not set `paths`: the emitted JavaScript breaks for consumers without the same aliases. Apps and libraries can use package.json `"imports"` (`#` specifiers) as a standard replacement for convenience aliases. https://www.typescriptlang.org/docs/handbook/modules/reference.html#paths-does-not-affect-emit
- Never point `paths` at monorepo packages or `node_modules`: the match resolves like a relative path and bypasses `exports`, `main`, `types`. Link packages with workspaces. https://www.typescriptlang.org/docs/handbook/modules/reference.html#paths-should-not-point-to-monorepo-packages-or-node_modules-packages
- `paths` values resolve against the tsconfig that defines them. One `*` per pattern, longest prefix wins, arrays are fallbacks. https://www.typescriptlang.org/docs/handbook/modules/reference.html#wildcard-substitutions
- `baseUrl` is deprecated in 6.0 and no longer a look-up root for module resolution. Before, it had higher precedence than `node_modules` lookups. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html#deprecated---baseurl

```json
{ "compilerOptions": { "paths": { "app/*": ["./src/app/*"] } } }
```

https://www.typescriptlang.org/tsconfig/#paths

## package.json `exports`, `imports`, `types`, `typesVersions`

- `exports` is read under `node16`, `nodenext` and `bundler`. TypeScript always matches `types` and `default`, plus `import` or `require` by syntax, plus `customConditions`. https://www.typescriptlang.org/docs/handbook/modules/reference.html#packagejson-exports
- `exports` blocks every subpath it does not list: with `"exports": "./dist/index.js"`, `pkg/dist/index.js` is unresolvable. https://www.typescriptlang.org/docs/handbook/modules/reference.html#example-exports-block-other-subpaths
- Resolution takes the first condition that matches. The page's example lists `types` first in each block. https://www.typescriptlang.org/docs/handbook/modules/reference.html#example-explicit-types-condition

```json
"./subpath": {
  "import": { "types": "./types/subpath/index.d.mts", "default": "./es/subpath/index.mjs" },
  "require": { "types": "./types/subpath/index.d.cts", "default": "./cjs/subpath/index.cjs" }
}
```

https://www.typescriptlang.org/docs/handbook/modules/reference.html#example-explicit-types-condition

- Versioned conditions `"types@>=5.2"` use `typesVersions` selector rules and are tried in written order: the page lists `>=5.2` before `>=4.6`. https://www.typescriptlang.org/docs/handbook/modules/reference.html#example-versioned-types-condition
- `typesVersions` is not read when `exports` is read. https://www.typescriptlang.org/docs/handbook/modules/reference.html#packagejson-typesversions
- `typesVersions` ranges overlap, so the redirect depends on order: write the higher range first (`>=3.2` before `>=3.1`). https://www.typescriptlang.org/docs/handbook/declaration-files/publishing.html#multiple-fields
- Without `exports`: `types`, then `typings` (legacy), then `main`. Keep a `"types"` field anyway, npm shows the TS badge from it. https://www.typescriptlang.org/docs/handbook/modules/reference.html#packagejson-main-and-types
- For a local `imports` or self-name import, TypeScript maps the output path back to the source through `outDir` and `rootDir`, so set `rootDir` explicitly. https://www.typescriptlang.org/docs/handbook/modules/reference.html#packagejson-imports-and-self-name-imports
- A scoped custom condition pointing at sources lets a workspace run `.ts` directly: `node --conditions=@my-package/development`. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-7.html#path-rewriting-for-relative-paths

## ESM and CJS interop

- `import * as ns` is always a namespace object and can never be called. https://www.typescriptlang.org/docs/handbook/modules/appendices/esm-cjs-interop.html
- In Node.js, a default import of CJS is always the whole `module.exports`, whatever `__esModule` says. A transpiled dependency with `exports.default = fn` is `fn()` in a bundler but `fn.default()` in real Node ESM. `node16` and `nodenext` catch it. https://www.typescriptlang.org/docs/handbook/modules/appendices/esm-cjs-interop.html#no-__esmodule-detection-the-double-default-problem
- Named imports from CJS come from static analysis in Node. Reading properties off the default import always works. https://www.typescriptlang.org/docs/handbook/modules/appendices/esm-cjs-interop.html#unreliable-named-exports
- `require()` of ESM crashes before Node 22.12.0 per the interop page (the theory page footnote says 20.19.0). `nodenext` models it as allowed but cannot detect a top-level `await` target, which throws. `node16` and `node18` forbid it. Dynamic `import()` always works. https://www.typescriptlang.org/docs/handbook/modules/reference.html#interoperability-rules
- No setting stops an ES module from importing a CJS dependency. https://www.typescriptlang.org/docs/handbook/modules/theory.html#esm-and-cjs-interoperability

## Interop for library authors

- A library shipping CJS avoids default exports: they are `pkg.default()` in Node ESM, `pkg()` in bundlers. Use `export =` for a single main export, named exports otherwise. https://www.typescriptlang.org/docs/handbook/modules/appendices/esm-cjs-interop.html#library-code-needs-special-considerations
- In `.d.ts`, a default import that depends on `esModuleInterop` infects consumers. Prefer `import express = require("express")`. https://www.typescriptlang.org/docs/handbook/modules/appendices/esm-cjs-interop.html

## Import and export syntax

- `import type` and inline `type` specifiers are always erased. A value imported with `import type` only works in type positions (TS1361 if called). https://www.typescriptlang.org/docs/handbook/2/modules.html
- `import type fs, { X } from "fs"` is an error: write `import type { default as fs, X } from "fs"`. https://www.typescriptlang.org/docs/handbook/modules/reference.html#type-only-imports-and-exports
- `export type * from "./m"` exists. Using `ns.Class` as a value from it errors. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-0.html#support-for-export-type-
- Import attributes use `with`: `import obj from "./x.json" with { type: "json" }`. The `asserts` keyword on imports is deprecated in 6.0 and errors. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-3.html#import-attributes and https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html#deprecated-asserts-keyword-on-imports
- A global `interface ImportAttributes { type: "json" }` restricts accepted attribute values. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-4.html#checked-import-attributes-and-assertions
- `resolution-mode` picks a resolution for a type import: `import type { T } from "pkg" with { "resolution-mode": "require" }`. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-3.html#stable-support-resolution-mode-in-import-types
- `declare module "path" { }` in a script file is an ambient module. With a top-level `import` or `export`, the same block becomes an augmentation. Put imports inside the block to keep it ambient. https://www.typescriptlang.org/docs/handbook/modules/reference.html#ambient-modules
- A wildcard module types assets: `declare module "*.html" { const content: string; export default content; }`. https://www.typescriptlang.org/docs/handbook/modules/reference.html#ambient-modules
