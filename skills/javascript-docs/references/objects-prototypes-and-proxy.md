# Objects, prototypes, Proxy, Symbol and eval

## Contents

- Own properties and static helpers
- Copying, defining and freezing
- Prototypes and null-prototype objects
- Proxy and Reflect
- Symbol and globalThis
- eval and code strings
- Old patterns

## Own properties and static helpers

- Test own properties with `Object.hasOwn(obj, key)`, never against `undefined`: the property may hold `undefined`. It also works on `Object.create(null)` objects. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Inheritance_and_the_prototype_chain , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/hasOwn
- `in` sees inherited properties. Setting a property to `undefined` keeps `"x" in obj` true: use `delete`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/in
- Call `Object.prototype` methods through static helpers or `.call()`, because an object may shadow them: `Object.prototype.propertyIsEnumerable.call(obj, "foo")`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object
- `Object.keys`/`entries` throw on `null`/`undefined`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/keys
- Loop keys and values with `for (const [k, v] of Object.entries(obj))`. `for...in` also visits inherited enumerable string keys. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Loops_and_iteration , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/for...in
- Group by string keys with `Object.groupBy()`. The result is a null-prototype object holding the same element references (not copies). Keys are strings or symbols, and other callback results are coerced to strings. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/groupBy
- Bracket access with external names exposes you to object injection. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Working_with_objects

## Copying, defining and freezing

- Deep-clone with `structuredClone()`. `Object.assign()` copies property values, so a nested object is copied by reference only. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/assign
- `Object.assign()` runs source getters and target setters, throws on a non-writable target property (leaving it half modified) and skips `null`/`undefined` sources. Spread does not trigger setters. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/assign , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Object_initializer
- To copy accessors and attributes, copy each property descriptor with `Object.getOwnPropertyDescriptor()` and apply them with `Object.defineProperty()`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/assign
- `Object.defineProperty()` defaults `writable`, `enumerable` and `configurable` to `false`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperty
- `Object.freeze()` is shallow, returns the same object, fails silently in sloppy mode, ignores private elements and throws on non-empty typed arrays. A recursive deep freeze loops forever on cycles: track visited objects, for example in a `WeakSet`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/freeze

## Prototypes and null-prototype objects

- Create with the prototype you want: `Object.create(proto)` or `{ __proto__: proto }` in a literal (standard, and can be faster). Changing a prototype later is very slow in every engine. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Inheritance_and_the_prototype_chain , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/setPrototypeOf
- Never extend `Object.prototype` or built-in prototypes. A later language addition breaks you. The only valid reason is polyfilling. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Inheritance_and_the_prototype_chain
- Use null-prototype objects as dictionaries: plain objects answer wrongly for keys like `"toString"` and are open to prototype pollution. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object

```js
const obj = Object.create(null);
const obj2 = { __proto__: null };
```
https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object

- Null-prototype objects have no `toString`: `${obj}` throws. `Object.groupBy()`, regex `groups`, `import.meta` and module namespaces return them. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object
- A constructor returning an object replaces the `new` result. A returned primitive is ignored. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/new

## Proxy and Reflect

- Forward with `Reflect` inside traps to keep default behavior. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy
- `set`, `deleteProperty` and `defineProperty` traps must return `true` on success: a `false` result throws `TypeError` in strict-mode assignment, strict-mode `delete` and `Object.defineProperty()`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy/Proxy/set , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy/Proxy/deleteProperty , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy/Proxy/defineProperty
- Traps must respect target invariants (non-configurable, non-writable, non-extensible), or they throw `TypeError`. `ownKeys` must list every non-configurable own key. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy/Proxy/ownKeys
- A proxy does not forward internal slots or private fields: `new Proxy(new Map(), {}).size` throws. Call methods with the real target as `this`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy

```js
const proxy = new Proxy(secret, {
  get(target, prop, receiver) {
    const value = target[prop];
    if (value instanceof Function) {
      return function (...args) {
        return value.apply(this === receiver ? target : this, args);
      };
    }
    return value;
  },
});
```
https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy

- Use `Proxy.revocable()` to bound the life of an exposed object: after `revoke()` any trap throws `TypeError`, and the target becomes eligible for garbage collection if nothing else references it. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy/revocable

## Symbol and globalThis

- `Symbol()` throws with `new`. `Symbol.for()` symbols are shared and never collected, so `WeakMap`, `WeakSet`, `WeakRef` and `FinalizationRegistry` reject them. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Symbol
- Avoid relying on `Symbol.species`: it allows arbitrary code execution and engines are investigating its removal. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Symbol/species
- Use `globalThis` for the global object in every environment. The property is writable and configurable so code can hide it, and in browsers it may be a proxy around the real global object. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/globalThis

## eval and code strings

- Replace `eval` with bracket access, callbacks (`setTimeout(() => ...)`), `JSON` for data. If code strings are unavoidable, use indirect eval or `Function()` with explicit bindings. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/eval
- Direct `eval(...)` reads and writes local scope, leaks `var` in sloppy code and blocks optimizations. `(0, eval)(...)` or `eval?.(...)` runs in global scope. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/eval

```js
function looseJsonParse(obj) {
  return eval?.(`"use strict";(${obj})`);
}
```
https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/eval

- `eval` returns a non-string argument unchanged: `eval(new String("2 + 2"))` returns the `String` object. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/eval
- Under CSP, prefer Trusted Types (`require-trusted-types-for 'script'` plus `trusted-types-eval` in `script-src`) to `unsafe-eval`, which is much less safe. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/eval

## Old patterns

| Old pattern | Replacement | Source |
| :-- | :-- | :-- |
| `obj.hasOwnProperty(k)` | `Object.hasOwn(obj, k)` | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object |
| `propertyIsEnumerable()` | `Object.getOwnPropertyDescriptor()` | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object |
| `obj.__proto__` accessor | `Object.getPrototypeOf`/`setPrototypeOf` | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/proto |
| `__defineGetter__`, `__defineSetter__` | accessor syntax or `Object.defineProperty()` | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/__defineGetter__ |
| `__lookupGetter__`, `__lookupSetter__` | `Object.getOwnPropertyDescriptor()` | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/__lookupGetter__ |
| `setTimeout("code")`, `setAttribute("onclick", "...")` | callbacks | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/eval |
