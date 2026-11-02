# Language core: equality, coercion, declarations, strict mode, ASI

## Contents

- Equality and sameness
- Coercion
- Declarations, hoisting and the temporal dead zone
- Strict mode
- Automatic semicolon insertion and lexical grammar
- Execution model and memory
- Old patterns

## Equality and sameness

- Default to `===`. Use `Object.is()` only to tell `-0` from `+0` (meta-programming, property descriptors). To treat two `NaN` values as equal, special-case the check with `isNaN` instead of reaching for `Object.is()`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Equality_comparisons_and_sameness
- `==` surprises: `[1,2] == "1,2"`, `"" == 0`, `0 == false`, `null == undefined` are true, `null == false` is false. Truthiness is not loose equality: `[]` is truthy yet `[] == false` is true. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Equality_comparisons_and_sameness , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Boolean
- `includes()`, `Map` and `Set` use SameValueZero: like `===`, but `NaN` equals `NaN`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Equality_comparisons_and_sameness
- `-0` comes from unary negation, `Math.round`, `Math.ceil`, `Math.pow`. `~`, `<<`, `>>` lose it. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Equality_comparisons_and_sameness
- `typeof null === "object"`, and `typeof` of arrays, dates and regexes is `"object"`. Use `Array.isArray` and similar checks. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/typeof
- `typeof` binds tighter than `+`: `typeof someData + " foo"` is `"number foo"`. Write `typeof (a + b)`.
- Check for `undefined` with `===`: `x == undefined` also matches `null`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/undefined

## Coercion

- Converting to a primitive tries `[Symbol.toPrimitive]()`, then `valueOf()`, then `toString()`. An object returned by `valueOf()` is ignored, so `{} + []` falls through to `toString()` and gives `"[object Object]"`. `Date` treats the default hint as string. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Data_structures
- Make custom `valueOf` or `[Symbol.toPrimitive]` idempotent: comparison operators coerce each operand every time, left before right. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Less_than
- The `Symbol.toPrimitive` hint (`"number"`, `"string"`, `"default"`) is only a weak signal. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Symbol/toPrimitive
- Never use `new Boolean`: `new Boolean(false)` is truthy. Convert with `Boolean(x)` or `!!x`. Constructors called with `new`, such as `new String("a")` and `new Number(1)`, return objects, so their `typeof` is `"object"`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Boolean , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/typeof
- To make a predicate return a boolean, write `!!obj && ...` or `obj !== null && ...`, not `obj && ...`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Boolean

## Declarations, hoisting and the temporal dead zone

- `let`, `const` and `class` throw `ReferenceError` before their declaration runs, even under `typeof`. `typeof` of an undeclared name gives `"undefined"` without throwing. The dead zone is about execution time, not text position. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/let
- `let foo = foo + 55` in a block that shadows an outer `foo` throws: the inner `foo` is in its dead zone.
- `const` freezes the binding, not the value: use `Object.freeze()` for the value. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/const
- `var` hoists as `undefined`, ignores blocks, and can be redeclared even in strict mode. A top-level `var` in a classic script becomes a non-deletable global property. Modules and CommonJS keep top-level declarations module-scoped. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/var
- `const z = y = x = f()` declares only `z`. `x` and `y` become implicit globals in sloppy mode and throw in strict mode. Write `const x = 5, y = 5`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Expressions_and_operators , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Assignment
- A function expression assigned to `var` is not hoisted as a function: an early call throws `TypeError`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/function
- Function declarations in blocks behave inconsistently in sloppy mode. Declare them in blocks only in strict code, where they are scoped to the block. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/function

## Strict mode

- Modules and class bodies are always strict. `"use strict"` applies to a whole script or function body only, before any other statement. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Strict_mode
- Strict mode turns silent failures into errors: assigning to an undeclared name, writing a read-only or getter-only property, deleting a non-configurable property, `delete name`, setting a property on a primitive, duplicate parameter names, legacy octals. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Strict_mode
- In strict code `this` stays `undefined` in a plain call and primitives are not boxed. `eval` no longer leaks variables. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Strict_mode
- `"use strict"` inside a function with default, rest or destructured parameters is a `SyntaxError`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Strict_mode#strict_mode_for_functions
- Migrate per file or per function: fix syntax and runtime errors, then review `this`, `arguments`, `eval` and block functions. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Strict_mode#transitioning_to_strict_mode

## Automatic semicolon insertion and lexical grammar

- `return`, `throw` and `yield` operands, labels after `break`/`continue`, the `=>` arrow and postfix `++`/`--` must stay on the same line. `return\n a + b` returns `undefined`. No line break after `async`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Lexical_grammar#automatic_semicolon_insertion
- Without semicolons, prefix a line that starts with `(`, `[`, `` ` ``, `+`, `-` or `/` with `;`, and end class fields with `;`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Lexical_grammar#automatic_semicolon_insertion

```js
;(() => {})()
;[1, 2, 3].forEach(console.log)
class A { a = 1; [b] = 2; *gen() {} }
```

- A statement cannot start with `function`, `class`, `async function`, `{` or `let[`. Wrap it in parentheses. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/Expression_statement
- Write octal as `0o777`. `0777` and `\45` escapes are syntax errors in strict mode. Use `\x` escapes. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Strict_mode#legacy_octal_literals , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Errors/Deprecated_octal_escape_sequence
- `1.toString()` is a `SyntaxError`. Write `(1).toString()`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Errors/Identifier_after_number
- Use `//# sourceMappingURL=` at the end of the script, not `//@`. Bundler annotations such as `/*#__PURE__*/` have no specification: check your bundler, since an incorrect annotation can remove required behavior. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Magic_comments

## Execution model and memory

- Jobs run to completion and block input. Split long work into short jobs and avoid blocking APIs such as `alert()` and synchronous XHR. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Execution_model#run-to-completion
- Do not rely on tail calls: proper tail calls apply in strict mode only and just JavaScriptCore (Safari) implements them. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Execution_model#tail_calls
- Garbage collection cannot be triggered programmatically in the core language. Engines use mark-and-sweep, so cycles are collected. `delete` does not free memory. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Memory_management , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Errors/Delete_in_strict_mode
- Avoid `WeakRef` and `FinalizationRegistry` where possible: semantics are almost unguaranteed. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Memory_management

## Old patterns

| Old pattern | Replacement | Source |
| :-- | :-- | :-- |
| `with (obj) {}` | a temporary variable | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/with |
| `0777`, `"\45"` | `0o777`, `"\x25"` | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Errors/Deprecated_octal_literal , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Errors/Deprecated_octal_escape_sequence |
| `//@ sourceMappingURL` | `//# sourceMappingURL` or the `SourceMap` header | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Errors/Deprecated_source_map_pragma |
| `Object.observe`, `watch`, `__noSuchMethod__` | `Proxy` | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Deprecated_and_obsolete_features#obsolete_features |
| `for each...in` | `for...of` | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Deprecated_and_obsolete_features#obsolete_features |
| `toSource()`, `uneval()` | `toString()` or your own serializer | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Deprecated_and_obsolete_features#obsolete_features |
