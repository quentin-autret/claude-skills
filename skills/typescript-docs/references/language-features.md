# Enums, namespaces, directives, `using` and other runtime features

## Contents

- Enums
- `const enum`
- Namespaces
- Triple-slash directives
- `using` and `await using`
- Iterators and symbols
- Destructuring and spread

## Enums

- Enums emit runtime code. A `const` object with `as const` often replaces them and stays plain JavaScript. They are also banned by `erasableSyntaxOnly`. https://www.typescriptlang.org/docs/handbook/enums.html https://www.typescriptlang.org/tsconfig/#erasableSyntaxOnly

```ts
const ODirection = { Up: 0, Down: 1, Left: 2, Right: 3 } as const;
type Direction = typeof ODirection[keyof typeof ODirection];
```

https://www.typescriptlang.org/docs/handbook/enums.html

- When an enum is used, prefer string members for values read at runtime or serialized: string members serialize readably, and the page sees no reason to mix string and number members. https://www.typescriptlang.org/docs/handbook/enums.html
- Numeric members get a reverse mapping (`Enum[Enum.A] === "A"`), string members do not. https://www.typescriptlang.org/docs/handbook/enums.html
- A member after a computed member needs an initializer (error 1061). In a `declare enum`, a member without initializer is always computed. https://www.typescriptlang.org/docs/handbook/enums.html
- Since 5.0 every enum is a union enum, so members work as types and can be narrowed. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-0.html#all-enums-are-union-enums
- Since 5.4, same-named enums from two declarations are compatible only when their known values are identical. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-4.html#new-enum-assignability-restrictions

## `const enum`

- Members are inlined and the declaration erased. Ambient `const enum` in a published `.d.ts` breaks consumers using `isolatedModules`, and inlines values from the build-time version of a dependency. https://www.typescriptlang.org/docs/handbook/enums.html
- Ways out: ban `const enum` with a linter, or publish with `preserveConstEnums` and strip `const` from the `.d.ts`. Inlining a project's own enums is fine. https://www.typescriptlang.org/docs/handbook/enums.html

## Namespaces

- Modules are the recommended organization for new code. Do not wrap a module's exports in `export namespace`: consumers then write `shapes.Shapes.Triangle`. https://www.typescriptlang.org/docs/handbook/namespaces-and-modules.html
- Write `namespace X {}`. The legacy `module X {}` form is an error in 6.0. `declare module "x" {}` for ambient modules is unaffected. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html#deprecated-legacy-module-syntax-for-namespaces
- The namespace pages describe concatenating multi-file namespaces with `outFile`, which 6.0 removed. Use a bundler. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html#deprecated-outfile
- `import q = x.y.z` aliases a namespace member. A value alias is a separate binding. It is banned under `erasableSyntaxOnly`. https://www.typescriptlang.org/docs/handbook/namespaces.html https://www.typescriptlang.org/tsconfig/#erasableSyntaxOnly
- Do not `/// <reference>` a module file: `import` it. https://www.typescriptlang.org/docs/handbook/namespaces-and-modules.html

## Triple-slash directives

- Valid only at the top of a file, before any statement. https://www.typescriptlang.org/docs/handbook/triple-slash-directives.html
- Declaration emit keeps handwritten directives only with `preserve="true"`. https://www.typescriptlang.org/docs/handbook/triple-slash-directives.html

```ts
/// <reference types="some-lib" preserve="true" />
```

https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-5.html#simplified-reference-directive-declaration-emit

- In `.ts` sources, declare `@types` globals through `types` in tsconfig rather than `/// <reference types>`. https://www.typescriptlang.org/docs/handbook/triple-slash-directives.html
- `/// <reference lib="es2017.string" />` pulls one built-in lib, the right tool for declaration authors. https://www.typescriptlang.org/docs/handbook/triple-slash-directives.html
- `/// <reference no-default-lib="true"/>` is no longer supported in 6.0: use `noLib` or `libReplacement`. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html#deprecated-no-default-lib-directives
- `amd-module` has no effect in 6.0. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html#deprecated-amd-umd-and-systemjs-values-of-module
- `amd-dependency` is deprecated: use `import "mod"`. https://www.typescriptlang.org/docs/handbook/triple-slash-directives.html

## `using` and `await using`

- `using x = ...` calls `x[Symbol.dispose]()` when the scope ends, including right before an early `return` or a thrown error, in first-in-last-out order. `null` and `undefined` are allowed. Destructuring is not. https://www.typescriptlang.org/docs/handbook/variable-declarations.html https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-2.html#using-declarations-and-explicit-resource-management

```ts
export function doSomeWork() {
    using file = new TempFile(".some_temp_file");
    if (someCondition()) { return; }
}
```

https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-2.html#using-declarations-and-explicit-resource-management

- `await using` awaits the disposal, not the value: write `await using y = await getResource()`. It is valid only in an `async` function or at module top level. https://www.typescriptlang.org/docs/handbook/variable-declarations.html
- With `await using` in scope, `return g()` without `await` can surface as an unhandled rejection. Write `return await g()`. https://www.typescriptlang.org/docs/handbook/variable-declarations.html
- If the body and the disposal both throw, a `SuppressedError` carries `.error` (disposal) and `.suppressed` (body). https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-2.html#using-declarations-and-explicit-resource-management
- `DisposableStack` handles one-off cleanup without a class: `using cleanup = new DisposableStack(); cleanup.defer(() => {...})`. Calling `defer` right after creating a value avoids odd dependency issues. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-2.html#using-declarations-and-explicit-resource-management
- On runtimes without native support, polyfill `Symbol.dispose` and `Symbol.asyncDispose`, and add `esnext` or `esnext.disposable` to `lib`. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-2.html#using-declarations-and-explicit-resource-management
- The variable-declarations page types `[Symbol.asyncDispose]` as a property: implement it as a method, `async [Symbol.asyncDispose]() {}`. https://www.typescriptlang.org/docs/handbook/variable-declarations.html

## Iterators and symbols

- Type a function parameter that accepts any iterable as `Iterable<X>`. `for..in` yields keys, `for..of` yields values. https://www.typescriptlang.org/docs/handbook/iterators-and-generators.html
- `unique symbol` is allowed only on `const` and `readonly static` declarations. Two `unique symbol` types never compare. https://www.typescriptlang.org/docs/handbook/symbols.html

```ts
declare const sym1: unique symbol;
class C { static readonly StaticSymbol: unique symbol = Symbol(); }
```

https://www.typescriptlang.org/docs/handbook/symbols.html

## Destructuring and spread

- Destructuring past a tuple's length is an error. Object assignment destructuring needs parentheses: `({ a, b } = obj)`. https://www.typescriptlang.org/docs/handbook/variable-declarations.html
- Put the pattern before the default and property defaults inside it. https://www.typescriptlang.org/docs/handbook/variable-declarations.html

```ts
function f({ a, b = 0 } = { a: "" }): void {}
f({}); // error, 'a' is required if you supply an argument
```

https://www.typescriptlang.org/docs/handbook/variable-declarations.html

- Object spread copies own enumerable properties only, so a spread class instance loses its methods. Later properties win: put overrides last. https://www.typescriptlang.org/docs/handbook/variable-declarations.html
