# Promises and async functions

## Contents

- Chains and handlers
- Choosing a combinator
- Creating promises
- async and await
- Memory and unhandled rejections
- Old patterns

## Chains and handlers

- Return the promise from every `then` callback. A started but unreturned promise is "floating": the next `then` runs early. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Using_promises
- Keep chains flat with one `catch` at the end. Nest only to limit what a `catch` covers. A rejection handler that completes normally turns the chain back to fulfilled, so rethrow to keep the error. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Using_promises#nesting , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise

```js
doSomething()
  .then((result) => doSomethingElse(result))
  .then((newResult) => doThirdThing(newResult))
  .catch(failureCallback);
```
https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Using_promises

- Put synchronous steps in one handler: each `then` costs a tick. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise
- Handlers always run as microtasks, even on a settled promise. Never write an API that calls back sometimes synchronously and sometimes asynchronously. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Using_promises#guarantees
- The `finally()` callback receives no argument, and `finally()` passes the outcome through, unless its callback throws or returns a rejected promise, which replaces the outcome. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/finally
- "Resolved" is not "fulfilled": `resolve(otherPromise)` locks onto the other, still pending, promise. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise
- Reject with `Error` instances to keep stack traces. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/catch
- Promises have no cancellation. Cancel the underlying operation with `AbortController`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Using_promises#cancellation
- A promise is work already running. For laziness use a function `() => expression`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise

## Choosing a combinator

- Run independent work concurrently. Default to `Promise.all()` (first rejection rejects). Use `allSettled()` when every outcome matters, `any()` for the first fulfillment, `race()` for the first settlement. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/all
- `Promise.all()` rejecting does not stop the other operations, and their results are lost. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Using_promises#composition
- Results follow input order. On an empty input `all` fulfills, `any` rejects with `AggregateError`, `race` stays pending forever. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/any
- `Promise.any()` rejects with an `AggregateError` whose reasons sit in `error.errors`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Errors/Promise_any_all_rejected
- Combinators mark every input as handled: later rejections are silently ignored. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/race
- `Promise.allKeyed()` and `Promise.allSettledKeyed()` are experimental. They take an object of promises and fulfill with an object of the same keys. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/allKeyed , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/allSettledKeyed

```js
const { a: resultA, b: resultB } = await Promise.allKeyed({ a: getA(), b: getB() });
```
https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/allKeyed

## Creating promises

- Use `new Promise` only to wrap a callback API, at the lowest level. Never wrap something already promise-based. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Using_promises#creating_a_promise_around_an_old_callback_api , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/Promise

```js
const wait = (ms) => new Promise((resolve) => setTimeout(resolve, ms));
```
https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Using_promises#creating_a_promise_around_an_old_callback_api

- The executor's return value is ignored, a throw after settling is ignored, and a promise whose resolvers are never called stays pending. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/Promise
- Lift a function that may be sync or async, and may throw, with `Promise.try(func, ...args)`. `Promise.resolve(func())` lets a sync throw escape. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/try

```js
return Promise.try(action)
  .then((result) => console.log(result))
  .catch((error) => console.error(error))
  .finally(() => console.log("Done"));
```
https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/try

- Use `Promise.withResolvers()` when code outside the executor (listeners, streams, queues) settles the promise. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/withResolvers

## async and await

- Awaiting one promise after another is sequential. Start independent work together with `Promise.all()` or `Promise.allSettled()`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function
- With two promises started together, `[await p1, await p2]` raises an unhandled rejection if `p2` rejects while `p1` is pending, even when the caller attaches `.catch`, because `p2` is not wired in yet. `Promise.all` wires both at once. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function
- Rewrite `.then().catch()` chains as one async function with `try/catch` around `await`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function

```js
async function getProcessedData(url) {
  let v;
  try {
    v = await downloadData(url);
  } catch (e) {
    v = await downloadFallbackData(url);
  }
  return processDataInWorker(v);
}
```
https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function

- `await` always yields at least one tick, even on a non-promise. A rejected promise returned without `await` drops the caller from the stack trace. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/await
- A forgotten `await` is only detectable through a type mismatch. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Using_promises
- An async function always returns a new promise. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function
- `forEach` expects a synchronous function and does not wait for promises returned by async callbacks. For sequential or concurrent async work, use promise composition, such as `for...of` with `await`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/forEach , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Using_promises#composition
- `Array.fromAsync()` awaits values one by one, `Promise.all()` concurrently. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/fromAsync
- Top-level `await` works only in modules. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/await
- The `AsyncFunction()` constructor parses its string arguments as code: an injection sink. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/AsyncFunction/AsyncFunction

## Memory and unhandled rejections

- Handlers stay on a pending promise: a loop of `pending.then(...)` runs out of memory. Combinators never remove their handlers, so racing a long-lived promise many times piles handlers on it. Cancel with an `AbortSignal`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/then , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise
- Browsers fire `unhandledrejection`. Node uses `process.on("unhandledRejection")`, and a listener that does not examine each rejection drops them silently. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Using_promises#promise_rejection_events
- Avoid relying on `Promise[Symbol.species]`: engines are investigating its removal. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/Symbol.species

## Old patterns

| Old pattern | Replacement | Source |
| :-- | :-- | :-- |
| Callback pyramids | promises and `async`/`await` | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Using_promises |
| `new Promise(r => r(func()))`, or `try/catch` around `Promise.resolve(func())` (not `Promise.resolve().then(func)`, which calls `func` asynchronously) | `Promise.try(func)` | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/try |
| `let resolve, reject` captured from an executor | `Promise.withResolvers()` | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/withResolvers |
| A rejection handler on every `then` | one trailing `catch()` | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise |
