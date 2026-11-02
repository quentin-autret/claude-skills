# AsyncLocalStorage, AsyncResource, async_hooks, diagnostics_channel

## Contents

- AsyncLocalStorage
- AsyncResource
- async_hooks (migrate away)
- diagnostics_channel
- TracingChannel
- Old patterns

## AsyncLocalStorage

- Use `AsyncLocalStorage` for request context, never a hand-made `async_hooks` implementation: it is optimized and memory safe. Several instances coexist independently. https://nodejs.org/docs/latest/api/async_context.html#class-asynclocalstorage
- Default to `run(store, fn)`. `getStore()` is `undefined` outside it. https://nodejs.org/docs/latest/api/async_context.html#asynclocalstoragerunstore-callback-args

```js
async function fn() {
  await asyncLocalStorage.run(new Map(), () => {
    asyncLocalStorage.getStore().set('key', value);
    return foo(); // awaited
  });
}
```
https://nodejs.org/docs/latest/api/async_context.html#usage-with-asyncawait

- Avoid `enterWith()` (experimental): it changes the context for the rest of the synchronous execution, so later listeners of the same `emit` see it too. https://nodejs.org/docs/latest/api/async_context.html#asynclocalstorageenterwithstore
- `new AsyncLocalStorage({ defaultValue, name })` is added in v24.0.0. https://nodejs.org/docs/latest/api/async_context.html#new-asynclocalstorageoptions
- Capture the current context with `AsyncLocalStorage.snapshot()` (stable since v23.11.0, backported to v22.15.0). It replaces `AsyncResource` for simple cases: https://nodejs.org/docs/latest/api/async_context.html#static-method-asynclocalstoragesnapshot

```js
class Foo {
  #runInAsyncScope = AsyncLocalStorage.snapshot();
  get() { return this.#runInAsyncScope(() => asyncLocalStorage.getStore()); }
}
```

- `withScope(store)` (added in v25.9.0, experimental) returns a disposable for `using`. In an async function the code before the first `await` runs in the caller's flow, so the scope leaks into the caller: prefer `run()` for async work. https://nodejs.org/docs/latest/api/async_context.html#asynclocalstoragewithscopestore
- Call `disable()` before an instance can be garbage collected. https://nodejs.org/docs/latest/api/async_context.html#asynclocalstoragedisable
- Context lost? Promisify callback APIs with `util.promisify()`, bind custom thenables with `AsyncResource`, and log `getStore()` after suspect calls to find the culprit. https://nodejs.org/docs/latest/api/async_context.html#troubleshooting-context-loss

## AsyncResource

- Extend `AsyncResource` in pools, queues and drivers so callbacks run in the caller's context. Call `emitDestroy()` yourself, exactly once. https://nodejs.org/docs/latest/api/async_context.html#class-asyncresource , https://nodejs.org/docs/latest/api/async_context.html#asyncresourceemitdestroy

```js
class DBQuery extends AsyncResource {
  constructor(db) { super('DBQuery'); this.db = db; }
  getInfo(query, callback) {
    this.db.get(query, (err, data) => { this.runInAsyncScope(callback, null, err, data); });
  }
  close() { this.db = null; this.emitDestroy(); }
}
```

- The `asyncResource` property added to functions by `bind()` is deprecated. https://nodejs.org/docs/latest/api/async_context.html#static-method-asyncresourcebindfn-type-thisarg

## async_hooks (migrate away)

- `createHook`, `AsyncHook` and `executionAsyncResource` are experimental and "strongly discouraged". Use `AsyncLocalStorage` for context, `process.getActiveResourcesInfo()` for active resources, and diagnostics_channel for diagnostics. https://nodejs.org/docs/latest/api/async_hooks.html
- If you must use hooks: callbacks must not be async, a throw kills the process, and `console.log()` inside recurses forever (use `fs.writeFileSync`). https://nodejs.org/docs/latest/api/async_hooks.html#printing-in-asynchook-callbacks
- Installing any hook turns on costly promise tracking. Opt out with `trackPromises: false`. https://nodejs.org/docs/latest/api/async_hooks.html#disabling-promise-execution-tracking

## diagnostics_channel

- Create channels once at module top level with `diagnostics_channel.channel(name)`, never `new Channel()`. Put the module name in the channel name and document the message shape. https://nodejs.org/docs/latest/api/diagnostics_channel.html , https://nodejs.org/docs/latest/api/diagnostics_channel.html#class-channel
- Guard expensive payloads with `hasSubscribers`: https://nodejs.org/docs/latest/api/diagnostics_channel.html#channelhassubscribers

```js
const channel = diagnostics_channel.channel('my-module.query');
if (channel.hasSubscribers) channel.publish({ some: 'data' });
```

- Subscribers run synchronously in the publisher's context, and a throw becomes `'uncaughtException'`. https://nodejs.org/docs/latest/api/diagnostics_channel.html#diagnostics_channelsubscribename-onmessage
- `channel.subscribe()` is fine again: its deprecation was revoked in v24.8.0. https://nodejs.org/docs/latest/api/diagnostics_channel.html#channelsubscribeonmessage
- Built-in channels (experimental) cover console, crypto (`crypto.fips.indicator`), HTTP client and server, HTTP/2, module `require`/`import`, net, UDP, `child_process`, worker threads and `sqlite.db.query`. https://nodejs.org/docs/latest/api/diagnostics_channel.html#built-in-channels , https://nodejs.org/docs/latest/api/diagnostics_channel.html#sqlite
- `channel.bindStore(store, transform)` and `runStores()` (experimental) link a channel to an `AsyncLocalStorage`. https://nodejs.org/docs/latest/api/diagnostics_channel.html#channelrunstorescontext-fn-thisarg-args

## TracingChannel

- `diagnostics_channel.tracingChannel()` is stable since v26.8.0. Names follow `tracing:module.class.method:start|end|asyncStart|asyncEnd|error`. https://nodejs.org/docs/latest/api/diagnostics_channel.html#diagnostics_channeltracingchannelnameorchannels , https://nodejs.org/docs/latest/api/diagnostics_channel.html#tracingchannel-channels
- A trace is published only if subscribers existed when it started. https://nodejs.org/docs/latest/api/diagnostics_channel.html#tracingchanneltracesyncfn-context-thisarg-args
- Track failures on the `error` event: one action can emit several errors. https://nodejs.org/docs/latest/api/diagnostics_channel.html#errorevent
- `tracePromise` returns non-thenables as is with a warning (v26.0.0). For async functions `end` fires when the promise is returned. https://nodejs.org/docs/latest/api/diagnostics_channel.html#tracingchanneltracepromisefn-context-thisarg-args
- `channel.withStoreScope()` and `boundedChannel()` are added in v26.1.0, experimental. https://nodejs.org/docs/latest/api/diagnostics_channel.html#diagnostics_channelboundedchannelnameorchannels

## Old patterns

| Old pattern | Replacement | Source |
| :-- | :-- | :-- |
| `async_hooks.createHook()` for request context | `AsyncLocalStorage` | https://nodejs.org/docs/latest/api/async_hooks.html |
| `new AsyncResource()` only to capture context | `AsyncLocalStorage.snapshot()` | https://nodejs.org/docs/latest/api/async_context.html#static-method-asynclocalstoragesnapshot |
| `boundFn.asyncResource` | keep a reference to the resource | https://nodejs.org/docs/latest/api/async_context.html#static-method-asyncresourcebindfn-type-thisarg |
| `v8.promiseHooks` for request context | `AsyncLocalStorage` | https://nodejs.org/docs/latest/api/v8.html#promise-hooks |
