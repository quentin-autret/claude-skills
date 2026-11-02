# Globals and Web APIs

## Contents

- fetch and WebSocket
- Web storage
- Microtasks and abort
- Experimental globals
- Old patterns

## fetch and WebSocket

- `fetch` is stable and built on undici. Its version is `process.versions.undici`. Pass a custom dispatcher with `fetch(url, { dispatcher })`. `setGlobalDispatcher()` from the `undici` package affects both. https://nodejs.org/docs/latest/api/globals.html#fetch
- `fetch` and the `http` global agent honor the built-in proxy support (`NODE_USE_ENV_PROXY=1` or `--use-env-proxy`, or `http.setGlobalProxyFromEnv()` at runtime; active development). https://nodejs.org/docs/latest/api/http.html#built-in-proxy-support
- A browser-compatible `WebSocket` client is global (added v21.0.0, no longer behind a flag since v22.0.0, stable since v22.4.0). Disable it with `--no-experimental-websocket`. https://nodejs.org/docs/latest/api/globals.html#class-websocket

## Web storage

- `localStorage` and `sessionStorage` are release candidates, on without a flag since v25.0.0. `localStorage` data is stored unencrypted in the `--localstorage-file`, capped at 10 MB, and shared by every user and request on a server. `sessionStorage` lives in memory (10 MB quota), only for the running process, and is not shared between workers. Since v26.0.0, `localStorage` without `--localstorage-file` throws a `DOMException` (v25 returned an empty object). Never keep per-user data there. https://nodejs.org/docs/latest/api/globals.html#localstorage

## Microtasks and abort

- The `process.nextTick` queue always drains before the microtask queue. An error thrown in `queueMicrotask` emits `'uncaughtException'`. https://nodejs.org/docs/latest/api/globals.html#queuemicrotaskcallback
- `AbortSignal.timeout(ms)` and `AbortSignal.any(signals)` are available. https://nodejs.org/docs/latest/api/globals.html#static-method-abortsignaltimeoutdelay
- Use `AbortSignal` to cancel an action or enforce a timeout, not to dispose of long-lived servers, streams or child processes: use `using` / `await using` for those. https://nodejs.org/docs/latest/api/deprecations.html#DEP0209

## Experimental globals

| Global | Status | Source |
| :-- | :-- | :-- |
| `navigator` | active development, `navigator.locks` experimental (v24.5.0) | https://nodejs.org/docs/latest/api/globals.html#navigator |
| `URLPattern` | experimental, added v24.0.0 | https://nodejs.org/docs/latest/api/globals.html#class-urlpattern |
| `EventSource` | experimental, needs `--experimental-eventsource` | https://nodejs.org/docs/latest/api/globals.html#class-eventsource |
| `Worker` (Web Workers) | experimental, added in v26.9.0, needs `--experimental-web-worker`, distinct from `node:worker_threads` | https://nodejs.org/docs/latest/api/globals.html#class-worker |
| `ErrorEvent` | added in v25.0.0 | https://nodejs.org/docs/latest/api/globals.html#errorevent |
| `QuotaExceededError` | added in v26.0.0 | https://nodejs.org/docs/latest/api/globals.html#class-quotaexceedederror |

## Old patterns

| Old pattern | Replacement | Source |
| :-- | :-- | :-- |
| `global` | `globalThis` | https://nodejs.org/docs/latest/api/globals.html#global |
| `atob()` / `btoa()` | `Buffer.from(data, 'base64')` / `buf.toString('base64')` | https://nodejs.org/docs/latest/api/globals.html#atobdata |
| `server.listen({ signal })`, `spawn(..., { signal })` to dispose | `await using server`, `using child` | https://nodejs.org/docs/latest/api/deprecations.html#DEP0209 |
