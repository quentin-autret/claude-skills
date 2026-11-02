# Writing and publishing declaration files

The Modules Theory page covers how `.d.mts` and `.d.cts` files are interpreted: a `.d.mts` pairs with `.mjs` and a `.d.cts` with `.cjs`.

## Contents

- Before writing a declaration
- Identify the library kind
- Do's and don'ts
- Module templates
- Global templates
- Dependencies inside declarations
- Publishing

## Before writing a declaration

- Try the import first: the package may bundle types. Then look for `@types/<name>`. Write your own only when neither exists. https://www.typescriptlang.org/docs/handbook/2/type-declarations.html#bundled-types
- In 6.0, `@types` packages are no longer auto-included as globals: add global-providing packages to `types`. The Type Declarations page still says they are discovered automatically. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html#types-now-defaults-to-
- `declare module "some-untyped-module";` in a `.d.ts` silences an untyped import as `any`. https://www.typescriptlang.org/docs/handbook/2/type-declarations.html#your-own-definitions
- A `.d.ts` describes one sibling JS file exactly. Hand-written declarations that drift from the JS give users false errors. https://www.typescriptlang.org/docs/handbook/modules/theory.html#the-role-of-declaration-files

## Identify the library kind

- Modular library (CommonJS or ES module): `module.d.ts`, `module-function.d.ts` (callable), `module-class.d.ts` (constructable), `module-plugin.d.ts` (mutates another module). Global script library: `global.d.ts`. Library extending built-in prototypes: `global-plugin.d.ts`. Module that mutates globals on import: `global-modifying-module.d.ts`. https://www.typescriptlang.org/docs/handbook/declaration-files/library-structures.html and https://www.typescriptlang.org/docs/handbook/declaration-files/templates.html
- Most popular globally-accessible libraries are actually written as UMD. Tells: `typeof define`, `typeof window`, `typeof module` checks at the top, and docs showing both `require` and `<script>`. https://www.typescriptlang.org/docs/handbook/declaration-files/library-structures.html
- Do not declare many global types. Namespace them under the library's global (`declare namespace cats { interface KittySettings {} }`). https://www.typescriptlang.org/docs/handbook/declaration-files/library-structures.html

## Do's and don'ts

All from https://www.typescriptlang.org/docs/handbook/declaration-files/do-s-and-don-ts.html

- A callback whose result is ignored returns `void`, not `any`.
- Callback parameters are required unless the callback is really called with fewer arguments. Write one signature with the maximum callback arity, never overloads by arity.
- Order overloads specific first: TypeScript picks the first match, so an early general overload hides the later ones.
- Collapse overloads that differ only by trailing parameters into optional parameters, when the return type is the same.
- Use a union instead of overloads that differ by the type of one parameter.
- Never declare a generic type that does not use its type parameter.

```ts
declare function beforeAll(action: (done: DoneFn) => void, timeout?: number): void;
```

https://www.typescriptlang.org/docs/handbook/declaration-files/do-s-and-don-ts.html

## Module templates

- `.d.ts` syntax mirrors ES modules. A CommonJS module whose whole export is one value uses `export =`. https://www.typescriptlang.org/docs/handbook/declaration-files/templates/module-d-ts.html

```ts
declare function getArrayLength(arr: any[]): number;
declare namespace getArrayLength { const maxInterval: 12; }
export = getArrayLength;
```

https://www.typescriptlang.org/docs/handbook/declaration-files/templates/module-d-ts.html

- A callable or constructable module (`export = Greeter`) is imported with `import x = require("mod")`, or with a default import when `allowSyntheticDefaultImports` or `esModuleInterop` is on (6.0 always enables both, and they can no longer be set to false). The template warns that `import * as x` of a module declaring a namespace can wrongly import a namespace object unless `esModuleInterop` is on. https://www.typescriptlang.org/docs/handbook/declaration-files/templates/module-function-d-ts.html
- `export as namespace myLib;` also exposes a module as a UMD global. Remove it for non-UMD libraries. https://www.typescriptlang.org/docs/handbook/declaration-files/templates/module-function-d-ts.html
- The class template exports `Greeter` but names its namespace `MyClass` and declares `greet: void`, which look like inconsistencies in the template. Use the class name for the namespace and a method signature for `greet`. https://www.typescriptlang.org/docs/handbook/declaration-files/templates/module-class-d-ts.html
- A module plugin imports the base module and augments an exported interface. Expose the base callable through a named interface so plugins can extend it. ES module namespaces are immutable, so plugins adding top-level exports fail under ESM, unchecked by TypeScript. https://www.typescriptlang.org/docs/handbook/declaration-files/templates/module-plugin-d-ts.html
- To support every import style (`require`, named, default), the JS itself must export all of them, for example `fastify.default = fastify`. https://www.typescriptlang.org/docs/handbook/declaration-files/templates/module-d-ts.html
- The declaration tree mirrors the library's subpaths: `myLib/bar/baz` is `bar/baz.d.ts`. https://www.typescriptlang.org/docs/handbook/declaration-files/templates/module-d-ts.html
- A value and an interface may share a name. An interface merges into a class of the same name, never into a type alias. A namespace adds statics to a class. https://www.typescriptlang.org/docs/handbook/declaration-files/deep-dive.html

## Global templates

- A global library is `declare function` overloads plus a `declare namespace` holding its properties and types. https://www.typescriptlang.org/docs/handbook/declaration-files/templates/global-d-ts.html
- A global plugin re-opens the built-in interface (`interface Number { toBinaryString(): string; }`). https://www.typescriptlang.org/docs/handbook/declaration-files/templates/global-plugin-d-ts.html
- A global-modifying module uses `declare global` and keeps `export {}` when it exports nothing. https://www.typescriptlang.org/docs/handbook/declaration-files/templates/global-modifying-module-d-ts.html

```ts
declare global {
  interface String { fancyFormat(opts: StringFormatOptions): string; }
}
export interface StringFormatOptions { fancinessLevel: number; }
```

https://www.typescriptlang.org/docs/handbook/declaration-files/templates/global-modifying-module-d-ts.html

## Dependencies inside declarations

- Depend on a global library with `/// <reference types="someLib" />`, on a module or UMD library with `import`. Never `/// <reference path>` in a declaration file. https://www.typescriptlang.org/docs/handbook/declaration-files/library-structures.html
- Never copy a dependency's declarations into yours or combine them with yours. If it does not package its own declarations, depend on its npm type declaration package. https://www.typescriptlang.org/docs/handbook/declaration-files/publishing.html

## Publishing

- Generated declarations ship with the package: set `"types"` next to `"main"`. https://www.typescriptlang.org/docs/handbook/declaration-files/publishing.html
- `@types/*` packages your declarations expose go in `dependencies`, not `devDependencies`. https://www.typescriptlang.org/docs/handbook/declaration-files/publishing.html
- `typesVersions` ranges match in order: highest first, or the mapping does not work. https://www.typescriptlang.org/docs/handbook/declaration-files/publishing.html
- Generate declarations from typed JS with `allowJs`, `declaration`, `emitDeclarationOnly` and `outDir`. Test them with `tsd` or `TSTyche`. https://www.typescriptlang.org/docs/handbook/declaration-files/dts-from-js.html