# Debugging: dev-only code, devtools, IDEs, Linux graphics

## Contents

- dev vs debug_assertions
- Consoles and devtools
- Debug builds
- IDE debugging of the core process
- Linux WebKitGTK graphics issues

## dev vs debug_assertions

- `#[cfg(dev)]` (and `tauri::is_dev()`) is true only under `tauri dev`. `#[cfg(debug_assertions)]` is true under `tauri dev` AND `tauri build --debug`. https://v2.tauri.app/develop/debug/

```rust
#[cfg(dev)]
{ /* `tauri dev` only */ }
#[cfg(debug_assertions)]
{ /* dev and debug builds */ }
```

https://v2.tauri.app/develop/debug/

## Consoles and devtools

- Rust panics and `println!` go to the terminal running `tauri dev`. Backtraces: `RUST_BACKTRACE=1 tauri dev` (PowerShell: `$env:RUST_BACKTRACE=1`). https://v2.tauri.app/develop/debug/
- Webview inspector: right-click > Inspect Element, `Ctrl+Shift+I` (Windows/Linux) or `Cmd+Option+I` (macOS). It is WebKit's inspector on Linux, Safari's on macOS, Edge DevTools on Windows. https://v2.tauri.app/develop/debug/
- Default: the inspector exists only in dev and debug builds. Enable it in release with the `devtools` Cargo feature on `tauri`. https://v2.tauri.app/develop/debug/
- Danger: devtools on macOS use a private API, which blocks App Store acceptance. Keep `devtools` out of App Store builds. https://v2.tauri.app/develop/debug/
- Programmatic: `window.open_devtools()` / `close_devtools()`, guarded by `#[cfg(debug_assertions)]`. https://v2.tauri.app/develop/debug/
- CrabNebula DevTools (`tauri-plugin-devtools`) shows logs, spans, command and event payloads, config and assets. Register it as early as possible and only in debug builds: https://v2.tauri.app/develop/debug/crabnebula-devtools/

```rust
#[cfg(debug_assertions)]
let devtools = tauri_plugin_devtools::init();
let mut builder = tauri::Builder::default();
#[cfg(debug_assertions)]
{ builder = builder.plugin(devtools); }
```

https://v2.tauri.app/develop/debug/crabnebula-devtools/

## Debug builds

- `tauri build --debug` (npm: `npm run tauri build -- --debug`) outputs to `src-tauri/target/debug/bundle`. https://v2.tauri.app/develop/debug/
- Run the built binary `src-tauri/target/(release|debug)/<app>` from a terminal to see Rust errors; a double-click closes the console on crash. https://v2.tauri.app/develop/debug/

## IDE debugging of the core process

- IDEs build with Cargo directly, bypassing the CLI: `beforeDevCommand` and `beforeBuildCommand` do NOT run. Start the dev server yourself or via a pre-launch task. https://v2.tauri.app/develop/debug/vscode/
- Build with `--no-default-features` so the app loads from the dev server instead of embedded assets (the CLI normally passes it). https://v2.tauri.app/develop/debug/rustrover/
- VS Code with vscode-lldb, `.vscode/launch.json`: https://v2.tauri.app/develop/debug/vscode/

```json
{
  "type": "lldb",
  "request": "launch",
  "name": "Tauri Development Debug",
  "cargo": { "args": ["build", "--manifest-path=./src-tauri/Cargo.toml", "--no-default-features"] },
  "preLaunchTask": "ui:dev"
}
```

https://v2.tauri.app/develop/debug/vscode/

- The `ui:dev` task must be `"isBackground": true`, since the dev server keeps running. For production debugging use `cargo build --release` with a `ui:build` pre-launch task. https://v2.tauri.app/develop/debug/vscode/
- On Windows, the `cppvsdbg` debugger (C/C++ extension) is usually faster than lldb and shows Rust enums better. Point `program` at `src-tauri/target/debug/<app>.exe`. https://v2.tauri.app/develop/debug/vscode/
- RustRover/CLion: without a root `Cargo.toml`, attach `src-tauri/Cargo.toml` or add `[workspace] members = ["src-tauri"]`. Use two run configs (Cargo with `--no-default-features`, plus the dev server) and start the dev server first. https://v2.tauri.app/develop/debug/rustrover/
- Neovim: `nvim-dap` with the `codelldb` adapter; `overseer.nvim` can run the dev server from `.vscode/tasks.json`. https://v2.tauri.app/develop/debug/neovim/

## Linux WebKitGTK graphics issues

- Symptoms: blank or white window, flicker or silent death on resize, `AcceleratedSurfaceDMABuf was unable to construct a complete framebuffer`, `Gdk-Message: Error 71 (Protocol error) dispatching to Wayland display.` Most often NVIDIA GPUs. https://v2.tauri.app/develop/debug/linux-graphics/
- Try in order (earlier steps keep acceleration): enable kernel mode setting (`nvidia_drm.modeset=1` for drivers older than 545), `__NV_DISABLE_EXPLICIT_SYNC=1`, `WEBKIT_DISABLE_DMABUF_RENDERER=1`, and last `WEBKIT_DISABLE_COMPOSITING_MODE=1`. https://v2.tauri.app/develop/debug/linux-graphics/
- Set it in `main()` before any webview exists, but ship an unconditional override only if your users are verified affected, since it slows everyone down: https://v2.tauri.app/develop/debug/linux-graphics/

```rust
#[cfg(target_os = "linux")]
std::env::set_var("WEBKIT_DISABLE_DMABUF_RENDERER", "1");
```

https://v2.tauri.app/develop/debug/linux-graphics/

- WebGL2 creation succeeds even on a software rasterizer, and WebKitGTK reports `Apple GPU` as renderer on every Linux machine. Give WebGL-heavy views a non-WebGL fallback on Linux. https://v2.tauri.app/develop/debug/linux-graphics/
- A black webview under Flatpak on Wayland may be solved by the optional `--env=WEBKIT_DISABLE_COMPOSITING_MODE=1` finish-arg. https://v2.tauri.app/distribute/flatpak/
