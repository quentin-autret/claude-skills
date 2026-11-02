# Iterators, generators and resource management

## Contents

- Iteration protocols
- Iterator helpers
- Generators
- using and await using
- DisposableStack
- WeakRef and FinalizationRegistry
- Old patterns

## Iteration protocols

- Built-in iterators inherit from `Iterator`. Make your own "proper iterator" with `class X extends Iterator` (still define `next()`) or `Iterator.from(obj)`: only those get helper methods. `new Iterator()` throws. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Iterator , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Iterator/Iterator
- A re-iterable object returns a new iterator from `[Symbol.iterator]()`. A one-shot one returns `this`. No reflection tells whether an object is an iterator. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Iterators_and_generators
- `return()` runs on early exit (`break`, `return`, an error in the consumer, destructuring that stops before the end), not when the iterator already returned `done: true`. A throw from `next()` aborts without cleanup. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Iteration_protocols
- An async-only iterable throws `x is not iterable` in `for...of` and spread. Use `for await...of` or `Array.fromAsync()`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Iteration_protocols
- Do not modify a collection while iterating it unless you know its implementation. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Iteration_protocols#concurrent_modifications_when_iterating
- `AsyncIterator` is not a global yet and async iterator helpers are only a proposal. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/AsyncIterator

## Iterator helpers

- Call `map`, `filter`, `take`, `drop`, `flatMap`, `reduce`, `find`, `some`, `every`, `toArray` on the iterator instead of spreading it into an array: one pass, no intermediate array, and infinite sources work. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Iterator

```js
const seq = fibonacci();
const firstThreeDigitTerm = seq.find((n) => n >= 100);
const totalDeposit = nameToDeposit.values().reduce((a, b) => a + b);
```
https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Iterator

- Helpers share their source: `it.drop(0)` is not a copy. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Iterator
- `take(limit)` throws `RangeError` for NaN or a negative limit. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Iterator/take
- `Iterator.concat()` takes iterables (wrap a bare iterator with `Iterator.from()`). For many or infinite sources use `flatMap`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Iterator/concat
- `Iterator.zip()` defaults to mode `"shortest"`. `"strict"` throws on unequal lengths, `"longest"` takes `padding`. Strings are rejected. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Iterator/zip

```js
for (const [index, [name, age]] of Iterator.zip([numbers, ages])) { /* shortest by default */ }
```
https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Iterator/zip

- `chunks(n)` and `windows(n)` are experimental. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Iterator/chunks , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Iterator/windows

## Generators

- The value passed to the first `next()` is ignored. `yield` works only directly in its generator, not in nested functions. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/yield
- Put cleanup in `try...finally` inside the generator: `return()` and `throw()` act like a `return` or `throw` at the paused `yield`, so `finally` runs. If `next()` was never called, or the generator already finished, they cannot be handled and just end it. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Generator , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/yield

## using and await using

- Tie a resource to a scope with `using` (sync) or `await using` (async). Disposers run in reverse declaration order, all of them, on error, `return` or `break`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/using , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Resource_management
- `using` accepts `null`/`undefined`. A value without `[Symbol.dispose]` throws `TypeError`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/using

```js
{
  using _ = new Lock();
  // lock released here
}
```
https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/using

- `await using file = await fs.open(...)` needs both awaits: one for acquisition, one for disposal. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/await_using
- `await using` falls back to `[Symbol.dispose]` without awaiting a promise it returns. Write `[Symbol.asyncDispose]` for async cleanup. Make disposers safe to call twice. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/await_using , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Symbol/dispose
- `using` is invalid at the top level of a classic script (fine in a module), at `switch` top level, in `for...in`, and with `export`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/using , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Resource_management
- A closure that escapes the scope sees a disposed resource. Nothing warns about a forgotten `using`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/using , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Resource_management#pitfalls
- When the body and disposers throw, you get a `SuppressedError` (`error` = later, `suppressed` = earlier). It has no `cause`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/using , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/SuppressedError
- When you drive an iterator by hand with `next()`, declare it with `using` so `return()` runs. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Iterator/Symbol.dispose
- The several `for await` plus `using` forms do different things. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/await_using

## DisposableStack

- Register a resource in the same statement that acquires it: `use(acquire())` for disposables, `adopt(value, cleanup)` for others, `defer(cb)` for unrelated cleanup. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/DisposableStack , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/DisposableStack/adopt

```js
using disposer = new DisposableStack();
const { signal } = disposer.adopt(new AbortController(), (c) => c.abort());
const pages = await Promise.all(urls.map((u) => fetch(u, { signal })));
```
https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Resource_management#examples

- Transfer ownership with `return disposer.move()` as the last step, and take it with `using` at the caller. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/DisposableStack/move
- Declare the stack with `using` (`await using` for `AsyncDisposableStack`). Call `dispose()` only to clean up early. `use()` on a disposed stack throws `ReferenceError`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/DisposableStack/dispose , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/DisposableStack/use

## WeakRef and FinalizationRegistry

- Avoid them where possible: callbacks may run late or never. Use them only to reduce memory, never for essential logic or critical cleanup. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/FinalizationRegistry , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Resource_management
- A fresh or just dereferenced `WeakRef` target survives the current job. A cleanup callback must check the `WeakRef` is empty before deleting a cache entry. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WeakRef , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Memory_management
- A held value that is an object is held strongly by the registry. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/FinalizationRegistry

## Old patterns

| Old pattern | Replacement | Source |
| :-- | :-- | :-- |
| `[...iterator].map(...)` | `iterator.map(...)` | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Iterator |
| Manual `try...finally` cleanup | `using` or `DisposableStack` | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Resource_management |
| `__iterator__`, `StopIteration`, legacy generators | `Symbol.iterator`, `function*` | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Deprecated_and_obsolete_features#legacy_generator_and_iterator |
| `FinalizationRegistry` for file handles | `using` | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Resource_management |
