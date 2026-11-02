# Decorators

## Contents

- Standard decorators (default)
- Decorator metadata
- Legacy decorators (`experimentalDecorators`)

## Standard decorators (default)

- Standard ECMAScript decorators need no flag. With `experimentalDecorators`, the older proposal applies and both checking and emit differ. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-0.html#differences-with-experimental-legacy-decorators
- Standard decorators cannot decorate parameters and do not work with `emitDecoratorMetadata`. Code that needs either keeps the legacy flag. Legacy decorator functions are unlikely to work as standard ones. Same URL.
- Signature: `(target, context)`. Returning a function replaces a method, returning nothing keeps it. Context types: `ClassMethodDecoratorContext`, `ClassFieldDecoratorContext`, `ClassGetterDecoratorContext`, `ClassAccessorDecoratorContext`. The 5.0 notes show only the method context, the field, getter and accessor contexts appear in the 5.2 notes. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-0.html#decorators and https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-2.html#decorator-metadata
- `context.addInitializer(fn)` hooks into the beginning of the constructor, or into class initialization for statics. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-0.html#decorators

```ts
function bound(originalMethod: any, context: ClassMethodDecoratorContext) {
    const methodName = context.name;
    if (context.private) {
        throw new Error(`'bound' cannot decorate private properties like ${methodName as string}.`);
    }
    context.addInitializer(function () {
        this[methodName] = this[methodName].bind(this);
    });
}
```

https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-0.html#decorators

- Stacked decorators apply from the closest to the farthest: in `@bound @loggedMethod greet()`, `@loggedMethod` wraps first. A factory must be called: `@loggedMethod("warn")`. Same URL.
- Type a method decorator with separate `This`, `Args` and `Return` parameters. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-0.html#writing-well-typed-decorators

```ts
function loggedMethod<This, Args extends any[], Return>(
    target: (this: This, ...args: Args) => Return,
    context: ClassMethodDecoratorContext<This, (this: This, ...args: Args) => Return>
) { /* ... */ }
```

https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-0.html#writing-well-typed-decorators

- Decorators go before or after `export` (`export default @register class Bar {}`), never both. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-0.html#differences-with-experimental-legacy-decorators
- `@super.decorate` is an error since 5.5: write `@(super.decorate)`. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-5.html#stricter-parsing-for-decorators
- A class decorator that returns a mixin class does not merge its type into the class: members it adds are unknown to the checker (the example uses `experimentalDecorators`). https://www.typescriptlang.org/docs/handbook/mixins.html#decorators-and-mixins

## Decorator metadata

- `context.metadata` is one plain object shared by every decorated member of a class, readable afterwards as `Class[Symbol.metadata]`. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-2.html#decorator-metadata
- Key a `WeakMap` by `context.metadata` to keep the data private. Same URL.
- A polyfill for `Symbol.metadata` is needed. The notes also require `target` `es2022` or below and a `lib` with `esnext` or `esnext.decorators`. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-2.html#decorator-metadata

```ts
Symbol.metadata ??= Symbol("Symbol.metadata");
```

```json
{ "compilerOptions": { "target": "es2022", "lib": ["es2022", "esnext.decorators", "dom"] } }
```

https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-2.html#decorator-metadata

## Legacy decorators (`experimentalDecorators`)

- The Decorators handbook page documents this legacy stage 2 form. Its example config sets `target: "ES5"`, deprecated in 6.0: use ES2015 or newer. https://www.typescriptlang.org/docs/handbook/decorators.html
- `emitDecoratorMetadata` is experimental and needs `reflect-metadata` imported. It injects design-time type information through `Reflect.metadata`, such as `design:type`. https://www.typescriptlang.org/docs/handbook/decorators.html
- Factories evaluate top to bottom, decorators then run bottom to top. https://www.typescriptlang.org/docs/handbook/decorators.html
- Order per class: instance members (parameters, then method, accessor, property), static members, constructor parameters, class. https://www.typescriptlang.org/docs/handbook/decorators.html
- A class decorator that returns a new constructor must keep the original prototype, and its added properties stay unknown to the type (error 2339). https://www.typescriptlang.org/docs/handbook/decorators.html
- Decorate only the first accessor of a `get` and `set` pair. Property and parameter decorators can only observe, their return value is ignored. https://www.typescriptlang.org/docs/handbook/decorators.html
- A constructor parameter decorator receives `key === undefined`: type `key` as `string | symbol | undefined`. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-0.html#more-accurate-type-checking-for-parameter-decorators-in-constructors-under---experimentaldecorators
- No decorators in declaration files or in `declare class`. Method and parameter decorators are also refused on overloads. https://www.typescriptlang.org/docs/handbook/decorators.html
