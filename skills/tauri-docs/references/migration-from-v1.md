# Migration: Tauri 1 and 2.0 beta to Tauri 2

## Contents

- Run the migrator first
- lib.rs and main.rs split
- tauri.conf.json moves
- Cargo features
- Rust API moves
- JavaScript API moves
- Menu, tray, events, updater
- Environment variables
- Windows origin change
- From 2.0 beta

## Run the migrator first

- `tauri migrate` automates most of the work, including turning the v1 allowlist into a capability file and converting sidecar scopes. The docs warn it does not replace reading the full guide. https://v2.tauri.app/start/migrate/from-tauri-1/ https://v2.tauri.app/develop/sidecar/

```sh
npm install @tauri-apps/cli@latest
npm run tauri migrate
```

https://v2.tauri.app/start/migrate/from-tauri-1/

## lib.rs and main.rs split

Mobile needs the app as a library. Rename `main.rs` to `lib.rs` and recreate a thin `main.rs`. https://v2.tauri.app/start/migrate/from-tauri-1/

```toml
[lib]
name = "app_lib"
crate-type = ["staticlib", "cdylib", "rlib"]
```

```rust
// src-tauri/src/lib.rs
#[cfg_attr(mobile, tauri::mobile_entry_point)]
pub fn run() { /* builder code */ }

// src-tauri/src/main.rs
#![cfg_attr(not(debug_assertions), windows_subsystem = "windows")]
fn main() { app_lib::run(); }
```

https://v2.tauri.app/start/migrate/from-tauri-1/

## tauri.conf.json moves

| Tauri 1 | Tauri 2 |
| :-- | :-- |
| `package > productName`, `package > version` | top-level `productName`, `version` |
| `tauri` | `app` |
| `tauri > allowlist` | capabilities and permissions |
| `allowlist > protocol > assetScope` | `app > security > assetProtocol > scope` |
| `tauri > cli`, `tauri > updater` | `plugins > cli`, `plugins > updater` (`active`, `dialog` removed) |
| `fileDropEnabled` | `app > windows > dragDropEnabled` |
| `systemTray` | `app > trayIcon` |
| `pattern` | `app > security > pattern` |
| `tauri > bundle` | top-level `bundle`; `identifier` top-level |
| `dmg`, `deb`, `appimage` | `bundle > macOS > dmg`, `bundle > linux > deb / appimage` |
| macOS/wix/nsis license fields | `bundle > licenseFile` |
| `webviewFixedRuntimePath` | `bundle > windows > webviewInstallMode` |
| `build > withGlobalTauri` | `app > withGlobalTauri` |
| `build > distDir`, `build > devPath` | `build > frontendDist`, `build > devUrl` |

https://v2.tauri.app/start/migrate/from-tauri-1/

- The binary is no longer renamed after `productName`: add top-level `mainBinaryName` equal to `productName`. https://v2.tauri.app/start/migrate/from-tauri-1/
- Apps using the updater must set `bundle > createUpdaterArtifacts`, to `"v1Compatible"` for apps already shipped with v1. https://v2.tauri.app/start/migrate/from-tauri-1/

## Cargo features

- Removed: `reqwest-client`, `reqwest-native-tls-vendored` (use `native-tls-vendored`), `process-command-api` and `shell-open-api` (shell plugin), `windows7-compat` (now on the notification plugin), `updater` (plugin), `linux-protocol-headers` (now default). `system-tray` is renamed `tray-icon`. https://v2.tauri.app/start/migrate/from-tauri-1/
- New: `linux-protocol-body` (custom protocol request bodies, needs webkit2gtk 2.40). https://v2.tauri.app/start/migrate/from-tauri-1/

## Rust API moves

- `tauri::api` is gone. dialog: `tauri-plugin-dialog`; file: `std::fs`; http: `tauri-plugin-http`; process command and shell: `tauri-plugin-shell`; clipboard: `tauri-plugin-clipboard-manager`; cli: `tauri-plugin-cli`; global shortcut: `tauri-plugin-global-shortcut`; updater: `tauri-plugin-updater`; `api::version`: the `semver` crate. https://v2.tauri.app/start/migrate/from-tauri-1/
- `api::path` is now `Manager::path()`; `process::current_binary` and `restart` are in `tauri::process`; the rewritten `api::ip` module (sic in the page) moved to `tauri::ipc` (see `tauri::ipc::Channel`). https://v2.tauri.app/start/migrate/from-tauri-1/

```rust
let home = app.path().home_dir().expect("failed to get home dir");
let path = app.path().resolve("path/to/something", BaseDirectory::Config)?;
```

https://v2.tauri.app/start/migrate/from-tauri-1/

- `Manager::fs_scope` is replaced by `tauri_plugin_fs::FsExt`; `Env.args` by `Env.args_os`. https://v2.tauri.app/start/migrate/from-tauri-1/
- `RemoteDomainAccessScope::enable_tauri_api` is removed: enable each core plugin with `add_plugin`. https://v2.tauri.app/start/migrate/from-tauri-1/
- Renames: `Window` to `WebviewWindow`, `WindowBuilder` to `WebviewWindowBuilder`, `WindowUrl` to `WebviewUrl`, `get_window` to `get_webview_window`, `parent_window` to `parent_raw`. Multi-webview windows need the `unstable` feature. https://v2.tauri.app/start/migrate/from-tauri-1/
- Each plugin: add `tauri-plugin-<name> = "2"` and `.plugin(tauri_plugin_<name>::init())`. Desktop-only plugins (global shortcut) go under a `cfg(not(any(target_os = "android", target_os = "ios")))` target dependency table. https://v2.tauri.app/start/migrate/from-tauri-1/

## JavaScript API moves

- `@tauri-apps/api` keeps the core modules, among them `core` (was `tauri`), `path`, `event` and `window`. cli, clipboard, dialog, fs, global-shortcut, http, os, notification, process, shell and updater moved to `@tauri-apps/plugin-<name>`. https://v2.tauri.app/start/migrate/from-tauri-1/

```diff
- import { invoke } from "@tauri-apps/api/tauri"
+ import { invoke } from "@tauri-apps/api/core"
```

https://v2.tauri.app/start/migrate/from-tauri-1/

- `@tauri-apps/api/window` becomes `@tauri-apps/api/webviewWindow`; `onMenuClicked` is removed. https://v2.tauri.app/start/migrate/from-tauri-1/
- fs renames: `Dir` to `BaseDirectory`, `createDir` to `mkdir`, `readBinaryFile` to `readFile`, `writeBinaryFile` to `writeFile`, `removeDir`/`removeFile` to `remove`, `renameFile` to `rename`. https://v2.tauri.app/start/migrate/from-tauri-1/
- `shell.open` moved to the opener plugin (`openUrl`, `openPath`). https://v2.tauri.app/plugin/shell/ https://v2.tauri.app/plugin/opener/

## Menu, tray, events, updater

- Menus use `tauri::menu`: `MenuBuilder::new(app)`, `MenuItemBuilder` (replaces `CustomMenuItem`), `SubmenuBuilder`, `PredefinedMenuItem`. `Builder::menu` takes a closure. `Builder::on_menu_event` is replaced by `App::on_menu_event` / `AppHandle::on_menu_event`. https://v2.tauri.app/start/migrate/from-tauri-1/
- `SystemTray` becomes `tauri::tray::TrayIconBuilder`; `on_event` splits into `on_menu_event` and `on_tray_icon_event`. https://v2.tauri.app/start/migrate/from-tauri-1/
- Events: `emit` reaches all listeners; `emit_to` / `emitTo` targets one; `emit_filter` filters on `EventTarget`; `listen_global` is renamed `listen_any`. https://v2.tauri.app/start/migrate/from-tauri-1/
- Updater: the built-in dialog and automatic check are gone. Without code calling `check()` users never get another update. https://v2.tauri.app/start/migrate/from-tauri-1/

```javascript
import { check } from '@tauri-apps/plugin-updater';
import { relaunch } from '@tauri-apps/plugin-process';
const update = await check();
if (update) { await update.downloadAndInstall(); await relaunch(); }
```

https://v2.tauri.app/plugin/updater/

## Environment variables

`TAURI_PRIVATE_KEY` to `TAURI_SIGNING_PRIVATE_KEY`, `TAURI_KEY_PASSWORD` to `TAURI_SIGNING_PRIVATE_KEY_PASSWORD`, `TAURI_SKIP_DEVSERVER_CHECK` to `TAURI_CLI_NO_DEV_SERVER_WAIT`, `TAURI_DEV_SERVER_PORT` to `TAURI_CLI_PORT`, `TAURI_PATH_DEPTH` to `TAURI_CLI_CONFIG_DEPTH`, `TAURI_FIPS_COMPLIANT` to `TAURI_BUNDLER_WIX_FIPS_COMPLIANT`, `TAURI_DEV_WATCHER_IGNORE_FILE` to `TAURI_CLI_WATCHER_IGNORE_FILENAME`, `TAURI_TRAY` to `TAURI_LINUX_AYATANA_APPINDICATOR`, `TAURI_APPLE_DEVELOPMENT_TEAM` to `APPLE_DEVELOPMENT_TEAM`, and `TAURI_PLATFORM`, `TAURI_ARCH`, `TAURI_FAMILY`, `TAURI_PLATFORM_VERSION`, `TAURI_PLATFORM_TYPE`, `TAURI_DEBUG` to their `TAURI_ENV_*` forms. https://v2.tauri.app/start/migrate/from-tauri-1/

## Windows origin change

Production frontends on Windows are now served from `http://tauri.localhost`, not `https://`. IndexedDB, LocalStorage and cookies are therefore reset after upgrading (unless v1 used `dangerousUseHttpScheme`). Keep the old origin with `app > windows > useHttpsScheme: true` or `WebviewWindowBuilder::use_https_scheme`. https://v2.tauri.app/start/migrate/from-tauri-1/

## From 2.0 beta

- Core permissions need the `core:` prefix (`core:path:default`, `core:event:default`, `core:window:default`, ...). `core:default` bundles all core defaults. https://v2.tauri.app/start/migrate/from-tauri-2-beta/
- The mobile dev server no longer listens network-wide: it tunnels to the device and the dev server reads `TAURI_DEV_HOST` (this does not yet apply automatically on physical iOS devices, which default to the public network address; `tauri ios dev --force-ip-prompt` selects the device TUN address). https://v2.tauri.app/start/migrate/from-tauri-2-beta/
- Store plugin: JS `Store` becomes `LazyStore`; Rust `with_store(...)` becomes `let store = app.store(path)?; store.set(...)`. https://v2.tauri.app/plugin/store/
