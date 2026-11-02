# node:events and node:timers

## Contents

- EventEmitter
- Async listeners and rejections
- events.once, events.on, abort listeners
- EventTarget
- Timers
- timers/promises
- Old patterns

## EventEmitter

- Always add an `'error'` listener: emitting `'error'` without one throws and exits. Observe errors without consuming them with `events.errorMonitor`. https://nodejs.org/docs/latest/api/events.html#error-events

```js
myEmitter.on(errorMonitor, (err) => { MyMonitoringTool.log(err); });
myEmitter.on('error', (err) => { console.error('whoops! there was an error'); });
```

- Listeners run synchronously in registration order. Defer inside the listener with `setImmediate()` when needed. https://nodejs.org/docs/latest/api/events.html#asynchronous-vs-synchronous
- `on()` does not de-duplicate. `removeListener()` removes at most one instance (the latest added), and removing during an `emit` does not stop the current call. https://nodejs.org/docs/latest/api/events.html#emitterremovelistenereventname-listener
- `'newListener'` fires before the listener is added, `'removeListener'` after removal. https://nodejs.org/docs/latest/api/events.html#event-newlistener
- The 10-listener limit is a warning (`MaxListenersExceededWarning`), not a cap. Prefer `emitter.setMaxListeners(n)`: changing `events.defaultMaxListeners` affects every emitter, existing ones included, but not `AbortSignal`. https://nodejs.org/docs/latest/api/events.html#eventsdefaultmaxlisteners

## Async listeners and rejections

- An async listener that rejects causes an unhandled rejection. Use `captureRejections: true` to route it to `'error'` (or `[Symbol.for('nodejs.rejection')]`). Never make the `'error'` handler async. https://nodejs.org/docs/latest/api/events.html#capture-rejections-of-promises

```js
const ee = new EventEmitter({ captureRejections: true });
ee.on('something', async (value) => { throw new Error('kaboom'); });
ee.on('error', console.log);
```

- Bind async context to an emitter with `EventEmitterAsyncResource`. https://nodejs.org/docs/latest/api/events.html#class-eventsEventEmitterAsyncResource-extends-eventemitter

## events.once, events.on, abort listeners

- `await once(emitter, name, { signal })` resolves with an array of arguments and rejects on `'error'` (not for `EventTarget`). https://nodejs.org/docs/latest/api/events.html#eventsonceemitter-name-options
- Sequential `await once()` calls can miss events emitted in the same tick. Wait for several at once: https://nodejs.org/docs/latest/api/events.html#caveats-when-awaiting-multiple-events

```js
await Promise.all([once(ee, 'foo'), once(ee, 'bar')]);
```

- `for await (const args of on(emitter, name, { signal }))` handles one event at a time even with `await` in the body. Do not use it for concurrent handling. https://nodejs.org/docs/latest/api/events.html#eventsonemitter-eventname-options
- Listen for abort with `events.addAbortListener(signal, fn)`, not `signal.addEventListener('abort')`: a third party can call `stopImmediatePropagation()` and block yours. It returns a `Disposable`. Since v26.11.0 an already-aborted signal calls the listener in a microtask. https://nodejs.org/docs/latest/api/events.html#eventsaddabortlistenersignal-listener

```js
using _ = addAbortListener(signal, () => cleanup());
```

## EventTarget

- No propagation, and async listener rejections are handled like sync throws: they become uncaught exceptions on `process.nextTick()`. https://nodejs.org/docs/latest/api/events.html#eventtarget-error-handling
- `NodeEventTarget` is not an `EventEmitter`: a listener registers once per type, with no `prependListener`, `errorMonitor` or `'newListener'`. https://nodejs.org/docs/latest/api/events.html#nodeeventtarget-vs-eventemitter
- `capture` is part of the de-duplication key: `removeEventListener` must pass the same value. `signal` removes the listener on abort. https://nodejs.org/docs/latest/api/events.html#eventtargetaddeventlistenertype-listener-options

## Timers

- Delays above 2147483647, below 1 or `NaN` become 1. Non-integers are truncated. No exact time or order is guaranteed. https://nodejs.org/docs/latest/api/timers.html#settimeoutcallback-delay-args
- An immediate scheduled from an immediate runs on the next loop iteration. https://nodejs.org/docs/latest/api/timers.html#setimmediatecallback-args
- Timers keep the process alive. Call `unref()` for background timers. https://nodejs.org/docs/latest/api/timers.html#timeoutunref
- Restart a timer without allocating with `timeout.refresh()`. https://nodejs.org/docs/latest/api/timers.html#timeoutrefresh
- Cancel with `using`: `Timeout` and `Immediate` implement `Symbol.dispose` (stable since v24.2.0). https://nodejs.org/docs/latest/api/timers.html#timeoutsymboldispose
- The numeric id from `timeout[Symbol.toPrimitive]()` is valid only in the thread that created it. https://nodejs.org/docs/latest/api/timers.html#timeoutsymboltoprimitive

## timers/promises

- Default to `node:timers/promises` for delays. Abort with `signal`: the promise rejects with `err.name === 'AbortError'`. https://nodejs.org/docs/latest/api/timers.html#cancelling-timers

```js
import { setTimeout as sleep } from 'node:timers/promises';
await sleep(100, 'result', { signal: AbortSignal.timeout(5000) });
```

- `setInterval()` from `timers/promises` is an async iterator: `break` to stop. https://nodejs.org/docs/latest/api/timers.html#timerspromisessetintervaldelay-value-options

```js
for await (const startTime of setInterval(100, Date.now())) {
  if ((Date.now() - startTime) > 1000) break;
}
```

- `scheduler.wait()` and `scheduler.yield()` are experimental. https://nodejs.org/docs/latest/api/timers.html#timerspromisesschedulerwaitdelay-options

## Old patterns

| Old pattern | Replacement | Source |
| :-- | :-- | :-- |
| `timeout.close()` | `clearTimeout(timeout)` | https://nodejs.org/docs/latest/api/timers.html#timeoutclose |
| `signal.addEventListener('abort', fn)` | `events.addAbortListener(signal, fn)` | https://nodejs.org/docs/latest/api/events.html#eventsaddabortlistenersignal-listener |
| `new Promise((r) => setTimeout(r, ms))` | `await setTimeout(ms)` from `node:timers/promises` | https://nodejs.org/docs/latest/api/timers.html |
| `events.listenerCount(emitter, name)` | still valid, deprecation revoked in v25.4.0 | https://nodejs.org/docs/latest/api/deprecations.html#DEP0033 |
