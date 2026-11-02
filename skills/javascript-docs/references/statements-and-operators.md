# Statements and operators

## Contents

- Nullish and optional operators
- Spread, destructuring, delete
- Arithmetic, BigInt and bitwise operators
- Comparison
- Control flow
- try, catch and finally
- Old patterns

## Nullish and optional operators

- Default with `??`, not `||`: `??` only replaces `null`/`undefined` and keeps `0`, `""`, `false`. `count || 42` gives 42 for `count = 0`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Nullish_coalescing
- Mixing `??` with `||` or `&&` without parentheses is a `SyntaxError`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Nullish_coalescing

```js
null || undefined ?? "foo";   // SyntaxError
(null || undefined) ?? "foo"; // "foo"
```
https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Nullish_coalescing

- Write `user?.id ?? fallback`, not `user && user.id ?? fallback` (a `SyntaxError`). `(user && user.id) ?? fallback` is valid but verbose, and an `&&` guard is not safe when `user` is a falsy non-nullish value such as `0`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Errors/Cant_use_nullish_coalescing_unparenthesized , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Optional_chaining
- `?.` short-circuits one continuous chain only: `(a?.b).c` throws when `a` is nullish. Chain every possibly nullish link: `obj?.method?.()`, `arr?.[42]`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Optional_chaining
- `obj?.p = 1`, `new Map?.()` and an optional tagged template are syntax errors. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Optional_chaining
- `x ??= f()`, `x ||= f()`, `x &&= f()` assign only when needed: no setter call and no error on a `const` when skipped. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Nullish_coalescing_assignment , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Logical_OR_assignment , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Logical_AND_assignment
## Spread, destructuring, delete

- Spread is shallow and skips non-enumerable properties and the prototype. Spreading a non-iterable into an array or call throws. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Spread_syntax
- Spread conditionally with `...(cond && { a: 1 })` in objects and `...(cond ? [x] : [])` in arrays. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Spread_syntax
- Destructuring assignment without a declaration needs parentheses: `({ a, b } = obj);`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Destructuring
- `null` and `undefined` cannot be destructured. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Errors/No_properties
- `delete arr[i]` leaves an empty slot and keeps `length`. Use `splice()`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/delete
- `delete` on a non-configurable property returns `false` in sloppy mode and throws in strict mode. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/delete

## Arithmetic, BigInt and bitwise operators

- Truncate with `Math.trunc()`, never `| 0`, `~~`, `>> 0`: bitwise operators convert to 32-bit integers and drop high bits. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Bitwise_OR , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Bitwise_NOT , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Right_shift
- Convert to a number with unary `+x`. Convert to a string with `` `${x}` `` or `String(x)`, not `"" + x`, which coerces to a primitive with the `"default"` hint. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Unary_plus , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Addition
- `"" + t` throws for Temporal objects (their `valueOf` throws) while `` `${t}` `` works. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Addition
- Strings win in `+`: `"1" + 2n` is `"12"`, `1n + 2` throws `TypeError`. Convert explicitly: `1n + BigInt(2)` or `Number(1n) + 2`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Addition
- `+1n` throws and `>>>` rejects BigInt (use `>>`). `**` works on BigInt, `Math.pow()` does not. A negative BigInt exponent and `x / 0n` throw `RangeError`. BigInt division truncates toward zero. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Unary_plus , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Unsigned_right_shift , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Exponentiation , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Division
- `-2 ** 2` is a `SyntaxError`. `**` is right-associative and `^` is XOR. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Exponentiation

```js
-(2 ** 2); // -4
(-2) ** 2; // 4
2 ** 3 ** 2; // 512
NaN ** 0; // 1
1 ** Infinity; // NaN
```
https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Exponentiation

- `%` is a remainder with the sign of the dividend (`-13 % 5` is `-3`). For a modulo write `((n % d) + d) % d`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Remainder
- Shift counts are taken modulo 32: `100 << 32` is 100. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Left_shift
- Precedence groups operators but operands always evaluate left to right. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Operator_precedence

## Comparison

- Relational operators turn `null` into 0 and `undefined` into NaN, and any NaN operand makes all four false: `null < 1` is true, `undefined < 3` is false. So `x < y` is not always `!(x >= y)`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Less_than
- Strings compare by UTF-16 code units, not code points. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Less_than

## Control flow

- Iterate own keys with `Object.keys()` or `Object.entries()`. `for...in` visits inherited enumerable keys and gives string indexes for arrays. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/for...in
- `for...of` creates a new variable per iteration and iterates strings by code point. `break` or `throw` calls the iterator's `return()`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/for...of
- `for await...of` also takes sync iterables, and then yields resolved values of promises. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/for-await...of
- `switch` compares with `===` and falls through without `break`. Wrap a `case` that declares `let`/`const`/`class` in `{ }`: cases share one scope. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/switch
- Always brace nested `if`: a dangling `else` binds to the closest `if`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/if...else
- Escape nested loops with a labeled `break`/`continue` instead of flags. Neither crosses a function boundary. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/label , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/break , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/continue
- `forEach` cannot stop: use `for...of` or `some()`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Errors/Bad_break
- A `for` initializer cannot contain a bare `in`: parenthesize it. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/for
- Wrap a multi-line returned expression in parentheses after `return`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/return

## try, catch and finally

- Throw `Error` instances or subclasses, not strings: catching code may expect properties such as `message`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/throw
- Handle only expected errors and rethrow the rest. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/try...catch

```js
try {
  myRoutine();
} catch (e) {
  if (e instanceof RangeError) {
    // handle expected error
  } else {
    throw e;
  }
}
```
https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/try...catch

- A `finally` that ends with `return`, `throw`, `break` or `continue` overrides the `try`/`catch` outcome, so a rethrown error vanishes. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/try...catch
- `catch { }` without a binding is allowed. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/try...catch

## Old patterns

| Old pattern | Replacement | Source |
| :-- | :-- | :-- |
| `a \|\| default` for missing values | `a ?? default` | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Nullish_coalescing |
| `obj.first && obj.first.second` | `obj.first?.second` | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Optional_chaining |
| `x \| 0`, `~~x` | `Math.trunc(x)` | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Bitwise_NOT |
| `"" + x` | `` `${x}` `` or `String(x)` | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Addition |
| `javascript:void(0)` links | event handlers | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/void |
| `with` statement | a temporary variable | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/with |
