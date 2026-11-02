# IPC: commands, errors, state

## Contents

- Model
- Defining and registering commands
- Arguments and return values
- Errors
- Async commands
- Injected parameters
- Managed state
- Old patterns

## Model

- IPC is asynchronous message passing: arguments and results are serialized (JSON-RPC like), so the Core can reject a request before any function runs. Commands are request/response (`invoke` works like `fetch`); events are one-way. https://v2.tauri.app/concept/inter-process-communication/
- All IPC goes through the Core process, which can filter it in one place. https://v2.tauri.app/concept/process-model/

## Defining and registering commands

```rust
#[tauri::command]
fn my_custom_command(invoke_message: String) {
  println!("I was invoked from JavaScript, with this message: {}", invoke_message);
}

tauri::Builder::default()
  .invoke_handler(tauri::generate_handler![my_custom_command])
  .run(tauri::generate_context!())
```

```js
import { invoke } from '@tauri-apps/api/core';
await invoke('my_custom_command', { invokeMessage: 'Hello!' });
```

https://v2.tauri.app/develop/calling-rust/

- Pitfall: only the last `invoke_handler` call counts. Put every command in one `generate_handler!`. https://v2.tauri.app/develop/calling-rust/
- Command names are global, not module-scoped: they must be unique across modules. Commands in another module must be `pub` and listed with their path (`commands::my_custom_command`), but JS calls the bare name. https://v2.tauri.app/develop/calling-rust/
- Pitfall: commands defined in `lib.rs` cannot be `pub` (error `__cmd__command_name is defined multiple times`). https://v2.tauri.app/develop/calling-rust/
- `window.__TAURI__.core.invoke` (global script) needs `app.withGlobalTauri` set to true in `tauri.conf.json`. With the npm package, import `invoke` from `@tauri-apps/api/core`. https://v2.tauri.app/develop/calling-rust/
- Registered app commands are allowed for every window by default. Restrict them in `build.rs` with `tauri_build::try_build(tauri_build::Attributes::new().app_manifest(tauri_build::AppManifest::new().commands(&["your_command"])))`. https://v2.tauri.app/security/capabilities/

## Arguments and return values

- JS keys are camelCase by default (`invokeMessage` for `invoke_message`). Opt into snake_case with `#[tauri::command(rename_all = "snake_case")]`. https://v2.tauri.app/develop/calling-rust/
- Arguments implement `serde::Deserialize`, return values `serde::Serialize`. `Result<(), E>` resolves to `null`. https://v2.tauri.app/develop/calling-rust/
- Large binary results: return `tauri::ipc::Response::new(data)` to skip JSON serialization. https://v2.tauri.app/develop/calling-rust/
- Raw body input: take `tauri::ipc::Request` and pass an `ArrayBuffer`/`Uint8Array` with headers as the third argument. https://v2.tauri.app/develop/calling-rust/

```js
await invoke('upload', data, { headers: { Authorization: 'apikey' } });
```

https://v2.tauri.app/develop/calling-rust/

- Streaming to the frontend: take a `tauri::ipc::Channel<T>` argument and call `send` on it. https://v2.tauri.app/develop/calling-rust/
- Rust/WASM frontends: Rust has no optional args, so bind two externs (for example `invoke_without_args` and `invoke`). https://v2.tauri.app/develop/calling-rust/

## Errors

- Return `Result<T, E>`: `Err` rejects the JS promise. `E` must implement `Serialize`. Quick path: `.map_err(|e| e.to_string())`. https://v2.tauri.app/develop/calling-rust/
- Optional, more idiomatic than `String`: one `thiserror` enum with a manual `Serialize` impl. To get `{ kind, message }` on the frontend, serialize through a helper enum with `#[serde(tag = "kind", content = "message")]` and `#[serde(rename_all = "camelCase")]`, mirrored by a TS type. It makes every possible error explicit. https://v2.tauri.app/develop/calling-rust/

## Async commands

- Sync commands run on the main thread unless marked `#[tauri::command(async)]`. Use async for heavy work to avoid freezing the UI. https://v2.tauri.app/develop/calling-rust/
- Pitfall: async commands cannot take borrowed arguments (`&str`, `State<'_, T>`) unless they return a `Result`. Owned types such as `String` also work, but not for `State`. https://v2.tauri.app/develop/calling-rust/

```rust
#[tauri::command]
async fn my_custom_command(value: &str) -> Result<String, ()> { Ok(value.to_string()) }
```

https://v2.tauri.app/develop/calling-rust/

- Never call `std::thread::sleep` in async code: it blocks the runtime thread. Use `tokio::time::sleep`. https://v2.tauri.app/learn/splashscreen/

## Injected parameters

- A command may also take `tauri::WebviewWindow`, `tauri::AppHandle`, `tauri::State<T>` or `tauri::ipc::Request`; the frontend does not pass them. https://v2.tauri.app/develop/calling-rust/
- To run under a non-default runtime (mock runtime in tests), make the command generic: `fn cmd<R: tauri::Runtime>(app: tauri::AppHandle<R>, window: tauri::WebviewWindow<R>)`. https://v2.tauri.app/develop/calling-rust/

## Managed state

- Register with `app.manage(value)` in `setup` or `Builder::manage`; read with `app.state::<T>()` from any `Manager` (for example `AppHandle`). https://v2.tauri.app/develop/state-management/
- Do not wrap in `Arc`: `State` already does. Wrap mutable state in `std::sync::Mutex`; use Tokio's async mutex only when the guard must live across `.await` or for IO resources such as a DB connection. https://v2.tauri.app/develop/state-management/

```rust
use std::sync::Mutex;
#[derive(Default)]
struct AppState { counter: u32 }

#[tauri::command]
fn increase_counter(state: tauri::State<'_, Mutex<AppState>>) -> u32 {
  let mut state = state.lock().unwrap();
  state.counter += 1;
  state.counter
}
// in setup: app.manage(Mutex::new(AppState::default()));
```

https://v2.tauri.app/develop/state-management/

- Pitfall: a wrong `State<T>` type compiles. The invoke is rejected at runtime with `state not managed for field ...`, and `Manager::state()` panics. Example: asking for `State<'_, AppState>` when `Mutex<AppState>` was managed. https://v2.tauri.app/develop/state-management/
- Mitigation: `type AppState = Mutex<AppStateInner>;` and always use the alias, never wrapped again. https://v2.tauri.app/develop/state-management/
- Outside commands (threads, `on_window_event`): clone an `AppHandle` (cheap) and call `handle.state::<T>()`, for example `window.app_handle().state::<Mutex<AppState>>()`. https://v2.tauri.app/develop/state-management/

## Old patterns

- `app_handle.path().app_dir()` and `GlobalShortcutManager` still appear in the docs' AppHandle sample. They are Tauri 1 style: use the `app.path()` resolvers (for example `app.path().resolve(path, BaseDirectory::Config)`) and the global-shortcut plugin. https://v2.tauri.app/develop/calling-rust/ and https://v2.tauri.app/start/migrate/from-tauri-1/
- `import { invoke } from "@tauri-apps/api/tauri"` is now `@tauri-apps/api/core`. https://v2.tauri.app/start/migrate/from-tauri-1/
