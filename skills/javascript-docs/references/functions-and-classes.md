# Functions and classes

## Contents

- Arrow functions and this
- Parameters
- Getters, methods and function properties
- Class fields and private elements
- Static members and inheritance
- Old patterns

## Arrow functions and this

- An arrow body in braces is a block: `() => { foo: 1 }` returns `undefined` (`foo:` is a label). Return an object with `() => ({ foo: 1 })`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/Arrow_functions#function_body
- Arrows have no own `this` or `arguments`, cannot be constructors and ignore `call`/`apply`/`bind` for `this`. Do not use them as object methods. Use them for callbacks that need the enclosing `this`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/Arrow_functions , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Functions
- Arrow class fields ("auto-bound methods") create one closure per instance, so they use more memory than prototype methods. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/Arrow_functions#cannot_be_used_as_methods
- `this` depends on how a function is called. Sloppy mode substitutes `globalThis` for `null`/`undefined`, strict mode keeps the raw value. Top-level `this` in a module is `undefined`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/this , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/bind

## Parameters

- Default values are evaluated at each call (a fresh `[]` each time) and apply only for `undefined`, not `null`. They can read earlier parameters, not body variables. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/Default_parameters

```js
function append(value, array = []) { array.push(value); return array; }
append(1); // [1]
append(2); // [2], not [1, 2]
```
https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/Default_parameters#evaluated_at_call_time

- Use rest parameters, not `arguments`: rest is a real `Array` holding only the extra arguments. There is one rest parameter, last, with no default. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/rest_parameters
- `fn.length` counts only parameters before the first default or rest. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/length

## Getters, methods and function properties

- Method shorthand (`{ m() {} }`) is not constructable, and only methods can use `super`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/Method_definitions
- Class accessors live on the prototype and are non-enumerable, unlike object-literal accessors. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/get#using_getters_in_classes
- A self-overwriting lazy getter must not back a value that changes, because it never recalculates. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/get#smart_self-overwriting_lazy_getters
- A setter that assigns to its own property name recurses forever. Use a separate backing property. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Errors/Too_much_recursion
- Do not drive logic from `fn.name`: minifiers rename functions. A static class member called `name` hides the built-in one. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/name
- A self-recursive function expression needs a name, local to its body. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/function
- Bound functions have no `prototype`, so they cannot be a base class. `new` ignores the bound `this` but keeps bound arguments. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/bind
- Do not chain constructors with `call()`/`apply()`: `new.target` is `undefined` and classes throw. Use `extends` or `Reflect.construct()`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/call
- `Function()` bodies see only global scope and are an injection sink. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/Function

## Class fields and private elements

- Declare fields (`name; age;`) to document the shape of the class. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes/Public_class_fields
- Fields use define semantics: a subclass field `field = 1` does not call a base setter, `this.field = 1` in the constructor does. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes/Public_class_fields

```js
class DerivedWithField extends Base { field = 1; }          // setter NOT called
class DerivedWithConstructor extends Base {
  constructor() { super(); this.field = 1; }                // setter called
}
```

- Field initializers run top to bottom and only see earlier fields. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes/Public_class_fields
- Use `#private` fields, methods and accessors for real encapsulation. They must be declared in the class body, are not accessible from subclasses, cannot be deleted or reached by brackets, and reading a missing one throws. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes/Private_elements , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Using_classes
- Brand-check with `#x in obj`, not try/catch. `"#x" in obj` tests a string key. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/in , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Using_classes

```js
static getX(obj) {
  if (#x in obj) return obj.#x;
  return "obj must be an instance of C";
}
```
https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes/Private_elements

- Constructors cannot be private: use a private flag. Stamping private fields twice on one object throws. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes/Private_elements#simulating_private_constructors , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes/Private_elements#returning_overriding_object
- A `Proxy` does not forward private fields: with `new Proxy(instance, {})`, a getter or method that reads `#x` runs with `this` set to the proxy and throws a `TypeError`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy#no_private_field_forwarding

## Static members and inheritance

- Use `static { }` blocks for static setup that needs statements or private static access, not an exposed `init()`. Static initializers cannot use `await` or `yield`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes/Static_initialization_blocks , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes/static
- In an instance method, call statics with `this.constructor.staticMethod()`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes/static#calling_static_members_from_a_class_constructor_and_other_methods
- Call `super()` before touching `this`. A derived constructor returns an object or `undefined` only. Do not return anything from a constructor. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes/extends , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Using_classes
- Classes are in the dead zone before their declaration, run strict, create no `globalThis` property, and throw when called without `new`. Detect a missing `new` with `new.target`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/class , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Using_classes , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/new.target
- Prefer composition to subclassing built-ins (performance and security cost). https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes/extends#subclassing_built-ins
- `class extends null` needs `return Object.create(new.target.prototype)` in the constructor. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes/extends#extending_null
- `[p1, p2].map(Promise.resolve)` throws: static factories need their class as `this`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes/extends#subclassing_built-ins
- Normally put methods on the prototype or in a class rather than recreating them inside a constructor, which costs speed and memory. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Closures

## Old patterns

| Old pattern | Replacement | Source |
| :-- | :-- | :-- |
| `var self = this`, `.bind(this)` in callbacks | arrow function | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Functions |
| IIFE per loop iteration | `let`/`const` or `for...of` | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Closures |
| `arguments` | rest parameters | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/arguments |
| `arguments.callee` | named function expression | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/arguments/callee |
| `Function.prototype.caller` | none, deprecated and non-standard, throws in strict, arrow, async and generator functions | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/caller |
| `Function.prototype.arguments` | the `arguments` variable inside the function | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/arguments |
| `Derived.prototype = Object.create(Base.prototype)` | `class Derived extends Base` | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Inheritance_and_the_prototype_chain |
| WeakMap-held private data | `#private` fields | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WeakMap#emulating_private_members |
