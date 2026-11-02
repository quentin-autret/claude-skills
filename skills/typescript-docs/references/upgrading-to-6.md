# Upgrading to TypeScript 6.0

## Contents

- What 6.0 is
- Changed defaults
- `types` defaults to `[]`
- `rootDir` defaults to the tsconfig directory
- Deprecated or removed options and their replacement
- Removing `baseUrl`
- Command-line and resolution changes
- Preparing for 7.0

## What 6.0 is

- 6.0 is a transition release toward TypeScript 7.0 (native port of the compiler). It stays API compatible with 5.9 but brings breaking changes and deprecations. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html#breaking-changes-and-deprecations-in-typescript-60
- Some deprecations and behavior changes produce errors that do not point at the real cause. The two most common symptoms are missing globals (`types`) and a shifted output tree (`rootDir`). https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html#up-front-adjustments
- The experimental `ts5to6` tool (https://github.com/andrewbranch/ts5to6) adjusts `baseUrl` and `rootDir` across a codebase. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html#breaking-changes-and-deprecations-in-typescript-60

## Changed defaults

| Option | 6.0 default | 5.x default |
| ------ | ----------- | ----------- |
| `strict` | `true` | `false` |
| `module` | `esnext` | not stated in the 6.0 notes |
| `target` | latest supported ES version (floating, `es2025` in 6.0) | `es5` per the older handbook |
| `types` | `[]` | every visible `@types` package |
| `rootDir` | directory of the `tsconfig.json` | longest common path of the inputs |
| `noUncheckedSideEffectImports` | `true` | `false` |
| `libReplacement` | `false` | `true` |

If a new default breaks the project, write the previous value explicitly in tsconfig, for example `"strict": false`. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html#simple-default-changes

- The handbook Basics page still says the default target is ES5 and the default experience is lenient. The 6.0 release notes win. https://www.typescriptlang.org/docs/handbook/2/basic-types.html#downleveling
- `noUncheckedSideEffectImports` on by default makes `import "./button.css"` fail when nothing resolves it. Declare the assets once in a global `.d.ts`. https://www.typescriptlang.org/tsconfig/#noUncheckedSideEffectImports

```ts
// ./src/globals.d.ts
declare module "*.css" {}
```

https://www.typescriptlang.org/tsconfig/#noUncheckedSideEffectImports

- A project that uses `@typescript/lib-*` replacement packages sets `"libReplacement": true`. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-8.html#the---libreplacement-flag

## `types` defaults to `[]`

- No `@types` package is included in the global scope unless listed. `process`, `describe`, `expect` and other globals disappear. Imports of typed packages keep their types. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html#types-now-defaults-to-
- Symptoms: `Cannot find name 'process'. Do you need to install type definitions for node?`, `Cannot find name 'describe'`, `Cannot find module '...' or its corresponding type declarations.` https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html#types-now-defaults-to-
- List the packages that provide globals. `"types": ["*"]` restores the 5.9 behavior but an explicit array is recommended for build speed and predictability. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html#up-front-adjustments

```json
{ "compilerOptions": { "types": ["node", "jest"] } }
```

https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html#types-now-defaults-to-

- The `typeRoots` page still describe the old "all visible `@types`" behavior. The `types` page and the 6.0 notes are the authority. https://www.typescriptlang.org/tsconfig/#types

## `rootDir` defaults to the tsconfig directory

- `rootDir` is the directory containing the `tsconfig.json`. It is inferred only when `tsc` runs on command-line files without a tsconfig. Symptom: output lands in `./dist/src/index.js` instead of `./dist/index.js`. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html#rootdir-now-defaults-to-

```json
{ "compilerOptions": { "rootDir": "./src" }, "include": ["./src"] }
```

https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html#rootdir-now-defaults-to-

- A tsconfig that includes files outside its own directory sets `rootDir` to cover them, for example `"rootDir": "../src"`. Same URL.

## Deprecated or removed options and their replacement

`"ignoreDeprecations": "6.0"` silences the deprecations in 6.0 only. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html#preparing-for-typescript-70

| Deprecated or removed | Replacement |
| --------------------- | ----------- |
| `target: es5` | ES2015 or newer, or post-process with another compiler |
| `downlevelIteration` (any value, even `false`) | Remove it, it only affected ES5 emit |
| `moduleResolution: node` (`node10`) | `nodenext` for Node.js, `bundler` for a bundler or Bun |
| `moduleResolution: classic` | Removed. `nodenext` or `bundler` |
| `module: amd`, `umd`, `systemjs`, `none` | An ES module target plus a bundler, or stay on 5.x. `amd-module` has no effect |
| `baseUrl` | Remove it and prefix `paths` entries |
| `esModuleInterop: false`, `allowSyntheticDefaultImports: false` | Always on, cannot be `false` |
| `alwaysStrict: false` | All code is JS strict mode. Rename identifiers such as `await`, `static`, `private`, `public` |
| `outFile` | Removed. An external bundler |
| `module Foo {}` | `namespace Foo {}` (error). `declare module "x" {}` still works |
| `import ... asserts { type: "json" }` | `with { type: "json" }` |
| `/// <reference no-default-lib="true"/>` | `noLib` or `libReplacement` |

Source for every row: https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html#breaking-changes-and-deprecations-in-typescript-60

- With `esModuleInterop` always on, a namespace import of a CommonJS callable becomes a default import. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html#deprecated---esmoduleinterop-false-and---allowsyntheticdefaultimports-false

```ts
// Before (esModuleInterop: false)
import * as express from "express";
// After
import express from "express";
```

https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html#deprecated---esmoduleinterop-false-and---allowsyntheticdefaultimports-false

- Options deprecated in 5.0 (`charset`, `target: ES3`, `importsNotUsedAsValues`, `preserveValueImports`, `noImplicitUseStrict`, `noStrictGenericChecks`, `keyofStringsOnly`, `suppressExcessPropertyErrors`, `suppressImplicitAnyIndexErrors`, `out`, `prepend` in project references, implicit OS `newLine`) have had no effect since 5.5 and are an error to specify in 6.0. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-5.html#disabling-features-deprecated-in-typescript-50

## Removing `baseUrl`

- With `baseUrl: "./src"`, `import "someModule.js"` resolves to `src/someModule.js` even when only aliases were intended, so imports that fail at runtime pass type checking. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html#deprecated---baseurl

```diff
-     "baseUrl": "./src",
      "paths": {
-       "@app/*": ["app/*"],
-       "@lib/*": ["lib/*"]
+       "@app/*": ["./src/app/*"],
+       "@lib/*": ["./src/lib/*"]
      }
```

https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html#deprecated---baseurl

- A project that used `baseUrl` as a real lookup root adds a catch-all `"*": ["./src/*"]` to `paths`. Same URL.

## Command-line and resolution changes

- `tsc foo.ts` in a directory with a `tsconfig.json` is an error: `error TS5112: tsconfig.json is present but will not be loaded if files are specified on commandline`. Run `tsc --ignoreConfig foo.ts` to compile with defaults. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html#specifying-command-line-files-when-tsconfigjson-exists-is-now-an-error
- `moduleResolution: bundler` combines with `module: commonjs`, often the most suitable upgrade path off `moduleResolution: node`. Long term: `module: preserve` with `bundler` (bundled app, Bun) or `module: nodenext` (Node.js app). https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html#combining---moduleresolution-bundler-with---module-commonjs
- Subpath imports may start with `#/` under `nodenext` and `bundler`. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html#subpath-imports-starting-with-

```json
{ "type": "module", "imports": { "#/*": "./dist/*" } }
```

https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html#subpath-imports-starting-with-

- `lib: ["dom"]` includes `dom.iterable` and `dom.asynciterable`. Simplify `["dom", "dom.iterable"]` to `["dom"]`. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html#the-dom-lib-now-contains-domiterable-and-domasynciterable

## Preparing for 7.0

- TypeScript 7.0 removes every option deprecated in 6.0, so `ignoreDeprecations` is a stopgap. Fix the warnings before trying 7.0 or `@typescript/native-preview`. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html#preparing-for-typescript-70
- Nightly builds install with `npm install -D typescript@next`. Point the editor at the workspace version with `"typescript.tsdk": "<path>/node_modules/typescript/lib"`. https://www.typescriptlang.org/docs/handbook/nightly-builds.html
