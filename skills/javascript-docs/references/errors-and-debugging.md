# Errors and debugging

## Contents

- Creating and wrapping errors
- Detecting errors
- Stack traces
- Common runtime errors and their causes
- Old patterns

## Creating and wrapping errors

- Throw `Error` objects or subclasses (or `DOMException`), never strings or numbers, so `name`, `message` and `stack` exist. Any value can be thrown, but the exception should always be an `Error` or subclass instance. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/throw
- Wrap a lower-level failure with `cause` instead of losing it. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Error/cause

```js
try {
  connectToDatabase();
} catch (err) {
  throw new Error("Connecting to database failed.", { cause: err });
}
```
https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Error/cause

- In a library, let callers tell error kinds apart through a structured `cause` rather than parsing messages, which may be reworded. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Error/cause

```js
throw new Error("RSA key generation requires integer inputs.", {
  cause: { code: "NonInteger", values: [p, q] },
});
```
https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Error/cause

- A subclass must forward `options` to `super(message, options)`, or `cause` is not installed. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Error

```js
class MyError extends Error {
  constructor(message, options) {
    super(message, options); // needed to install "cause"
  }
}
```
https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Error

- Set `name` in an `Error` subclass, or `error.name` stays `"Error"`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes/constructor
- Transpiled to pre-class code, an `Error` subclass instance is not `instanceof` the subclass without extra configuration or `Object.setPrototypeOf(this, CustomError.prototype)`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Error
- The second `Error` argument is read only when it is an object. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Error/Error
- `AggregateError` (from `Promise.any()`) lists unrelated failures in `errors`. `SuppressedError` (from disposal) chains `error` and `suppressed` and has no `cause`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Errors/Promise_any_all_rejected , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/SuppressedError

## Detecting errors

- Detect errors with `Error.isError(value)`, not `instanceof Error`: it works across realms, rejects fakes like `{ __proto__: Error.prototype }` and accepts `DOMException`. It is recent, so check the page's browser compatibility table before relying on it. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Error/isError
- Normalize what you catch. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Error/isError

```js
try { throw "Oops"; } catch (e) {
  if (!Error.isError(e)) e = new Error(e);
  console.error(e.message);
}
```
https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Error/isError

- Catch only what you expect, test with `instanceof`, and rethrow the rest. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/try...catch
- Error messages are worded differently in each engine and can be reworded, so do not parse them. Test the error type with `instanceof` or use a structured `cause`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Error/cause , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Errors/Promise_any_all_rejected

## Stack traces

- `Error.prototype.stack` is non-standard: present in all major engines, but its content differs and so does where it lives (accessor on `Error.prototype` in Firefox, data property on each instance in Chrome and Safari). Do not parse it. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Error/stack
- `Error.stackTraceLimit` is non-standard (V8, JavaScriptCore): lower it for speed, raise it to debug. Setting it elsewhere is harmless. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Error/stackTraceLimit
- Guard `Error.captureStackTrace()` with `if (Error.captureStackTrace)`. It suits custom error objects. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Error/captureStackTrace , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Error
- A rejected promise returned without `await` drops the caller from the stack trace. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/await
- `InternalError` is non-standard (Firefox reports "too much recursion" with it). https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Error , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Errors/Too_much_recursion

## Common runtime errors and their causes

- `ReferenceError` before a `let`/`const` declaration: the temporal dead zone, decided by execution order. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Errors/Cant_access_lexical_declaration_before_init
- `TypeError` on a getter-only property: silent in sloppy mode, thrown in strict code. Classes are always strict, so assigning to a getter-only private element always throws. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Errors/Getter_only
- `TypeError` setting a property on a primitive in strict mode. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Errors/Cant_assign_to_property
- "Do not know how to serialize a BigInt": use a replacer, convert, or define `BigInt.prototype.toJSON`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Errors/BigInt_not_serializable
- "cyclic object value": `JSON.stringify` cannot serialize cycles. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Errors/Cyclic_object_value
- `JSON.parse` `SyntaxError`: the string is not valid JSON, for example a trailing comma. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Errors/JSON_bad_parse
- Invalid Date: `toISOString()` throws `RangeError`, while `toString()` gives `"Invalid Date"` and `toJSON()` gives `null`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Errors/Invalid_date
- Invalid array length: `new Array(n)` with a negative, fractional or too large `n`, or growth through `push`/`concat`. The `ArrayBuffer` constructor coerces a fractional length to an integer, `Array` does not. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Errors/Invalid_array_length
- "too much recursion" / "Maximum call stack size exceeded" (`InternalError` in Firefox, `RangeError` in Chrome and Safari): a missing base case or too many calls, including a setter assigning to its own name. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Errors/Too_much_recursion
- `matchAll`/`replaceAll` "requires global RegExp": add `g`, or use `match`/`replace` for one match. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Errors/Requires_global_RegExp

```js
const existingPattern = /./;
const newPattern = new RegExp(
  existingPattern.source,
  `${existingPattern.flags}g`,
);
"abc".replaceAll(newPattern, "f"); // "fff"
```
https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Errors/Requires_global_RegExp

- Key not weakly held: `WeakMap`/`WeakSet` keys must be objects or non-registered symbols. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Errors/Key_not_weakly_held
- Invalid assignment target: a call, `new`, `super()` or `this` is a value, not a reference. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Errors/Invalid_assignment_left-hand_side
- `import` declaration not at top level: use dynamic `import()` for conditional loading. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Errors/import_decl_module_top_level
- Arrow body parsed as a label: `() => { greet: function greet(){} }`. Wrap the object in parentheses or use method syntax. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Errors/Function_label
- Private elements cannot be added to unrelated objects (`TypeError`: object is not the right class). To attach state to such objects, prefer a `WeakMap` to the return-override trick. Use `#x in obj` to check first. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Errors/Get_set_missing_private

## Old patterns

| Old pattern | Replacement | Source |
| :-- | :-- | :-- |
| `throw "message"` | `throw new Error("message")` | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/throw |
| Catch and rethrow a new error without the original | `new Error(msg, { cause: err })` | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Error/cause |
| `e instanceof Error` across realms | `Error.isError(e)` | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Error/isError |
| `arguments.callee`, `fn.caller`, `fn.arguments` | stop using them: they are deprecated and throw `TypeError` in strict mode | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Errors/Deprecated_caller_or_arguments_usage |
