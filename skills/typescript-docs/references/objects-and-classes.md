# Object types, classes, merging and mixins

## Contents

- Object types
- Readonly, arrays and tuples
- Classes
- `this` in classes
- Privacy and structure
- Declaration merging and augmentation
- Mixins

## Object types

- In a destructuring pattern, `{ shape: Shape }` renames, it does not annotate. Annotate the whole parameter and default inside the pattern. https://www.typescriptlang.org/docs/handbook/2/objects.html#optional-properties

```ts
function paintShape({ shape, xPos = 0, yPos = 0 }: PaintOptions) { /* ... */ }
```

https://www.typescriptlang.org/docs/handbook/2/objects.html#optional-properties

- Excess property checks apply only to fresh object literals. Most are real bugs, so revise the type declaration, or add `[propName: string]: unknown`, rather than assert. https://www.typescriptlang.org/docs/handbook/2/objects.html#excess-property-checks
- A string index signature forces every named property to match its value type. `[k: string]: number; name: string` is an error: widen the index to `number | string`. https://www.typescriptlang.org/docs/handbook/2/objects.html#index-signatures
- Conflicting property types error between two `interface` declarations, but in an intersection they compile and the property becomes `never`. https://www.typescriptlang.org/docs/handbook/2/objects.html#interface-extension-vs-intersection
- `keyof` on a type with a string index signature is `string | number`. https://www.typescriptlang.org/docs/handbook/2/keyof-types.html
- Prefer a generic object type (`Box<T>`) over `any`, assertions or one overload per content type. https://www.typescriptlang.org/docs/handbook/2/objects.html#generic-object-types

## Readonly, arrays and tuples

- `readonly` is compile-time only, shallow, and ignored by assignability: a `Person` assigns to a `ReadonlyPerson` and changes through the alias. https://www.typescriptlang.org/docs/handbook/2/objects.html#readonly-properties
- Take `readonly T[]` for arrays you do not mutate. `readonly string[]` is not assignable to `string[]`. https://www.typescriptlang.org/docs/handbook/2/objects.html#the-readonlyarray-type
- `[3, 4] as const` is `readonly [3, 4]` and does not fit a `[number, number]` parameter. Type tuple parameters `readonly [number, number]` by default. https://www.typescriptlang.org/docs/handbook/2/objects.html#readonly-tuple-types
- Optional tuple elements come last and change `length` (`2 | 3`). Rest elements can sit anywhere: `[string, ...boolean[], number]`. https://www.typescriptlang.org/docs/handbook/2/objects.html#tuple-types

## Classes

- `implements` only checks. It neither types method parameters (they stay implicit `any`) nor adds optional members. https://www.typescriptlang.org/docs/handbook/2/classes.html#cautions
- Initialization order: base fields, base constructor, derived fields, derived constructor. A base constructor never sees derived field values. https://www.typescriptlang.org/docs/handbook/2/classes.html#initialization-order
- With `target` ES2022 or newer, or `useDefineForClassFields`, a redeclared inherited field overwrites what the parent set. Declare it with `declare` to narrow the type without emit. https://www.typescriptlang.org/docs/handbook/2/classes.html#type-only-field-declarations

```ts
class DogHouse extends AnimalHouse {
  declare resident: Dog;
  constructor(dog: Dog) { super(dog); }
}
```

https://www.typescriptlang.org/docs/handbook/2/classes.html#type-only-field-declarations

- `super.x` reaches prototype methods only. Accessing a field (such as an arrow function property) through `super` is an error. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-3.html#checks-for-super-property-accesses-on-instance-fields
- An override must stay a subtype: adding a required parameter that the base method lacks errors. https://www.typescriptlang.org/docs/handbook/2/classes.html#overriding-methods
- A derived class that omits `protected` on a redeclared member makes it public. Repeat `protected` unless that is intended. https://www.typescriptlang.org/docs/handbook/2/classes.html#exposure-of-protected-members
- Static members cannot use class type parameters or names such as `name`, `length`, `call`. https://www.typescriptlang.org/docs/handbook/2/classes.html#type-parameters-in-static-members
- Prefer a plain function or object to a class of only static members. https://www.typescriptlang.org/docs/handbook/2/classes.html#why-no-static-classes
- Accept `new () => Base` rather than `typeof Base` when the callee instantiates the class: an abstract class is then rejected at the call site. https://www.typescriptlang.org/docs/handbook/2/classes.html#abstract-construct-signatures
- Parameter properties (`constructor(public x: number)`) are not allowed under `erasableSyntaxOnly`. https://www.typescriptlang.org/tsconfig/#erasableSyntaxOnly

## `this` in classes

- Arrow function property: correct `this` at runtime, one copy per instance, no `super.method` in subclasses. A `this` parameter: checked statically, shared on the prototype, `super` works, JS callers can still misuse it. https://www.typescriptlang.org/docs/handbook/2/classes.html#arrow-functions

```ts
class MyClass2 {
  name = "MyClass";
  getName(this: MyClass2) { return this.name; }
}
```

https://www.typescriptlang.org/docs/handbook/2/classes.html#this-parameters

- Return `this` from fluent methods so subclasses keep their type. `other: this` accepts only the same subclass. https://www.typescriptlang.org/docs/handbook/2/classes.html#this-types
- `hasValue(): this is { value: T }` narrows a field lazily. https://www.typescriptlang.org/docs/handbook/2/classes.html#this-based-type-guards

## Privacy and structure

- TS `private` and `protected` are erased, and `s["secretKey"]` passes the checker. Use `#field` when privacy must hold at runtime. `#field` compiles to WeakMaps for targets up to ES2021. https://www.typescriptlang.org/docs/handbook/2/classes.html#caveats
- Classes compare structurally, statics and constructors excluded. A `private` or `protected` member makes compatibility nominal: it must originate from the same class. https://www.typescriptlang.org/docs/handbook/type-compatibility.html#private-and-protected-members-in-classes
- An empty class has no members, so anything can be used in place of it. https://www.typescriptlang.org/docs/handbook/2/classes.html#relationships-between-classes

## Declaration merging and augmentation

- Non-function interface members with the same name must have the same type, or it errors. Function members become overloads, with later declarations first, except single string-literal parameters, which bubble to the top. https://www.typescriptlang.org/docs/handbook/declaration-merging.html#merging-interfaces
- A namespace merged after a function, class or enum adds statics. It must come after the declaration and `export` its members. Classes never merge with classes. https://www.typescriptlang.org/docs/handbook/declaration-merging.html#merging-namespaces-with-classes-functions-and-enums
- Module augmentation patches existing declarations only: no new top-level declarations, no default export augmentation. Consumers import the patching file for its side effect. https://www.typescriptlang.org/docs/handbook/declaration-merging.html#module-augmentation

```ts
declare module "./observable" {
  interface Observable<T> {
    map<U>(f: (x: T) => U): Observable<U>;
  }
}
```

https://www.typescriptlang.org/docs/handbook/declaration-merging.html#module-augmentation

- `declare global { interface Array<T> { ... } }` inside a module augments globals, with the same limits. https://www.typescriptlang.org/docs/handbook/declaration-merging.html#global-augmentation
- All declarations of a merged symbol must agree on exported or local, ambient modules included. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-2.html#consistent-export-checking-for-merged-symbols

## Mixins

- Write mixins as a generic factory returning `class extends Base`, constrained by a constructor type. https://www.typescriptlang.org/docs/handbook/mixins.html#constrained-mixins

```ts
type GConstructor<T = {}> = new (...args: any[]) => T;
type Positionable = GConstructor<{ setPos: (x: number, y: number) => void }>;
function Jumpable<TBase extends Positionable>(Base: TBase) {
  return class Jumpable extends Base { jump() { this.setPos(0, 20); } };
}
```

https://www.typescriptlang.org/docs/handbook/mixins.html#constrained-mixins

- Mixins cannot declare `private` or `protected` members: use `#private`. https://www.typescriptlang.org/docs/handbook/mixins.html#how-does-a-mixin-work
- A decorator cannot add members to the class type. Use an intersection type or interface merging. https://www.typescriptlang.org/docs/handbook/mixins.html#constraints
