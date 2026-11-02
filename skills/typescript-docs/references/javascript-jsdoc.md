# JavaScript with JSDoc, and migrating to TypeScript

## Contents

- Turning checks on in JS
- How `.js` files are inferred
- JSDoc tags that matter
- JSDoc traps
- Migrating a JS project
- Babel and other transpilers

## Turning checks on in JS

- JSDoc types in a `.js` file raise no errors by default. Enable them per file with `// @ts-check` on the first line. For many files, switch to a `jsconfig.json`. `// @ts-nocheck` skips a file, `// @ts-ignore` or `// @ts-expect-error` on the preceding line skips one line. https://www.typescriptlang.org/docs/handbook/intro-to-js-ts.html

```js
// @ts-check
/** @type {number} */
var x;
x = false; // Not OK
```

https://www.typescriptlang.org/docs/handbook/intro-to-js-ts.html

## How `.js` files are inferred

- Properties assigned only in methods are optional. A property never assigned in the class body is `unknown`. Declare it in the constructor with a JSDoc type. https://www.typescriptlang.org/docs/handbook/type-checking-javascript-files.html#properties-are-inferred-from-assignments-in-class-bodies
- Object literals are open-ended in JS (`obj.b = 2` allowed). Close one with a JSDoc type. https://www.typescriptlang.org/docs/handbook/type-checking-javascript-files.html#object-literals-are-open-ended
- A value initialized to `null` or `undefined` is `any`, and `[]` is `any[]`, even under `strictNullChecks`. https://www.typescriptlang.org/docs/handbook/type-checking-javascript-files.html#null-undefined-and-empty-array-initializers-are-of-type-any-or-any
- Parameters of an unannotated JS function are optional. Once annotated, mark optional ones `[name]`. https://www.typescriptlang.org/docs/handbook/type-checking-javascript-files.html#function-parameters-are-optional-by-default
- Missing type arguments default to `any`: `class C extends Component` gives `any` props. Add `/** @augments {Component<{a: number}, State>} */`. https://www.typescriptlang.org/docs/handbook/type-checking-javascript-files.html#unspecified-type-parameters-default-to-any

## JSDoc tags that matter

All from https://www.typescriptlang.org/docs/handbook/jsdoc-supported-types.html unless noted.

- `@import` brings types into JSDoc without a runtime import. The symbols work only in comments. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-5.html#the-jsdoc-import-tag

```js
/** @import { Pet } from "./types" */
```

https://www.typescriptlang.org/docs/handbook/jsdoc-supported-types.html#import

- `@type {import("./types").Pet}` and `typeof import("./accounts").userAccount` also work.
- Casts wrap the expression in parentheses after a `@type` comment: `/** @type {number} */ (numberOrString)`. `/** @type {const} */ (1)` is a const assertion.
- `@satisfies` checks a value without widening it, unlike `@type`. https://www.typescriptlang.org/docs/handbook/jsdoc-supported-types.html#satisfies
- `@overload` comments declare overloads. The last plain comment documents the implementation. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-0.html#overload-support-in-jsdoc
- `@typedef {{ prop1: string, prop3?: number }} SpecialType` and `@callback` define types.
- `@template {string} K` constrains only the first name in a list. `@template [T=object]` sets a default.
- `@extends` passes type arguments to a generic base class, for classes only. `@override` is checked with `noImplicitOverride`.
- Only `@deprecated`, `@see` and `@link` work in `.ts` files. Unsupported tags (`@async`, `@yields`, `@memberof`) are silently ignored.

## JSDoc traps

All from https://www.typescriptlang.org/docs/handbook/jsdoc-supported-types.html#unsupported-patterns

- A postfix `=` on a property type in an object literal type does not mark it optional: write `{ a: string, b?: number }`.
- `?number` means `number | null` only under `strictNullChecks`, and `!number` is just `number`. Write TypeScript unions.
- Legacy synonyms: `array` is `Array<any>`, `promise` is `Promise<any>`, `Object` and `object` are `any`. `noImplicitAny: true` turns the last four off (`array`, `promise`, `Object`, `object`). https://www.typescriptlang.org/docs/handbook/jsdoc-supported-types.html#legacy-type-synonyms
- `@public`, `@private`, `@protected` have no effect in constructor functions. https://www.typescriptlang.org/docs/handbook/jsdoc-supported-types.html#property-modifiers
- A constructor function also callable without `new` cannot use `@constructor`. https://www.typescriptlang.org/docs/handbook/jsdoc-supported-types.html#constructor

## Migrating a JS project

- Keep input and output directories apart (`src` and `built`). https://www.typescriptlang.org/docs/handbook/migrating-from-javascript.html
- The page's starter config sets `target: "es5"`, and its declaration-files step says to set `moduleResolution` to `node`. Both are deprecated in 6.0. Use a current target, and `nodenext` or `bundler` for `moduleResolution`. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html#breaking-changes-and-deprecations-in-typescript-60
- Turn on strict options such as `noImplicitAny` before converting files. Errors still emit JavaScript unless `noEmitOnError` is set. https://www.typescriptlang.org/docs/handbook/migrating-from-javascript.html
- Do not silence `Cannot find name 'require'` with `declare function require`. Convert to imports and set `module`. https://www.typescriptlang.org/docs/handbook/migrating-from-javascript.html
- An object built property by property (`var options = {}`) is typed `{}`. Put properties in the literal or assert `{} as Options`. https://www.typescriptlang.org/docs/handbook/migrating-from-javascript.html
- The page suggests `any` for "anything goes". Prefer `unknown`, which is safer because it is not legal to do anything with an `unknown` value. https://www.typescriptlang.org/docs/handbook/2/functions.html#unknown
- Type `this` in prototype-assigned functions with a `this` parameter under `noImplicitThis`. https://www.typescriptlang.org/docs/handbook/migrating-from-javascript.html

## Babel and other transpilers

- Babel's `preset-typescript` transpiles without checking and cannot emit `.d.ts`. Run `tsc` for checks and declarations. https://www.typescriptlang.org/docs/handbook/babel-with-typescript.html

```json
"declaration": true,
"emitDeclarationOnly": true,
"isolatedModules": true
```

https://www.typescriptlang.org/docs/handbook/babel-with-typescript.html

- `isolatedModules` makes TypeScript warn about code a single-file transpiler cannot handle. https://www.typescriptlang.org/tsconfig/#isolatedModules
- Write `import type` or inline `type` imports so Babel, swc or esbuild know which imports can be removed. https://www.typescriptlang.org/docs/handbook/2/modules.html
- DOM: `getElementById` returns `HTMLElement | null`. `querySelector<E extends Element>` takes a type argument for general selectors, and `querySelectorAll` returns `NodeListOf`, which has no `map` or `filter`. https://www.typescriptlang.org/docs/handbook/dom-manipulation.html
