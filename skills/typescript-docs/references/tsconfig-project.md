# tsconfig and project setup

## Contents

- Finding the config and the starter config
- `extends`
- `files`, `include`, `exclude`
- `rootDir` and `outDir`
- `types` and `typeRoots`
- `target` and `lib`
- `skipLibCheck`
- Declarations, `composite`, `incremental`
- `noEmit` and `noCheck`
- Project references and `tsc -b`
- Watch mode
- Build tools

## Finding the config and the starter config

- `tsc` with no input files searches for `tsconfig.json` from the current directory up the parent chain. `-p` takes a directory or any `.json` path. https://www.typescriptlang.org/docs/handbook/tsconfig-json.html
- In 6.0, `tsc foo.ts` next to a `tsconfig.json` is error TS5112. `tsc --ignoreConfig foo.ts` compiles with defaults, never with the project's options. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html#specifying-command-line-files-when-tsconfigjson-exists-is-now-an-error
- Since 5.9, `tsc --init` writes a short prescriptive config: `module: nodenext`, `target: esnext`, `types: []`, `sourceMap`, `declaration`, `declarationMap`, `strict`, `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`, `verbatimModuleSyntax`, `isolatedModules`, `noUncheckedSideEffectImports`, `moduleDetection: force`, `skipLibCheck`, `jsx: react-jsx`. For Node.js add `"types": ["node"]` and install `@types/node`. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-9.html#minimal-and-updated-tsc---init

## `extends`

- Node.js-style resolution: `./configs/base` without `.json` works. Circular `extends` is an error. https://www.typescriptlang.org/tsconfig/#extends
- A package path such as `@tsconfig/node12/tsconfig.json` can serve as a base. https://www.typescriptlang.org/docs/handbook/tsconfig-json.html
- `extends` accepts an array. Later entries win on conflict. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-0.html#supporting-multiple-configuration-files-in-extends
- `files`, `include` and `exclude` in the child replace the base's, they never merge. `references` is never inherited. https://www.typescriptlang.org/tsconfig/#extends
- Relative paths resolve against the file that wrote them. In a shared base, write `${configDir}` so the path resolves against the extending config. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-5.html#the-configdir-template-variable-for-configuration-files

```json5
{
    "compilerOptions": {
        "typeRoots": ["${configDir}/node_modules/@types", "${configDir}/custom-types"],
        "outDir": "${configDir}/dist"
    }
}
```

https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-5.html#the-configdir-template-variable-for-configuration-files

## `files`, `include`, `exclude`

- `files` lists exact names, no globs, and errors on a missing file. `include` takes globs relative to the tsconfig directory: `*`, `?`, `**/`. https://www.typescriptlang.org/tsconfig/#include
- A last segment with no extension and no wildcard is a directory: `.ts`, `.tsx`, `.d.ts` are included, plus `.js` and `.jsx` with `allowJs`. https://www.typescriptlang.org/tsconfig/#include
- `exclude` only filters what `include` found. A file reached by an `import`, a `types` entry, a `/// <reference>` or `files` still enters the program. It is not a mechanism that prevents a file from being included. https://www.typescriptlang.org/tsconfig/#exclude

## `rootDir` and `outDir`

- `outDir` receives every emitted file and keeps the source tree shape. Without it, `.js` lands next to each `.ts`. https://www.typescriptlang.org/tsconfig/#outDir
- In 6.0, `rootDir` defaults to the directory containing the `tsconfig.json`. The option page still describes the older inference (longest common path of non-declaration inputs). Set `rootDir` explicitly when sources sit in `src/`. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html#rootdir-now-defaults-to-
- `rootDir` never selects files. Every emitted file must sit under it, so `rootDir: "core"` with a root-level `helpers.ts` included is an error. https://www.typescriptlang.org/tsconfig/#rootDir

## `types` and `typeRoots`

- In 6.0, `types` defaults to `[]`: list the packages that provide globals, such as `"types": ["node", "jest"]`. A package left out adds no globals and no auto-import suggestions, but importing it stays fully typed. https://www.typescriptlang.org/tsconfig/#types
- `types` names packages, `typeRoots` names folders. A set `typeRoots` includes only those roots and every package in them, and drops `./node_modules/@types` unless listed. https://www.typescriptlang.org/tsconfig/#typeRoots
- A failed lookup through `typeRoots` does not walk up to parent `node_modules/@types`. In a monorepo, list `../../node_modules/@types` too. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-1.html#explicit-typeroots-disables-upward-walks-for-node_modulestypes

## `target` and `lib`

- `target` sets the emitted language level and the default `lib`. In 6.0 it defaults to the newest supported version and `es5` is deprecated (lowest is ES2015). https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html#simple-default-changes and https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html#deprecated-target-es5
- In 6.0, `"dom"` already contains `dom.iterable` and `dom.asynciterable`, which are now empty files. Listing `"dom"` alone is enough. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html#the-dom-lib-now-contains-domiterable-and-domasynciterable

## `skipLibCheck`

- It skips type checking of declaration files (all `.d.ts`). Do not use it to hide two disagreeing copies of a library's types: deduplicate the dependency. A legitimate use is a temporary bridge while upgrading TypeScript. https://www.typescriptlang.org/tsconfig/#skipLibCheck

## Declarations, `composite`, `incremental`

- For `.d.ts` from JS files, use `emitDeclarationOnly` or `outDir` so sources are not overwritten. https://www.typescriptlang.org/tsconfig/#declaration
- `composite` makes `declaration` default to `true`, `rootDir` default to the tsconfig directory, and requires every implementation file to be matched by `include` or `files`. https://www.typescriptlang.org/tsconfig/#composite
- Since 5.6, `tsc --build` always writes a `.tsbuildinfo` file for each project, even without `incremental`. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-6.html#tsbuildinfo-is-always-written
- `isolatedDeclarations` (needs `declaration` or `composite`) errors when an exported API is not annotated enough for a tool to emit `.d.ts` without a checker. Locals and trivial initializers (`export let x = 10`) need nothing. Adopt it only for fast external declaration emit or parallel builds, it gives no speed by itself. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-5.html#using-isolateddeclarations

## `noEmit` and `noCheck`

- `noEmit`: TypeScript only checks, another tool transpiles. `noCheck`: emit without full checking. Run `tsc --noCheck` for emit and `tsc --noEmit` for checks, with separate `tsBuildInfoFile` values if both run at once. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-6.html#the-nocheck-option
- Without `isolatedDeclarations`, `noCheck` TypeScript may still do as much checking as needed to emit `.d.ts`, lazily computing only the types of unannotated declarations. Same URL.

## Project references and `tsc -b`

- A referenced project must set `composite`. Importing from it reads its built `.d.ts`, not its sources. https://www.typescriptlang.org/docs/handbook/project-references.html
- Plain `tsc` never builds references. `tsc -b` builds out-of-date projects in order. Flags: `--verbose`, `--dry`, `--clean`, `--force`, `--watch`. https://www.typescriptlang.org/docs/handbook/project-references.html
- The project references page says `tsc -b` acts as if `noEmitOnError` were on. Since 5.6 it keeps building downstream projects after errors. Add `--stopOnBuildErrors` to stop at the first failing project. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-6.html#allow---build-with-intermediate-errors
- `--build` accepts emit flags such as `--declaration` and `--sourceMap`. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-0.html#passing-emit-specific-flags-under---build
- A solution tsconfig holds only `references` and `"files": []`. Without the empty `files`, files compile twice. https://www.typescriptlang.org/docs/handbook/project-references.html

```json
{ "files": [], "references": [ { "path": "./src" }, { "path": "./test" } ] }
```

https://www.typescriptlang.org/docs/handbook/project-references.html

- One tsconfig describes one environment. Server, DOM, worker and test code each get their own tsconfig linked by references. https://www.typescriptlang.org/docs/handbook/modules/guides/choosing-compiler-options.html#im-writing-an-app
- Add `declarationMap` for Go to Definition across projects. After cloning, build once or editors show spurious errors. https://www.typescriptlang.org/docs/handbook/project-references.html

## Watch mode

- Configure watching in `watchOptions` (`watchFile`, `watchDirectory`, `fallbackPolling`, `excludeDirectories`, `excludeFiles`). Linux has no native recursive directory watching. The page's example sets `"moduleResolution": "node"`, deprecated in 6.0: do not copy that line. https://www.typescriptlang.org/docs/handbook/configuring-watch.html

## Build tools

- Vite supports importing `.ts` files out of the box and only transpiles, with no type checking. https://www.typescriptlang.org/docs/handbook/integrating-with-build-tools.html
- Babel does not type check and cannot create `.d.ts` files. Run `tsc` separately for type checking, with `declaration`, `emitDeclarationOnly` and `isolatedModules` on. https://www.typescriptlang.org/docs/handbook/babel-with-typescript.html