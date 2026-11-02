# IPC: events, channels, evaluating JS

## Contents

- Picking a mechanism
- Emitting events
- Listening and unlistening
- Channels
- Evaluating JS from Rust
- Old patterns

## Picking a mechanism

- Default: a command for request/response, a `Channel` for streams, events for small fire-and-forget notifications. https://v2.tauri.app/develop/calling-frontend/ https://v2.tauri.app/concept/inter-process-communication/
- Events are not typed, always async, cannot return values and carry JSON only. They evaluate JS under the hood, so they suit neither large payloads nor high throughput. https://v2.tauri.app/develop/calling-rust/ https://v2.tauri.app/develop/calling-frontend/
- Event data sits outside the capabilities system: no fine-grained control over who receives what. Access to the events API itself is controlled by the Event permissions of the core permissions (events use commands under the hood). https://v2.tauri.app/develop/calling-frontend/ https://v2.tauri.app/concept/inter-process-communication/
- Channels are ordered and fast. Tauri uses them internally for download progress, child process output and WebSocket messages. https://v2.tauri.app/develop/calling-frontend/

## Emitting events

- Rust: `AppHandle` and `WebviewWindow` implement `Emitter` and `Listener`. Payloads are any `Serialize + Clone` type; add `#[serde(rename_all = "camelCase")]` for JS-friendly keys. https://v2.tauri.app/develop/calling-frontend/

```rust
use tauri::{Emitter, EventTarget};
app.emit("download-started", &url).unwrap();          // every listener
app.emit_to("login", "login-result", result).unwrap(); // one webview label
app.emit_filter("open-file", path, |target| match target {
  EventTarget::WebviewWindow { label } => label == "main" || label == "file-viewer",
  _ => false,
}).unwrap();
```

https://v2.tauri.app/develop/calling-frontend/

- JS: `emit(name, payload)` globally, `emitTo(label, name, payload)` for one webview. https://v2.tauri.app/develop/calling-rust/
- Pitfall: events sent to a specific webview do not reach regular global listeners. Catch everything with `listen(name, cb, { target: { kind: 'Any' } })` in JS, or `listen_any` in Rust. https://v2.tauri.app/develop/calling-rust/ https://v2.tauri.app/develop/calling-frontend/
- In JS, `WebviewWindow.listen` only receives events aimed at its own target. https://v2.tauri.app/start/migrate/from-tauri-1/
- Events emitted by the frontend also fire listeners registered from the frontend. https://v2.tauri.app/develop/_sections/frontend-listen/

## Listening and unlistening

- `listen` returns a Promise of the `unlisten` function, and the listener lives for the whole app otherwise. Await it before calling. https://v2.tauri.app/develop/_sections/frontend-listen/

```js
import { listen } from '@tauri-apps/api/event';
const unlisten = await listen('sync-complete', (event) => { /* event.payload */ });
unlisten();
```

https://v2.tauri.app/develop/_sections/frontend-listen/

- Page reload or navigation unregisters listeners, an SPA router does not. Unlisten on unmount. React: https://v2.tauri.app/develop/_sections/frontend-listen/

```js
useEffect(() => {
  const unlisten = listen('sync-complete', handler);
  return () => { unlisten.then((fn) => fn()); };
}, []);
```

https://v2.tauri.app/develop/_sections/frontend-listen/

- Vue: `onUnmounted(() => unlistenPromise?.then((fn) => fn()))`. Svelte: return the cleanup from `$effect`. Register in `useEffect` / `onMounted` / `$effect` when the handler needs DOM refs. https://v2.tauri.app/develop/_sections/frontend-listen/
- Listeners run in registration order, but async listeners under rapid events can finish out of order. Use a Channel when order matters. https://v2.tauri.app/develop/_sections/frontend-listen/
- `once(name, cb)` (JS) and `app.once` (Rust) fire a single time. https://v2.tauri.app/develop/_sections/frontend-listen/
- Rust listeners receive the payload as a JSON string: https://v2.tauri.app/develop/_sections/frontend-listen/

```rust
use tauri::Listener;
let event_id = app.listen("download-started", |event| {
  if let Ok(p) = serde_json::from_str::<DownloadStarted>(&event.payload()) { /* ... */ }
});
app.unlisten(event_id);
```

https://v2.tauri.app/develop/_sections/frontend-listen/

- Scope to one webview with `webview.listen(...)`. Inside a handler, stop with `handle.unlisten` and the event's id. https://v2.tauri.app/develop/_sections/frontend-listen/

## Channels

The command takes a `Channel<T>`, the frontend creates it and passes it in the args. Tag the enum so JS can switch on it. https://v2.tauri.app/develop/calling-frontend/

```rust
use tauri::ipc::Channel;
#[derive(Clone, serde::Serialize)]
#[serde(tag = "event", content = "data")]
enum DownloadEvent { /* Started { .. }, Progress { .. }, Finished */ }

#[tauri::command]
fn download(url: String, on_event: Channel<DownloadEvent>) {
  // on_event.send(DownloadEvent::...).unwrap();
}
```

```ts
import { invoke, Channel } from '@tauri-apps/api/core';
const onEvent = new Channel<DownloadEvent>();
onEvent.onmessage = (message) => { console.log(`got download event ${message.event}`); };
await invoke('download', { url: '...', onEvent });
```

https://v2.tauri.app/develop/calling-frontend/

## Evaluating JS from Rust

`webview.eval("console.log('hello from Rust')")`. To inject Rust values safely, serialize them with the `serialize-to-javascript` crate. https://v2.tauri.app/develop/calling-frontend/

## Old patterns

- v2 events rely on event targets rather than on the event source. `listen_global` is now `listen_any`. JS `event.listen()` acts like `listen_any` unless a `target` is given. https://v2.tauri.app/start/migrate/from-tauri-1/
- The v1 `api::ip` module (spelled so on the page) was rewritten and moved to `tauri::ipc`; see `tauri::ipc::Channel`. https://v2.tauri.app/start/migrate/from-tauri-1/
