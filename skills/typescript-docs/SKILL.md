---
name: typescript-docs
description: Configures, writes and upgrades TypeScript 6.0 code and tsconfig.json from the official TypeScript documentation. Use when upgrading from TypeScript 5.x to 6.0 (strict, types, rootDir, baseUrl, moduleResolution node, target es5, outFile, esModuleInterop), choosing module and moduleResolution, writing package.json exports and imports, fixing ESM and CommonJS interop, enabling verbatimModuleSyntax or erasableSyntaxOnly, setting up project references and tsc --build, writing .d.ts files, checking JavaScript with JSDoc, or debugging narrowing, generics, mapped types, decorators and enums. Triggers on TypeScript, tsc, tsconfig, TS5112, ignoreDeprecations, nodenext, bundler, .d.ts, @ts-check, TypeScript 6 upgrade.
metadata:
  version: "6.0"
  source: https://www.typescriptlang.org/docs/
  read: "2026-11-02"
---

# TypeScript 6.0

## Overview

This skill holds TypeScript 6.0 rules that are easy to get wrong: changed defaults, options deprecated on the way to 7.0, and module settings that must match the runtime. Every claim carries its page URL.

## Version covered

TypeScript 6.0, the release the site documents. 6.0 prepares TypeScript 7.0, which removes every option deprecated in 6.0. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html#preparing-for-typescript-70

Where an older page contradicts the 6.0 release notes, the skill states the 6.0 behavior. Read the quick start, then open the one reference file that matches the task.

## Quick start

- 6.0 defaults: `strict: true`, `module: esnext`, `target` the newest ES version (`es2025`), `types: []`, `rootDir` the tsconfig directory, `noUncheckedSideEffectImports: true`, `libReplacement: false`. Write the old value explicitly to keep 5.x behavior. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html#simple-default-changes
- List globals in `types`. `Cannot find name 'process'` or `'describe'` after an upgrade means a missing entry. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html#types-now-defaults-to-

```json
{ "compilerOptions": { "types": ["node", "jest"], "rootDir": "./src" } }
```

https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html#up-front-adjustments

- Pick `module` for the host. Node.js app: `"module": "nodenext"`. Code that is bundled: `"module": "preserve"` or `esnext` with `"moduleResolution": "bundler"`. npm library: `"module": "node18"`. https://www.typescriptlang.org/docs/handbook/modules/guides/choosing-compiler-options.html and https://www.typescriptlang.org/tsconfig/#module
- Write relative imports with the runtime extension: `import { x } from "./math.js"` for `math.ts`. Specifiers are never rewritten, unless `rewriteRelativeImportExtensions` applies to `.ts` specifiers. https://www.typescriptlang.org/docs/handbook/modules/theory.html#module-specifiers-are-not-transformed-by-default
- `paths` never changes emitted imports. Drop `baseUrl` (deprecated) and prefix each `paths` entry with `./`. Package.json `"imports"` is a standard replacement for convenience `paths` aliases. https://www.typescriptlang.org/docs/handbook/modules/reference.html#paths-does-not-affect-emit
- Set `verbatimModuleSyntax`: imports with the `type` modifier are dropped entirely. https://www.typescriptlang.org/tsconfig/#verbatimModuleSyntax
- Add `erasableSyntaxOnly` so syntax with runtime semantics is an error, for Node.js type stripping. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-8.html#the---erasablesyntaxonly-option
- `exclude` only filters `include`: an imported file still compiles. `extends`: the inheriting `files`, `include`, `exclude` overwrite the base ones, and `references` is never inherited. https://www.typescriptlang.org/tsconfig/#exclude and https://www.typescriptlang.org/tsconfig/#extends
- `tsc file.ts` beside a `tsconfig.json` is error TS5112. Run `tsc -p` for the project or `tsc --ignoreConfig file.ts`. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html#specifying-command-line-files-when-tsconfigjson-exists-is-now-an-error and https://www.typescriptlang.org/docs/handbook/tsconfig-json.html
- `ignoreDeprecations: "6.0"` is a stopgap: fix the deprecation instead. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html#preparing-for-typescript-70
- Vite and Babel only transpile, they do not type check: run `tsc` separately. https://www.typescriptlang.org/docs/handbook/integrating-with-build-tools.html and https://www.typescriptlang.org/docs/handbook/babel-with-typescript.html
- An `as const` object may replace an enum. https://www.typescriptlang.org/docs/handbook/enums.html

## Reference files

| File | Open it for |
| ---- | ----------- |
| [references/upgrading-to-6.md](references/upgrading-to-6.md) | Moving a tsconfig to 6.0, deprecated options, TS5112, preparing for 7.0 |
| [references/release-changes.md](references/release-changes.md) | New errors after upgrading from any 5.x version |
| [references/tsconfig-project.md](references/tsconfig-project.md) | `extends`, `include`, `rootDir`, `types`, `target`, `composite`, project references, `tsc -b` |
| [references/compiler-checks.md](references/compiler-checks.md) | `strict` family, `verbatimModuleSyntax`, `isolatedModules`, `erasableSyntaxOnly`, interop flags |
| [references/modules-choosing.md](references/modules-choosing.md) | Picking `module` and `moduleResolution` per scenario |
| [references/modules-packages-interop.md](references/modules-packages-interop.md) | package.json `exports` and `imports`, `paths`, ESM and CJS interop |
| [references/narrowing-and-functions.md](references/narrowing-and-functions.md) | Narrowing, overloads, `this` parameters, generic functions |
| [references/objects-and-classes.md](references/objects-and-classes.md) | Index signatures, tuples, classes, declaration merging, mixins |
| [references/type-level.md](references/type-level.md) | Conditional, mapped and template literal types, utility types |
| [references/language-features.md](references/language-features.md) | Enums, namespaces, triple-slash directives, `using`, symbols |
| [references/decorators.md](references/decorators.md) | Standard and legacy decorators |
| [references/jsx.md](references/jsx.md) | `jsx` modes, the `JSX` namespace |
| [references/declaration-files.md](references/declaration-files.md) | Writing and publishing `.d.ts` files |
| [references/javascript-jsdoc.md](references/javascript-jsdoc.md) | `checkJs`, JSDoc tags, JS to TS migration |

## Old patterns

| Old usage | Replacement | Source |
| --------- | ----------- | ------ |
| `"baseUrl": "./src"` with `paths` | `paths` entries prefixed `./src/` | https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html#deprecated---baseurl |
| `"moduleResolution": "node"` or `"node10"` | `nodenext` (Node.js) or `bundler` | https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html#deprecated---moduleresolution-node-aka---moduleresolution-node10 |
| `"moduleResolution": "classic"` | `nodenext` or `bundler` | https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html#deprecated---moduleresolution-classic |
| `"target": "es5"`, `downlevelIteration` | `es2015` or newer, remove `downlevelIteration` | https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html#deprecated-target-es5 |
| `"module": "amd"`, `"umd"`, `"system"`, `"none"` | An ES module target and a bundler | https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html#deprecated-amd-umd-and-systemjs-values-of-module |
| `outFile` | An external bundler | https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html#deprecated-outfile |
| `"esModuleInterop": false`, `import * as express from "express"` | Default import `import express from "express"` | https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html#deprecated---esmoduleinterop-false-and---allowsyntheticdefaultimports-false |
| `"alwaysStrict": false` | Strict-mode code, rename reserved identifiers | https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html#deprecated---alwaysstrict-false |
| `module Foo {}` | `namespace Foo {}` | https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html#deprecated-legacy-module-syntax-for-namespaces |
| `import x from "./a.json" assert { type: "json" }` | `with { type: "json" }` | https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html#deprecated-asserts-keyword-on-imports |
| `/// <reference no-default-lib="true"/>` | `noLib` or `libReplacement` | https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html#deprecated-no-default-lib-directives |
| `"lib": ["dom", "dom.iterable"]` | `"lib": ["dom"]` | https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html#the-dom-lib-now-contains-domiterable-and-domasynciterable |
| `importsNotUsedAsValues`, `preserveValueImports` | `verbatimModuleSyntax` | https://www.typescriptlang.org/tsconfig/#verbatimModuleSyntax |
| `out`, `charset`, `target: ES3`, `prepend`, `suppressImplicitAnyIndexErrors` | No effect since 5.5, an error in 6.0: delete them | https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-5.html#disabling-features-deprecated-in-typescript-50 |
| `app.css.d.ts` | `app.d.css.ts` with `allowArbitraryExtensions` | https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-0.html#allowarbitraryextensions |
| `/// <amd-dependency path="x" />` | `import "x";` | https://www.typescriptlang.org/docs/handbook/triple-slash-directives.html |
| `@typedef {import("./x").T}` in JS | `/** @import { T } from "./x" */` | https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-5.html#the-jsdoc-import-tag |
