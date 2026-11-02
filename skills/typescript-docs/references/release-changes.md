# Behavior changes from 5.0 to 6.0

New errors and semantics that appear when code written for an older 5.x compiler runs on 6.0. This file covers code behavior, not tsconfig migration.

## Contents

- 6.0
- 5.9
- 5.8
- 5.7
- 5.6
- 5.5
- 5.4
- 5.0 to 5.3

## 6.0

- A method-syntax function that never uses `this` is no longer context sensitive, so generic inference works whatever the property order. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html#less-context-sensitivity-on-this-less-functions

- `--stableTypeOrdering` orders union members like 7.0 (6.0 order follows declaration order and can print `500 | 100` in declaration emit). It costs up to 25% check time: turn it on only to diagnose 6.0 versus 7.0 differences, then fix with an explicit type argument or annotation. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html#the---stabletypeordering-flag
- `es2025` is valid for `target` and `lib`. It holds `RegExp.escape`, and `Promise.try`, `Iterator` methods and `Set` methods move there from `esnext`. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html#es2025-option-for-target-and-lib
- `Temporal` types need `target: esnext` or `lib: ["esnext"]` (or `esnext.temporal`). `Map` and `WeakMap` `getOrInsert` and `getOrInsertComputed` are in `esnext`. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html#new-types-for-temporal

## 5.9

- `ArrayBuffer` is no longer a supertype of typed arrays such as Node's `Buffer`. Errors such as `Type 'Buffer' is not assignable to type 'ArrayBuffer'`: update `@types/node` first, then write `Uint8Array<ArrayBuffer>` or pass `.buffer`. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-9.html#libdts-changes
- Fixed type-variable leaks in inference can raise new errors. Add explicit type arguments to the generic call. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-9.html#type-argument-inference-changes
- `import defer * as ns from "./mod.js"` accepts only namespace imports. The module is still loaded, only its execution waits for the first property access. TypeScript does not downlevel it: it needs `module: preserve` or `esnext` and a supporting runtime or bundler. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-9.html#support-for-import-defer

## 5.8

- A conditional expression in a `return` is checked branch by branch against the declared return type, so `any`-typed ternaries stop hiding bugs. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-8.html#granular-checks-for-branches-in-return-expressions

```ts
function getUrlObject(urlString: string): URL {
    return untypedCache.has(urlString) ?
        untypedCache.get(urlString) :
        urlString; // error: Type 'string' is not assignable to type 'URL'.
}
```

- Under `module: nodenext`, `require()` of ESM is allowed and import assertions (`assert`) are errors. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-8.html#restrictions-on-import-assertions-under---module-nodenext

## 5.7

- Reading a `let x: number` that is never assigned anywhere, from a nested function, is an error. A possibly assigned variable stays unflagged. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-7.html#checks-for-never-initialized-variables
- Typed arrays are generic: `Uint8Array<TArrayBuffer extends ArrayBufferLike = ArrayBufferLike>`. For errors about `Buffer` and `ArrayBufferLike`, you may need to update `@types/node`. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-7.html#typedarrays-are-now-generic-over-arraybufferlike
- Under `module: nodenext`, a JSON import needs `with { type: "json" }` and has only a default export. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-7.html#validated-json-imports-in---module-nodenext

```ts
import myConfigA from "./myConfig.json" with { type: "json" };
```

- A class method with a non-literal computed name adds an index signature (`[x: symbol]: () => number`) to the class type. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-7.html#creating-index-signatures-from-non-literal-method-names-in-classes

## 5.6

- Conditions that are syntactically always truthy or never nullish are errors: a regex literal in `if`, `x => 0` for `x >= 0`, `value < options.max ?? 100`. `true`, `false`, `0` and `1` stay allowed. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-6.html#disallowed-nullish-and-truthy-checks
- `strictBuiltinIteratorReturn`, part of `strict`, makes the return type of built-in iterators `undefined` instead of `any`, so `value` is possibly `undefined`. Give your own `Iterator<T, TReturn>` an explicit `TReturn`. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-6.html#strict-builtin-iterator-checks-and-strictbuiltiniteratorreturn
- Built-in iterators are `IteratorObject` (`ArrayIterator`, `MapIterator`, `SetIterator`) and carry helpers such as `map` and `take`. A value declared `Iterator<T>` or `Iterable<T>` is only the protocol: wrap it with `Iterator.from(...)` to get the helpers. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-6.html#iterator-helper-methods
- `.mts` and `.cts` and the `package.json` `"type"` inside `node_modules` are respected in every `module` mode except `amd`, `umd`, `system`. Before 5.6, a default import from an ESM-only package could be accepted unsafely under `esnext` plus `bundler`. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-6.html#respecting-file-extensions-and-packagejson-from-within-node_modules
- `noImplicitOverride` requires `override` on computed members that shadow a base member. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-6.html#correct-override-checks-on-computed-properties

## 5.5

- Inferred type predicates make `.filter(x => x !== null)` return `number[]`, so a later `nums.push(null)` errors. Annotate the variable `(number | null)[]` when it must accept `null`. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-5.html#inferred-type-predicates
- Regex literals are syntax-checked (unbalanced groups, missing backreferences, features newer than `target`). `new RegExp("...")` strings are not. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-5.html#regular-expression-syntax-checking
- Declaration emit no longer adds `/// <reference types="node" />` and drops handwritten reference directives unless marked `preserve="true"`. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-5.html#simplified-reference-directive-declaration-emit

## 5.4

- Conditional type constraints are the union of both branches unless `T` provably always or never extends, so some conditional types stop being assignable to one branch. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-4.html#more-accurate-conditional-type-constraints
- Two enums with the same name and members must have identical known values to be compatible. Members named `Infinity`, `-Infinity` or `NaN` are errors. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-4.html#new-enum-assignability-restrictions and https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-4.html#name-restrictions-on-enum-members
- Under `isolatedModules`, an import that conflicts with a local value (`import { Something } ...; let Something = 123;`) must be `import type` or renamed. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-4.html#errors-when-type-only-imports-conflict-with-local-values

## 5.0 to 5.3

- `super.member` where `member` is a class field (such as an arrow function property) is an error: `super` only reaches prototype members. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-3.html#checks-for-super-property-accesses-on-instance-fields
- `module` and `moduleResolution` must both be Node-flavored when either is `node16` or `nodenext`. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-2.html#module-and-moduleresolution-must-match-under-recent-nodejs-settings
- When a set `typeRoots` fails to resolve, TypeScript no longer walks up to parent `node_modules/@types` (error TS2688): add `./node_modules/@types` explicitly. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-1.html#explicit-typeroots-disables-upward-walks-for-node_modulestypes
- Getters and setters may have unrelated types when both are annotated, so `box.value = box.value` is not guaranteed valid. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-1.html#unrelated-types-for-getters-and-setters
- All enums are union enums. Assigning an out-of-domain literal to an enum type is an error. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-0.html#all-enums-are-union-enums and https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-0.html#enum-overhaul
- `>`, `<`, `<=`, `>=` on `number | string` are errors, write `+ns > 4`. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-0.html#forbidden-implicit-coercions-in-relational-operators
- `newLine` defaults to `LF` and `forceConsistentCasingInFileNames` to `true`. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-0.html#deprecations-and-default-changes
- Under `experimentalDecorators`, a constructor parameter decorator receives `key === undefined`: widen the `key` type. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-0.html#more-accurate-type-checking-for-parameter-decorators-in-constructors-under---experimentaldecorators
