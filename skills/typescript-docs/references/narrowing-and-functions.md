# Everyday types, narrowing and functions

## Contents

- Types to prefer
- Literal inference and assertions
- Narrowing traps
- Type predicates
- Narrowing in closures and indexed access
- Functions and overloads
- Generic functions

## Types to prefer

- Use `unknown`, not `any`, for values accepted or returned unchecked: nothing is legal on `unknown` until narrowed. https://www.typescriptlang.org/docs/handbook/2/functions.html#unknown
- Use `object`, never `Object`. https://www.typescriptlang.org/docs/handbook/2/functions.html#object
- Use lowercase `string`, `number` and `boolean`, never `String`, `Number` or `Boolean`. https://www.typescriptlang.org/docs/handbook/2/everyday-types.html#the-primitives-string-number-and-boolean
- Avoid the `Function` type: its calls return `any`. Accept `() => void` for a function you do not call. https://www.typescriptlang.org/docs/handbook/2/functions.html#function
- Default to `interface` for object shapes and use `type` for unions, tuples and other features interfaces lack. Interfaces show by name in errors and `extends` is often faster to check than intersections. https://www.typescriptlang.org/docs/handbook/2/everyday-types.html#differences-between-type-aliases-and-interfaces
- A type alias is never a distinct type: `type Sanitized = string` accepts any `string`. https://www.typescriptlang.org/docs/handbook/2/everyday-types.html#type-aliases
- Model variants as a discriminated union with a literal `kind`, not optional fields plus `!`. https://www.typescriptlang.org/docs/handbook/2/narrowing.html#discriminated-unions

## Literal inference and assertions

- Object literal properties widen: `{ method: "GET" }` gives `method: string`. Add `as const` to the object to keep literals. https://www.typescriptlang.org/docs/handbook/2/everyday-types.html#literal-inference

```ts
const req = { url: "https://example.com", method: "GET" } as const;
handleRequest(req.url, req.method);
```

https://www.typescriptlang.org/docs/handbook/2/everyday-types.html#literal-inference

- Assertions and `!` have no runtime check. An impossible assertion errors, `expr as unknown as T` forces it. https://www.typescriptlang.org/docs/handbook/2/everyday-types.html#type-assertions
- `!!value` has a literal type (`!!"world"` is `true`), `Boolean(value)` is `boolean`. https://www.typescriptlang.org/docs/handbook/2/narrowing.html#truthiness-narrowing
- `const` type parameters infer like `as const` at the call site. The constraint must be readonly: `<const T extends readonly string[]>`. Passing a variable instead of an inline literal still infers `string[]`. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-0.html#const-type-parameters

## Narrowing traps

- `typeof x === "object"` keeps `null` in the type. https://www.typescriptlang.org/docs/handbook/2/narrowing.html#typeof-type-guards
- Truthiness checks on primitives drop `0`, `NaN`, `""` and `0n` silently. Compare with `!== undefined`. https://www.typescriptlang.org/docs/handbook/2/narrowing.html#truthiness-narrowing
- `!= null` removes both `null` and `undefined`. https://www.typescriptlang.org/docs/handbook/2/narrowing.html#equality-narrowing
- `"swim" in pet` keeps members where `swim` is optional on both branches. https://www.typescriptlang.org/docs/handbook/2/narrowing.html#the-in-operator-narrowing
- Assignment narrows against the declared type: a later assignment of any declared member stays legal. https://www.typescriptlang.org/docs/handbook/2/narrowing.html#assignments
- Exhaustiveness: assign the leftover to `never` in `default`, so a new union member fails to compile. https://www.typescriptlang.org/docs/handbook/2/narrowing.html#exhaustiveness-checking

```ts
default:
  const _exhaustiveCheck: never = shape;
  return _exhaustiveCheck;
```

https://www.typescriptlang.org/docs/handbook/2/narrowing.html#exhaustiveness-checking

- `switch (true)` narrows per `case` condition. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-3.html#switch-true-narrowing
- `if (isA(x) === true)` narrows through the predicate. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-3.html#narrowing-on-comparisons-to-booleans
- `instanceof` through `static [Symbol.hasInstance](v): v is T` narrows to `T`, not to the class. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-3.html#instanceof-narrowing-through-symbolhasinstance

## Type predicates

- TypeScript infers `x is T` for a function with no return annotation, a single `return`, no parameter mutation, and a boolean tied to a refinement of the parameter. `.filter(bird => bird !== undefined)` returns `Bird[]` with no hand-written guard. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-5.html#inferred-type-predicates
- Predicates mean "if and only if". `.filter(score => !!score)` on `number | undefined` infers nothing, because `0` is falsy. That truthiness filter drops zeros: filter with `!== undefined`. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-5.html#inferred-type-predicates
- `x => x` never infers a predicate, `x => !!x` may (for object types). https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-5.html#inferred-type-predicates
- A hand-written `is` predicate is unchecked, no safer than `as`. Give the inferred form as the fix, not a hand-written guard. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-5.html#inferred-type-predicates

## Narrowing in closures and indexed access

- A narrowed `let` or parameter stays narrowed in non-hoisted closures created after its last assignment, unless a nested function assigns it anywhere. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-4.html#preserved-narrowing-in-closures-following-last-assignments
- `obj[key]` narrows when neither `obj` nor `key` is reassigned. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-5.html#control-flow-narrowing-for-constant-indexed-accesses
- A function can capture a block-scoped variable before its declaration, but calling it before the declaration is illegal: a modern runtime targeting ES2015 throws, and TypeScript is permissive and does not report it. https://www.typescriptlang.org/docs/handbook/variable-declarations.html
- `Object.groupBy` returns all-optional keys: check each before use. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-4.html#objectgroupby-and-mapgroupby

## Functions and overloads

- A `() => void` contextual type accepts a function that returns a value, so `src.forEach(el => dst.push(el))` compiles. A literal annotated `: void` must not return. https://www.typescriptlang.org/docs/handbook/2/functions.html#return-type-void
- A function with return type `undefined` needs no `return`. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-1.html#easier-implicit-returns-for-undefined-returning-functions
- `(string) => void` declares a parameter named `string` of type `any`. https://www.typescriptlang.org/docs/handbook/2/functions.html#function-type-expressions
- Never mark a callback parameter optional unless the caller sometimes omits it. A callback with fewer parameters is always accepted. https://www.typescriptlang.org/docs/handbook/2/functions.html#optional-parameters-in-callbacks
- Prefer a union parameter over overloads. The implementation signature is invisible to callers, and a call resolves to one overload, so a `string | any[]` argument fails against separate overloads. https://www.typescriptlang.org/docs/handbook/2/functions.html#writing-good-overloads
- Type `this` with a fake first parameter and a `function` expression, not an arrow. https://www.typescriptlang.org/docs/handbook/2/functions.html#declaring-this-in-a-function

```ts
interface DB { filterUsers(filter: (this: User) => boolean): User[]; }
db.filterUsers(function (this: User) { return this.admin; });
```

https://www.typescriptlang.org/docs/handbook/2/functions.html#declaring-this-in-a-function

- `const args = [8, 5]` is `number[]`: spread it into fixed parameters only with `as const`. https://www.typescriptlang.org/docs/handbook/2/functions.html#rest-arguments
- A default `x = 10` makes `x` a `number` in the body, callers may still pass `undefined`. https://www.typescriptlang.org/docs/handbook/2/functions.html#optional-parameters

## Generic functions

- Use the type parameter itself rather than constraining it: `<T>(arr: T[]): T` returns `T`, `<T extends any[]>(arr: T)` returns `any`. https://www.typescriptlang.org/docs/handbook/2/functions.html#push-type-parameters-down
- Use as few type parameters as possible. One that does not relate two values (such as `Func extends (arg: Type) => boolean`) is a red flag, and one that appears in a single place deserves reconsidering. https://www.typescriptlang.org/docs/handbook/2/functions.html#use-fewer-type-parameters
- A constrained generic must return the same kind it received, not a fresh `{ length }` literal. https://www.typescriptlang.org/docs/handbook/2/functions.html#working-with-constrained-values
- `NoInfer<T>` blocks inference to the contained type, so a parameter adds no inference candidate, instead of a second type parameter used once. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-4.html#the-noinfer-utility-type

```ts
function createStreetLight<C extends string>(colors: C[], defaultColor?: NoInfer<C>) {}
createStreetLight(["red", "yellow", "green"], "blue");  // Error
```

https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-4.html#the-noinfer-utility-type

- Explicit type arguments fix mismatched inference: `combine<string | number>([1, 2, 3], ["hello"])`. https://www.typescriptlang.org/docs/handbook/2/functions.html#specifying-type-arguments
