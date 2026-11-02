---
name: javascript-docs
description: Teaches current JavaScript (ECMAScript) practice drawn from the MDN JavaScript documentation. Use when writing, reviewing or fixing modern JavaScript, choosing a built-in API, or replacing a deprecated pattern. Covers equality and coercion, let/const and the temporal dead zone, strict mode, ASI, functions, classes and private fields, objects, prototypes, Proxy, Reflect, Symbol, eval, operators (??, ?., spread, BigInt arithmetic), statements, Promise, async/await, Promise.try, iterators and iterator helpers, generators, using/await using, DisposableStack, WeakRef, Array (toSorted, at, groupBy, fromAsync), Map, Set methods, WeakMap, strings and Unicode, encodeURIComponent, RegExp (v flag, RegExp.escape, lastIndex), Number, Math, BigInt, JSON (rawJSON, revivers), typed arrays, ArrayBuffer, base64, SharedArrayBuffer, Atomics, Temporal, Date, Intl (DateTimeFormat, NumberFormat, Collator, Segmenter, PluralRules), errors (cause, Error.isError), ES modules, import attributes, import.meta, dynamic import.
metadata:
  version: "MDN living documentation, latest ECMAScript as documented, no versioned path"
  source: https://developer.mozilla.org/en-US/docs/Web/JavaScript
  fetched: "2026-11-02"
---

# JavaScript (MDN)

This skill condenses the MDN JavaScript Guide and Reference into what differs from habit: pitfalls, surprising defaults, deprecated APIs with their replacements, and the patterns MDN recommends. Each claim in the reference files ends with the MDN page it comes from. Basics every JavaScript developer knows are left out.

## Version covered

MDN tracks the living ECMAScript standard and has no versioned path. It also documents features from TC39 stage 3, before they are final, so examples may use features that are not yet everywhere. https://developer.mozilla.org/en-US/docs/Web/JavaScript

Recent features may have limited availability. Before relying on one (Temporal, iterator helpers, `using`, `Promise.try`, `RegExp.escape`, `Error.isError`, `Math.sumPrecise`, `Uint8Array.fromBase64`, Set methods, `Map.prototype.getOrInsert`), check the target runtime. Items MDN marks experimental are labeled as such in the references.

## How to navigate

1. Read the highest-value rules below. They catch the most frequent mistakes.
2. Open the reference file that matches the task, from the table at the end. Each file stands alone. A task with time zones, DST or durations needs both Temporal files. A task with cleanup or `using` needs the iterators and resource management file.
3. When touching old code, read the "Old patterns" table at the end of the matching file.

## Highest-value rules

- Default with `??`, not `||`, and never mix them without parentheses. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Nullish_coalescing
- Use `===`. `Object.is()` only to tell `-0` from `+0`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Equality_comparisons_and_sameness
- Test own properties with `Object.hasOwn()`, never against `undefined`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/hasOwn
- Use a `Map` for keys known only at run time, not a plain object. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Map#objects_vs._maps
- Never extend `Object.prototype` or built-in prototypes. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Inheritance_and_the_prototype_chain
- Deep-clone with `structuredClone()`. Spread and `Object.assign()` are shallow. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/assign
- Return `({ ... })` from an arrow: braces alone are a block. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/Arrow_functions#function_body
- Use `#private` fields and brand-check with `#x in obj`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes/Private_elements
- A `finally` that returns or throws swallows the pending error. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/try...catch
- Throw `Error` objects and wrap causes with `new Error(msg, { cause })`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Error/cause
- Return every promise from `then` callbacks and end chains with one `catch`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Using_promises
- Start independent async work together with `Promise.all()`, not sequential `await`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function
- `forEach` neither stops early nor awaits async callbacks. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/forEach
- Lift sync-or-async callbacks with `Promise.try()`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/try
- Release resources with `using` / `await using` or a `DisposableStack`. Never rely on `FinalizationRegistry` for cleanup. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Resource_management
- `await using file = await fs.open(...)` needs both awaits: one for acquisition, one for disposal. Async cleanup goes in `[Symbol.asyncDispose]`, since a promise returned by `[Symbol.dispose]` is not awaited. Disposers run in reverse order, and errors from the body and disposers arrive as one `SuppressedError` (`error` = later, `suppressed` = earlier). https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/await_using
- Call iterator helpers on the iterator instead of spreading it into an array. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Iterator
- Default `sort()` compares strings: pass `(a, b) => a - b`, and prefer `toSorted()` to keep the original. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/sort
- Group with `Object.groupBy()` / `Map.groupBy()`, not `reduce()`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/reduce
- Strings are UTF-16 code units: count graphemes with `Intl.Segmenter`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/length
- Encode URL parts with `encodeURIComponent()`, not `encodeURI()`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/encodeURI
- Escape user text in patterns with `RegExp.escape()`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/RegExp/escape
- A reused `g` or `y` regex keeps `lastIndex` between calls. Prefer `matchAll()`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/RegExp/test
- Truncate with `Math.trunc()`, pass a radix to `parseInt()`, use `Number.isNaN()`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/parseInt
- Integers above 2^53 need BigInt, and JSON corrupts them unless you use strings, `JSON.rawJSON` or `context.source`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/JSON
- Use `crypto.getRandomValues()`, never `Math.random()`, for anything security related. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Math/random
- Prefer `Temporal` to the legacy `Date`. Temporal objects throw on `<` and `-`: use `compare()`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Date
- Use an IANA zone name, not a fixed offset, in `ZonedDateTime`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal/ZonedDateTime#time_zones_and_offsets
- Keep zoned values zoned: `toPlainDateTime()` drops the zone, so later arithmetic ignores DST. `ZonedDateTime.from()` requires a `[zone]` annotation (parse a bare offset string with `Instant.from()`), and `ZonedDateTime.since()` defaults to hours (`PT25H` across a DST end) unless you pass `largestUnit`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal/ZonedDateTime
- Reuse one `Intl` formatter and never compare its output with fixed strings. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Internationalization#formatting_data
- Load JSON modules with `with { type: "json" }` and resolve specifiers with `import.meta.resolve()`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Modules#loading_non-javascript_resources
- Avoid relying on `Symbol.species`: engines are investigating its removal. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Symbol/species

## Reference files

| File | Read it for |
| :-- | :-- |
| [references/language-core-and-equality.md](references/language-core-and-equality.md) | `==` versus `===`, coercion, `typeof`, `let`/`const`/`var` and the dead zone, strict mode, ASI, octals, execution model, obsolete syntax |
| [references/functions-and-classes.md](references/functions-and-classes.md) | arrow functions and `this`, default and rest parameters, accessors, `bind`, class fields, `#private`, static blocks, `extends`, `super` |
| [references/objects-prototypes-and-proxy.md](references/objects-prototypes-and-proxy.md) | `Object.hasOwn`, `assign`, `defineProperty`, `freeze`, `groupBy`, prototypes, null-prototype objects, `Proxy`, `Reflect`, `Symbol`, `globalThis`, `eval` |
| [references/statements-and-operators.md](references/statements-and-operators.md) | `??`, `?.`, logical assignment, spread, destructuring, `delete`, arithmetic and bitwise traps, BigInt operators, comparison, loops, `switch`, labels, `try`/`finally` |
| [references/async-and-promises.md](references/async-and-promises.md) | promise chains, combinators, `Promise.try`, `withResolvers`, `allKeyed`, `async`/`await` ordering, unhandled rejections, promise memory leaks |
| [references/iterators-and-resource-management.md](references/iterators-and-resource-management.md) | iteration protocols, iterator helpers, `Iterator.zip`/`concat`, generators, `using`, `await using`, `DisposableStack`, `SuppressedError`, `WeakRef`, `FinalizationRegistry` |
| [references/arrays-and-collections.md](references/arrays-and-collections.md) | copying array methods, `at`, `includes`, `findLast`, `sort`, `reduce`, sparse arrays, `Array.from`, `Map`, `Set` methods, `getOrInsert`, `WeakMap`, `WeakSet` |
| [references/strings.md](references/strings.md) | UTF-16, code points and graphemes, `localeCompare`, `normalize`, `replace` patterns, `replaceAll`, `matchAll`, `split`, `String.raw`, URI encoding |
| [references/regexp.md](references/regexp.md) | `RegExp.escape`, method choice, `lastIndex`, `y` and `d` flags, `u` and `v` modes, `\p{...}`, modifiers, anchors, groups, backreferences, lookbehind |
| [references/numbers-math-json.md](references/numbers-math-json.md) | float precision, `parseInt`, rounding, `toFixed`, BigInt, `Math.random`, `Math.sumPrecise`, `JSON.stringify` rules, revivers, `JSON.rawJSON` |
| [references/binary-data-and-shared-memory.md](references/binary-data-and-shared-memory.md) | typed arrays, `DataView` endianness, resizable and transferred `ArrayBuffer`, base64 and hex on `Uint8Array`, `SharedArrayBuffer`, `Atomics` |
| [references/date-and-temporal.md](references/date-and-temporal.md) | moving from `Date` to `Temporal`, choosing a Temporal type, shared Temporal rules, plain dates and times, maintaining `Date` code |
| [references/temporal-time-zones-and-durations.md](references/temporal-time-zones-and-durations.md) | `ZonedDateTime`, IANA zones, DST disambiguation and `offset`, `since`/`until`, `Duration` balancing and rounding, `Temporal.Now` |
| [references/intl.md](references/intl.md) | locale choice and fallback, `DateTimeFormat`, `NumberFormat` digit options, `Collator`, `Segmenter`, `PluralRules`, `RelativeTimeFormat`, `ListFormat`, `DurationFormat` |
| [references/errors-and-debugging.md](references/errors-and-debugging.md) | `cause`, `Error` subclasses, `Error.isError`, `AggregateError`, stack traces, common runtime error messages and their fixes |
| [references/modules.md](references/modules.md) | module versus script, live bindings, cycles, `import()`, `import.meta`, import attributes, `import defer`, `import source`, import maps |

Crawled pages, version note and fetch date are listed in [SOURCES.md](SOURCES.md).
