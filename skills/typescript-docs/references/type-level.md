# Type-level programming

## Contents

- Generics and variance
- `keyof`, `typeof`, indexed access
- Conditional types and `infer`
- Mapped types
- Template literal types
- Utility types
- Compatibility surprises

## Generics and variance

- Inside a generic function, treat `Type` as any type: `arg.length` on a bare `Type` is error 2339. Constrain with `extends`. https://www.typescriptlang.org/docs/handbook/2/generics.html#working-with-generic-type-variables
- Constrain a key by another parameter to check property names. https://www.typescriptlang.org/docs/handbook/2/generics.html#using-type-parameters-in-generic-constraints

```ts
function getProperty<Type, Key extends keyof Type>(obj: Type, key: Key) {
  return obj[key];
}
```

https://www.typescriptlang.org/docs/handbook/2/generics.html#using-type-parameters-in-generic-constraints

- Type parameter defaults can replace overload sets. Required parameters never follow optional ones, the default must satisfy the constraint, and the default is used when inference has no candidate. https://www.typescriptlang.org/docs/handbook/2/generics.html#generic-parameter-defaults
- Enums and namespaces cannot be generic. https://www.typescriptlang.org/docs/handbook/2/generics.html#generic-types
- Variance is inferred. `in`, `out`, `in out` are for the extremely rare case where certain circular types make the inferred variance inaccurate. They act only in instantiation-based comparisons, so they cannot force invariance against a structurally compared object literal. Never write one that contradicts the real variance. https://www.typescriptlang.org/docs/handbook/2/generics.html#variance-annotations
- An unused type parameter has no effect: `Empty<string>` is assignable to `Empty<number>` when `T` appears in no member. Brands need `T` in a member. https://www.typescriptlang.org/docs/handbook/type-compatibility.html#generics

## `keyof`, `typeof`, indexed access

- `ReturnType<f>` is error 2749. Write `ReturnType<typeof f>`. https://www.typescriptlang.org/docs/handbook/2/typeof-types.html
- Type-position `typeof` takes identifiers and properties only, never calls. https://www.typescriptlang.org/docs/handbook/2/typeof-types.html#limitations
- Index only with types: `const key = "age"; Person[key]` errors. Use a type alias instead, such as `type key = "age"`. https://www.typescriptlang.org/docs/handbook/2/indexed-access-types.html
- `typeof MyArray[number]` gives the element type of an array literal. https://www.typescriptlang.org/docs/handbook/2/indexed-access-types.html
- For an enum, `keyof typeof Enum` is the key union. `keyof Enum` is not. https://www.typescriptlang.org/docs/handbook/enums.html#enums-at-compile-time

## Conditional types and `infer`

- Replace a family of overloads with one conditional return type. https://www.typescriptlang.org/docs/handbook/2/conditional-types.html

```ts
type NameOrId<T extends number | string> = T extends number ? IdLabel : NameLabel;
declare function createLabel<T extends number | string>(idOrName: T): NameOrId<T>;
```

https://www.typescriptlang.org/docs/handbook/2/conditional-types.html

- Move a check into the conditional instead of constraining the parameter: `type MessageOf<T> = T extends { message: unknown } ? T["message"] : never`. https://www.typescriptlang.org/docs/handbook/2/conditional-types.html#conditional-type-constraints
- A naked type parameter distributes over a union: `ToArray<string | number>` is `string[] | number[]`. Wrap both sides in tuples to stop it. https://www.typescriptlang.org/docs/handbook/2/conditional-types.html#distributive-conditional-types

```ts
type ToArrayNonDist<Type> = [Type] extends [any] ? Type[] : never;
```

https://www.typescriptlang.org/docs/handbook/2/conditional-types.html#distributive-conditional-types

- Inferring from an overloaded function uses the last signature, as `ReturnType` does. https://www.typescriptlang.org/docs/handbook/2/conditional-types.html#inferring-within-conditional-types
- The same `infer U` in covariant positions yields a union, in contravariant positions an intersection. https://www.typescriptlang.org/docs/handbook/advanced-types.html#type-inference-in-conditional-types
- `infer` is not allowed in a type parameter constraint (error 1338). Put it in a conditional. https://www.typescriptlang.org/docs/handbook/advanced-types.html#type-inference-in-conditional-types
- The constraint of a generic conditional is the union of its branches unless `T` provably always or never extends. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-4.html#more-accurate-conditional-type-constraints
- The deprecated Advanced Types page says conditional types cannot reference themselves recursively. The current Conditional Types page does not repeat that. https://www.typescriptlang.org/docs/handbook/advanced-types.html#distributive-conditional-types

## Mapped types

- `+` or `-` add or remove `readonly` and `?`: `-readonly [P in keyof T]`, `[P in keyof T]-?`. https://www.typescriptlang.org/docs/handbook/2/mapped-types.html#mapping-modifiers
- Remap keys with `as`, filter with `never` through a conditional type, and use `string & Property` inside a template literal. https://www.typescriptlang.org/docs/handbook/2/mapped-types.html#key-remapping-via-as

```ts
type Getters<Type> = {
  [Property in keyof Type as `get${Capitalize<string & Property>}`]: () => Type[Property]
};
```

https://www.typescriptlang.org/docs/handbook/2/mapped-types.html#key-remapping-via-as

- Map over any union of objects: `[E in Events as E["kind"]]: (event: E) => void`. https://www.typescriptlang.org/docs/handbook/2/mapped-types.html#key-remapping-via-as
- A mapped type cannot add members inside its braces: intersect with an object type. https://www.typescriptlang.org/docs/handbook/advanced-types.html#mapped-types
- Homomorphic types (`[P in keyof T]`: `Partial`, `Readonly`, `Pick`) copy modifiers. `Record` creates properties without them. Inference back through a mapped type only works for homomorphic ones. https://www.typescriptlang.org/docs/handbook/advanced-types.html#mapped-types

## Template literal types

- Unions in interpolations expand to every combination. Generate large string unions ahead of time instead. https://www.typescriptlang.org/docs/handbook/2/template-literal-types.html
- Infer a key from a pattern to type a callback. https://www.typescriptlang.org/docs/handbook/2/template-literal-types.html#inference-with-template-literals

```ts
type PropEventSource<Type> = {
  on<Key extends string & keyof Type>(eventName: `${Key}Changed`, callback: (newValue: Type[Key]) => void): void;
};
```

https://www.typescriptlang.org/docs/handbook/2/template-literal-types.html#inference-with-template-literals

- `Uppercase`, `Lowercase`, `Capitalize`, `Uncapitalize` are compiler intrinsics, not locale aware. https://www.typescriptlang.org/docs/handbook/2/template-literal-types.html#intrinsic-string-manipulation-types

## Utility types

- `Awaited<T>` unwraps promises recursively. https://www.typescriptlang.org/docs/handbook/utility-types.html#awaitedtype
- `Exclude` and `Extract` filter union members, `Omit` and `Pick` filter keys. `Exclude<Shape, { kind: "circle" }>` works on discriminated unions. https://www.typescriptlang.org/docs/handbook/utility-types.html#excludeuniontype-excludedmembers
- `NoInfer<T>` blocks inference from one position. https://www.typescriptlang.org/docs/handbook/utility-types.html#noinfertype
- `ThisType<T>` is a marker for object literal `this`, requires `noImplicitThis`, and transforms nothing. https://www.typescriptlang.org/docs/handbook/utility-types.html#thistypetype
- `Parameters<string>` and `ReturnType<Function>` are errors (2344). https://www.typescriptlang.org/docs/handbook/utility-types.html#parameterstype
- `InstanceType<typeof Point>` gives the instance type of a class value. https://www.typescriptlang.org/docs/handbook/utility-types.html#instancetypetype

## Compatibility surprises

- No best common supertype is invented: `[new Rhino(), new Elephant(), new Snake()]` is `(Rhino | Elephant | Snake)[]`. Annotate `Animal[]`. https://www.typescriptlang.org/docs/handbook/type-inference.html#best-common-type
- A function with fewer parameters assigns to one with more. Optional and required parameters are interchangeable in that check. https://www.typescriptlang.org/docs/handbook/type-compatibility.html#comparing-two-functions
- `unknown` assigns only to `unknown` and `any`. Nothing but `never` assigns to `never`, not even `any`. https://www.typescriptlang.org/docs/handbook/type-compatibility.html#any-unknown-object-void-undefined-null-and-never-assignability
- `T & string` with `T extends "abc" | "def"` reduces to `T`, `T & number` to `never`. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-4.html#more-aggressive-reduction-of-intersections-between-type-variables-and-primitive-types
