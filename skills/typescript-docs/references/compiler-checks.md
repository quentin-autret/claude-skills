# Strictness, checks and emit flags

## Contents

- `strict` and its family
- Checks outside `strict`
- `verbatimModuleSyntax`
- `isolatedModules`
- `erasableSyntaxOnly`
- `esModuleInterop` and `allowSyntheticDefaultImports`
- Import extension flags
- Side-effect imports, JSON, module detection
- Class fields

## `strict` and its family

- In 6.0, `strict` defaults to `true`. Set `"strict": false` to keep 5.x behavior, or turn single family members off. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html#simple-default-changes
- Future versions may add checks under `strict`, so an upgrade can surface new errors. https://www.typescriptlang.org/tsconfig/#strict
- `strictBuiltinIteratorReturn` belongs to `strict`: `next().value` of built-in iterators is `T | undefined` until `done` is checked. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-6.html#strict-builtin-iterator-checks-and-strictbuiltiniteratorreturn
- `useUnknownInCatchVariables` types every catch variable `unknown`. Narrow before use. https://www.typescriptlang.org/tsconfig/#useUnknownInCatchVariables

```ts
try { /* ... */ } catch (err: unknown) {
  if (err instanceof Error) { console.log(err.message); }
}
```

https://www.typescriptlang.org/tsconfig/#useUnknownInCatchVariables

- `strictPropertyInitialization` only follows the constructor body, never methods it calls. Use `name!: string` only when something else fills the field. https://www.typescriptlang.org/docs/handbook/2/classes.html#--strictpropertyinitialization
- Without `strictFunctionTypes`, function parameters are compared bivariantly, which is unsound. https://www.typescriptlang.org/docs/handbook/type-compatibility.html#function-parameter-bivariance
- A library should compile with `strict`: without it, type-level code that ends up in the output `.d.ts` files can error when a consumer compiles with `strict` (for example an `extends` clause that is an error only under `strictNullChecks`). https://www.typescriptlang.org/docs/handbook/modules/guides/choosing-compiler-options.html#im-writing-a-library

## Checks outside `strict`

- `noUncheckedIndexedAccess` adds `undefined` to reads through an index signature: `env.NODE_ENV` is `string | undefined`, declared `env.NAME` stays `string`. https://www.typescriptlang.org/tsconfig/#noUncheckedIndexedAccess
- `exactOptionalPropertyTypes` makes `prop?: T` mean "absent or `T`". Assigning `undefined` then errors (2322, 2412). https://www.typescriptlang.org/tsconfig/#exactOptionalPropertyTypes
- `noImplicitOverride` requires `override` on every overriding member (error 4114). https://www.typescriptlang.org/tsconfig/#noImplicitOverride
- `noPropertyAccessFromIndexSignature` forces `obj["key"]` for keys only covered by an index signature (error 4111). https://www.typescriptlang.org/tsconfig/#noPropertyAccessFromIndexSignature
- `noFallthroughCasesInSwitch` (7029) requires every non-empty `case` to end with `break`, `return` or `throw`. https://www.typescriptlang.org/tsconfig/#noFallthroughCasesInSwitch
- `noImplicitReturns` checks that all code paths in a function return a value (2366). https://www.typescriptlang.org/tsconfig/#noImplicitReturns

## `verbatimModuleSyntax`

- Imports and exports without `type` are kept, those with `type` are dropped. It replaces the default import elision, which drops imports used only as types and so changes whether side-effect modules run. https://www.typescriptlang.org/tsconfig/#verbatimModuleSyntax

```ts
import type { A } from "a";                 // erased entirely
import { b, type c, type d } from "bcd";    // becomes: import { b } from "bcd"
import { type xyz } from "xyz";             // becomes: import {} from "xyz"
```

https://www.typescriptlang.org/tsconfig/#verbatimModuleSyntax

- `import { type xyz } from "xyz"` is rewritten to `import {} from "xyz"`, while `import type { xyz } from "xyz"` is erased entirely. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-0.html#--verbatimmodulesyntax
- ES `import` and `export` are never rewritten to `require`: in a file emitted as CommonJS they are errors. Write `import foo = require("foo")` and `export = {...}` there. https://www.typescriptlang.org/tsconfig/#verbatimModuleSyntax
- It suits mostly-ESM Node projects. The docs do not recommend it for CommonJS projects that may move to ESM later. https://www.typescriptlang.org/docs/handbook/modules/theory.html#input-module-syntax

## `isolatedModules`

- Reports code a single-file transpilation process (such as Babel or `ts.transpileModule`) cannot interpret correctly. It changes no emit. https://www.typescriptlang.org/tsconfig/#isolatedModules
- Errors: re-exporting a type without `export type`, a `namespace` in a non-module file (1277), using members of an ambient `const enum`. https://www.typescriptlang.org/tsconfig/#isolatedModules

## `erasableSyntaxOnly`

- For type stripping (Node.js 23.6+, ts-blank-space, Amaro): errors on syntax with runtime semantics. https://www.typescriptlang.org/tsconfig/#erasableSyntaxOnly
- Banned: `enum`, `namespace` or `module` with runtime code, class parameter properties, `import x = require()`, `import Bar = container.Bar`, `export =`, `<T>expr` assertions. The flag catches most such constructs, not all. https://www.typescriptlang.org/tsconfig/#erasableSyntaxOnly

```ts
class Point { constructor(public x: number, public y: number) { } } // error
```

https://www.typescriptlang.org/tsconfig/#erasableSyntaxOnly

- Combine it with `verbatimModuleSyntax` so no import elision happens. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-8.html#the---erasablesyntaxonly-option

## `esModuleInterop` and `allowSyntheticDefaultImports`

- In 6.0, neither can be `false`. The option page still describes the off state as default: ignore it. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html#deprecated---esmoduleinterop-false-and---allowsyntheticdefaultimports-false
- With interop on, a namespace import is an object and cannot be called. Two emitted helper functions shim CommonJS default imports. `importHelpers` makes the emit terser. https://www.typescriptlang.org/tsconfig/#esModuleInterop
- `import * as fs from "fs"` copies only own properties. For an API exposed through the prototype chain, use the default import. https://www.typescriptlang.org/tsconfig/#esModuleInterop
- `allowSyntheticDefaultImports` changes only type checking, never emit. https://www.typescriptlang.org/docs/handbook/modules/appendices/esm-cjs-interop.html#allowsyntheticdefaultimports-and-esmoduleinterop

## Import extension flags

- `allowImportingTsExtensions` allows `./a.ts` specifiers, only with `noEmit` or `emitDeclarationOnly`. The bundler or runtime resolves them. https://www.typescriptlang.org/tsconfig/#allowImportingTsExtensions
- To emit while writing `.ts` specifiers, use `rewriteRelativeImportExtensions`: relative `./x.ts`, `.tsx`, `.mts`, `.cts` become `.js`, `.jsx`, `.mjs`, `.cjs`. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-7.html#path-rewriting-for-relative-paths
- Not rewritten: extensionless specifiers, package paths, `#` imports, dynamic `import(getPath())`. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-7.html#path-rewriting-for-relative-paths
- Type-only imports may use `.ts` extensions without any flag. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-2.html#type-only-import-paths-with-typescript-implementation-file-extensions
- `allowArbitraryExtensions` types `./app.css` from `app.d.css.ts`. The older `app.css.d.ts` describes a file named `app.css.js` and fails in ESM under `node16` and `nodenext`. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-0.html#allowarbitraryextensions

```ts
// app.d.css.ts
declare const css: { cookieBanner: string };
export default css;
```

https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-0.html#allowarbitraryextensions

## Side-effect imports, JSON, module detection

- `noUncheckedSideEffectImports` is `true` in 6.0: an unresolvable `import "x"` errors. Declare assets with `declare module "*.css" {}` in a global `.d.ts`. https://www.typescriptlang.org/tsconfig/#noUncheckedSideEffectImports
- `resolveJsonModule` allows importing `.json` files and types the import from the static JSON shape. https://www.typescriptlang.org/tsconfig/#resolveJsonModule
- `module: preserve` implicitly sets `moduleResolution: bundler`, `esModuleInterop` and `resolveJsonModule`. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-4.html#support-for-require-in---moduleresolution-bundler-and---module-preserve
- Under `nodenext`, a JSON import needs a `type: "json"` import attribute. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-7.html#validated-json-imports-in---module-nodenext
- `moduleDetection: "auto"` (default) treats a file as a module when it has `import` or `export`, when `"type": "module"` applies under `node16` or `nodenext`, or when it is JSX under `react-jsx`. `"force"` makes every non-declaration file a module. https://www.typescriptlang.org/tsconfig/#moduleDetection

## Class fields

- With `target` ES2022 or newer, or `useDefineForClassFields: true`, fields follow standard define semantics and are initialized after the parent constructor, overwriting what it set. Redeclare an inherited field for typing only with `declare`. https://www.typescriptlang.org/docs/handbook/2/classes.html#type-only-field-declarations
