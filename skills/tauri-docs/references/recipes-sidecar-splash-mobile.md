# Recipes: sidecars, splashscreen, file associations, mobile multi-window

## Contents

- Sidecar binaries
- Node.js as a sidecar
- Splashscreen
- File associations
- Multi-window on mobile

## Sidecar binaries

- List the binary in `bundle.externalBin` (paths relative to `src-tauri`). For each target, a file with the same name plus `-<target-triple>` must exist, with `.exe` on Windows. https://v2.tauri.app/develop/sidecar/

```json
{ "bundle": { "externalBin": ["binaries/my-sidecar"] } }
```

`src-tauri/binaries/my-sidecar-x86_64-unknown-linux-gnu`, `my-sidecar-aarch64-apple-darwin`, `my-sidecar-x86_64-pc-windows-msvc.exe`. https://v2.tauri.app/develop/sidecar/

- Host triple: `rustc --print host-tuple` (Rust 1.84+); older toolchains parse `rustc -vV` for `host: (\S+)`. https://v2.tauri.app/develop/sidecar/ https://v2.tauri.app/learn/sidecar-nodejs/
- The shell plugin must be installed and initialized first. https://v2.tauri.app/develop/sidecar/
- Pitfall: Rust `sidecar()` takes the file NAME only (`"my-sidecar"`), while JS `Command.sidecar()` takes the exact `externalBin` string (`"binaries/my-sidecar"`). https://v2.tauri.app/develop/sidecar/

```rust
use tauri_plugin_shell::ShellExt;
let sidecar_command = app.shell().sidecar("my-sidecar").unwrap().arg(cmd).arg(message);
let output = sidecar_command.output().await.unwrap();
// or: let (mut rx, mut child) = app.shell().sidecar("my-sidecar").unwrap().spawn()?;
// read CommandEvent::Stdout from rx, write with child.write(...)
```

https://v2.tauri.app/learn/sidecar-nodejs/ https://v2.tauri.app/develop/sidecar/

```js
import { Command } from '@tauri-apps/plugin-shell';
const output = await Command.sidecar('binaries/my-sidecar', ['hello', message]).execute();
```

https://v2.tauri.app/learn/sidecar-nodejs/

- JS needs `shell:allow-execute` (for `execute()`) or `shell:allow-spawn` (for `spawn()`). `args: true` allows anything, `false` nothing; otherwise list args in exact order, static strings or `{ "validator": "<regex>" }`, and JS must pass the full array matching it. https://v2.tauri.app/develop/sidecar/

```json
{ "identifier": "shell:allow-execute", "allow": [{
  "name": "binaries/my-sidecar", "sidecar": true,
  "args": ["arg1", "-a", "--arg2", { "validator": "\\S+" }] }] }
```

https://v2.tauri.app/develop/sidecar/

- The docs' rename script does not handle cross-compiling to another architecture. https://v2.tauri.app/develop/sidecar/
- Sidecars and their arguments are defined by permissions now, not the v1 allowlist; `tauri migrate` converts the old scope. https://v2.tauri.app/start/migrate/from-tauri-1/ https://v2.tauri.app/develop/sidecar/

## Node.js as a sidecar

- Desktop only. Compile the script to a binary (the docs use `@yao-pkg/pkg`) rather than shipping the Node runtime, which exposes readable JS and is larger. https://v2.tauri.app/learn/sidecar-nodejs/
- Talk to it via argv/stdout for short tasks, or stdin/stdout, a localhost server or local sockets; each has its own security concerns. https://v2.tauri.app/learn/sidecar-nodejs/

## Splashscreen

- Docs' advice: a splashscreen admits slow loading. Default to opening the main window with a small spinner; use a splash only for style or hard startup requirements. https://v2.tauri.app/learn/splashscreen/
- Setup: `main` window with `"visible": false`, a `splashscreen` window with `"url": "/splashscreen"`. https://v2.tauri.app/learn/splashscreen/
- Do the heavy work in `setup` through `tauri::async_runtime::spawn` so windows appear meanwhile. Track frontend and backend completion in `Mutex<SetupState>` state, then close the splash and `main_window.show()`. https://v2.tauri.app/learn/splashscreen/
- Never `std::thread::sleep` in async code: it freezes the app. Use `tokio::time::sleep` (`cargo add tokio -F time`). https://v2.tauri.app/learn/splashscreen/

## File associations

- Declare `bundle.fileAssociations`; the CLI generates Android intent filters and iOS document types. https://v2.tauri.app/learn/mobile-file-associations/

```json
{ "bundle": { "fileAssociations": [
  { "ext": ["png"], "mimeType": "image/png" },
  { "ext": ["mydata"], "mimeType": "application/octet-stream",
    "exportedType": { "identifier": "com.example.myapp.mydata", "conformsTo": ["public.data"] } }
] } }
```

https://v2.tauri.app/learn/mobile-file-associations/

- `ext` has no leading dot. `mimeType` is required for Android matching (common ones are inferred). Custom extensions need an `exportedType` on Apple platforms. Defaults: `role` `Editor`, `rank` `Default`, all Android actions (`Send`, `SendMultiple`, `View`). https://v2.tauri.app/learn/mobile-file-associations/
- Opened files arrive as `RunEvent::Opened` (macOS, iOS, Android only). Handle both cases: store URLs in managed state for a cold start and expose a command, and emit an event while running. https://v2.tauri.app/learn/mobile-file-associations/

```rust
.run(|app, event| {
    #[cfg(any(target_os = "macos", target_os = "ios", target_os = "android"))]
    if let tauri::RunEvent::Opened { urls } = event {
        use tauri::Emitter;
        app.state::<OpenedUrls>().0.lock().unwrap().extend(urls.clone());
        app.emit("opened", urls).unwrap();
    }
});
```

```javascript
const initialUrls = await invoke('opened_urls'); // cold start
await listen('opened', (event) => { handleFiles(event.payload); }); // already running
```

https://v2.tauri.app/learn/mobile-file-associations/

## Multi-window on mobile

- Needs Android 12L (API 32)+ or iOS 13+; check `app.supportsMultipleWindows` at runtime. On phones windows do not sit side by side (Android stacks activities, iPhone often replaces the UI); true concurrency is an iPad experience. https://v2.tauri.app/learn/mobile-multiwindow/
- Grant `core:webview:allow-create-webview-window`. iOS scene windows get labels `main-1`, ... so use `"windows": ["main", "main-*"]`. https://v2.tauri.app/learn/mobile-multiwindow/
- Use a history router (`createBrowserRouter`), not a hash router, so each window has its own path. https://v2.tauri.app/learn/mobile-multiwindow/
- Android: add `androidx.window:window` and `androidx.startup:startup-runtime`, an activity per extra window extending `TauriActivity` with `PROPERTY_ACTIVITY_EMBEDDING_SPLITS_ENABLED`, and split rules in `res/xml/main_split_config.xml`. https://v2.tauri.app/learn/mobile-multiwindow/
- iOS: `src-tauri/Info.ios.plist` with `UIApplicationSupportsMultipleScenes` true, then handle `RunEvent::SceneRequested` to build the window. https://v2.tauri.app/learn/mobile-multiwindow/
