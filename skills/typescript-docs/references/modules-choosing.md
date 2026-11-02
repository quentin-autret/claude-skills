# Modules: choosing `module` and `moduleResolution`

## Contents

- Mental model
- Settings by scenario
- `module` values
- Implied and enforced options
- Module format detection under Node modes
- `moduleResolution` modes

## Mental model

- TypeScript models a host (Node.js, a bundler, a TS-first runtime). `module` and `moduleResolution` describe that host: the system outside TypeScript that consumes the output code (or the `.ts` files directly) and that TypeScript's module analysis tries to model. https://www.typescriptlang.org/docs/handbook/modules/theory.html#who-is-the-host
- TypeScript reasons about the output files: it checks that imports in the emitted JavaScript resolve at runtime. https://www.typescriptlang.org/docs/handbook/modules/theory.html#typescripts-job-concerning-modules
- Module specifiers are emitted as written, so write the output extension: a relative import of `math.mts` is written `./math.mjs`. TypeScript resolves the runtime extension back to the source file. https://www.typescriptlang.org/docs/handbook/modules/theory.html#module-specifiers-are-not-transformed-by-default
- `module` matters with `noEmit` too: it decides format detection, interop rules, `import.meta` and top-level `await`. https://www.typescriptlang.org/docs/handbook/modules/theory.html#the-module-output-format
- In 6.0, `module` defaults to `esnext` and `classic` resolution is removed. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html#simple-default-changes

## Settings by scenario

### App compiled by tsc and run in Node.js

- `"module": "nodenext"` (implies `moduleResolution: nodenext`, `esModuleInterop`, `target: esnext`), plus `"verbatimModuleSyntax": true`. Emit ESM with `"type": "module"` in package.json or `.mts` files. https://www.typescriptlang.org/docs/handbook/modules/guides/choosing-compiler-options.html#im-compiling-and-running-the-outputs-in-nodejs
- `node16`, `node18`, `node20` and `nodenext` are the only correct values for code run by Node.js, even when all files share one format. `esnext` or `commonjs` do not model Node's format detection and interop. https://www.typescriptlang.org/docs/handbook/modules/reference.html#node16-node18-node20-nodenext
- `nodenext` follows the latest Node.js and allows `require()` of ESM. `node20` is a stable Node 20 model (implies `target: es2023`). `node18` disallows `require()` of ESM and allows import assertions. https://www.typescriptlang.org/docs/handbook/modules/reference.html#node16-node18-node20-nodenext and https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-9.html#support-for---module-node20

### App with a bundler, or tsx

```json5
{ "compilerOptions": {
  "module": "esnext", "moduleResolution": "bundler", "esModuleInterop": true,
  "customConditions": ["module"],   // consult the bundler docs
  "noEmit": true,                   // or emitDeclarationOnly
  "allowImportingTsExtensions": true, "allowArbitraryExtensions": true,
  "verbatimModuleSyntax": true      // or isolatedModules
} }
```

https://www.typescriptlang.org/docs/handbook/modules/guides/choosing-compiler-options.html#im-using-a-bundler

- Do not set `"type": "module"` or use `.mts` in a bundler project: some bundlers change interop then, and `bundler` resolution cannot model it. https://www.typescriptlang.org/docs/handbook/modules/guides/choosing-compiler-options.html#im-using-a-bundler
- `module: preserve` fits a bundler or runtime that reads raw `.ts` (Bun included). It keeps each statement as written, emits `import x = require()` as `require`, and implies `bundler`, `esModuleInterop` and `resolveJsonModule`. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-4.html#support-for-require-calls-in---moduleresolution-bundler-and---module-preserve
- tsx behaves like a bundler: use bundler settings. ts-node keeps Node semantics: use Node settings. https://www.typescriptlang.org/docs/handbook/modules/guides/choosing-compiler-options.html#im-using-tsx

### Library published to npm, compiled by tsc

```json5
{ "compilerOptions": {
  "module": "node18", "target": "es2020",   // lowest target you support
  "strict": true, "verbatimModuleSyntax": true,
  "declaration": true, "sourceMap": true, "declarationMap": true,
  "rootDir": "src", "outDir": "dist"
} }
```

https://www.typescriptlang.org/docs/handbook/modules/guides/choosing-compiler-options.html#im-writing-a-library

- `node18` because code that works in Node.js almost always works in bundlers, not the reverse. `bundler` resolution is infectious: `export * from "./utils"` type checks and fails in Node.js with ERR_MODULE_NOT_FOUND. https://www.typescriptlang.org/docs/handbook/modules/theory.html#module-resolution-for-libraries
- A separate `outDir` is required when the package ships `.ts` sources, or consumers load `.ts` instead of `.d.ts`. https://www.typescriptlang.org/docs/handbook/modules/guides/choosing-compiler-options.html#im-writing-a-library

### Library bundled by a bundler

- One compilation cannot use two resolution modes. Bundle the declaration files too, since every `.d.ts` stands for exactly one JS file. Without bundled declarations, use `nodenext`. The docs suggest not bundling a library. https://www.typescriptlang.org/docs/handbook/modules/guides/choosing-compiler-options.html#considerations-for-bundling-libraries
- Dual ESM and CJS emit from one compilation leaves one output unchecked and is incompatible with `verbatimModuleSyntax`. Test every output and run `@arethetypeswrong/cli` before publishing. https://www.typescriptlang.org/docs/handbook/modules/guides/choosing-compiler-options.html#notes-on-dual-emit-solutions

### Browser ESM without a bundler

- No dedicated option. Approximate with `module: nodenext`, `"type": "module"` and `paths` mapping URLs to local `.d.ts`. https://www.typescriptlang.org/docs/handbook/modules/guides/choosing-compiler-options.html#im-writing-es-modules-for-the-browser-with-no-bundler-or-module-compiler

## `module` values

| Value | Use |
| ----- | --- |
| `nodenext`, `node20`, `node18`, `node16` | Code run by Node.js. Emit CJS or ESM per file |
| `preserve` | Bundlers and runtimes reading `.ts` |
| `esnext`, `es2022`, `es2020` | Bundled code. `es2020` adds `import.meta`, `es2022` top-level `await` |
| `commonjs` | Not recommended: use a Node mode to emit CJS for Node |
| `amd`, `umd`, `systemjs`, `none` | No longer supported in 6.0 |

https://www.typescriptlang.org/docs/handbook/modules/reference.html#node16-node18-node20-nodenext and https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html#deprecated-amd-umd-and-systemjs-values-of-module

- Under `es*` values, `import mod = require("mod")` is not allowed in non-declaration files. https://www.typescriptlang.org/docs/handbook/modules/reference.html#es2015-es2020-es2022-esnext
- `import defer` works only under `preserve` and `esnext`, never downleveled. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-9.html#support-for-import-defer

## Implied and enforced options

- `nodenext` implies `moduleResolution: nodenext` and `target: esnext`. `node16` and `node18` imply `moduleResolution: node16` and `target: es2022`. All Node modes and `preserve` imply `esModuleInterop`. https://www.typescriptlang.org/docs/handbook/modules/reference.html#implied-and-enforced-options
- When either option is `node16` or `nodenext`, the other must be Node-flavored too. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-2.html#module-and-moduleresolution-must-match-under-recent-nodejs-settings
- The 5.x pages say `bundler` needs `module: esnext` or `preserve`. In 6.0, `bundler` also accepts `module: commonjs`. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html#combining---moduleresolution-bundler-with---module-commonjs

## Module format detection under Node modes

- `.mts`, `.mjs`, `.d.mts` are ESM. `.cts`, `.cjs`, `.d.cts` are CJS. `.ts`, `.tsx`, `.js`, `.d.ts` are ESM only if the nearest `package.json` has `"type": "module"`. A project of plain `.ts` files without it emits CJS. https://www.typescriptlang.org/docs/handbook/modules/reference.html#module-format-detection
- The nearest `package.json` decides the format of `.ts`, `.js` and `.d.ts` files, so a nested `package.json` can change it. Dependency `.d.ts` files are classified the same way. https://www.typescriptlang.org/docs/handbook/modules/theory.html#module-format-detection
- In a CJS-detected file, `import` compiles to `require` and resolves with the `require` algorithm. `verbatimModuleSyntax` forbids such imports. https://www.typescriptlang.org/docs/handbook/modules/reference.html#node16-nodenext-1

## `moduleResolution` modes

- `node16` and `nodenext`: ESM imports need extensions and do not resolve directory `index` files. Both support `exports`, `imports` and `typesVersions`. https://www.typescriptlang.org/docs/handbook/modules/reference.html#node16-nodenext-1
- `bundler`: supports `exports` and `imports` and never requires extensions on relative imports. Right for bundlers and Bun, wrong for a library consumed without a bundler. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-0.html#--moduleresolution-bundler
- `node10` (formerly `node`) ignores `exports` and is deprecated in 6.0. `classic` is removed. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html#deprecated---moduleresolution-node-aka---moduleresolution-node10
- TypeScript never resolves an omitted `.mjs`, `.mts`, `.cjs` or `.cts` extension. https://www.typescriptlang.org/docs/handbook/modules/reference.html#extensionless-relative-paths
- Dynamic `import()` always uses the `import` algorithm, but `typeof import("./mod")` follows the importing file's format. https://www.typescriptlang.org/docs/handbook/modules/reference.html#node16-nodenext-1
