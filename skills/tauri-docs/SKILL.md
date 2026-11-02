---
name: tauri-docs
description: Tauri 2 apps, tauri.conf.json, capabilities and permissions, commands and invoke, events and channels, plugins, bundling, signing and the updater, migration from Tauri 1. Guides Tauri 2 work from the official v2 documentation, including scopes, managed state, plugin authoring, CSP and the isolation pattern, testing and debugging, Linux, Windows and macOS packages, and mobile and store releases. Use when writing or reviewing a Tauri 2 app, adding a plugin or a capability, fixing a "not allowed" or "forbidden path" error, wiring IPC between Rust and the webview, packaging, signing or publishing a desktop or mobile build, or upgrading a Tauri 1 or 2.0 beta project.
metadata:
  version: "Tauri 2.x (docs at v2.tauri.app, unversioned beyond major 2)"
  source: https://v2.tauri.app/
  fetched: "2026-11-02"
---

# Tauri 2 documentation guide

## Overview

Tauri builds desktop and mobile apps from a Rust core and a web frontend rendered by the system webview (WebView2 on Windows, WKWebView on macOS and iOS, WebKitGTK on Linux). This skill condenses the official Tauri 2 guides into pitfalls, surprising defaults, recommended patterns and Tauri 1 replacements. Every claim below and in the reference files ends with the URL of the page it comes from, so it can be checked.

## Version covered

Tauri 2.x, as documented at https://v2.tauri.app/. The docs are not versioned beyond major 2. Where a feature needs a specific minor (for example `removeUnusedCommands` or event mocking), the reference file says so. Generated API, config and CLI reference pages are not covered: check them on the site for exact signatures.

## How to navigate

1. Read the quick start below: it holds the rules that cause most failures.
2. Open only the reference file for the task, from the table at the end.
3. Each reference file is self-contained and starts with its contents list.
4. To verify a claim, open the URL that follows it.

## Quick start: rules and pitfalls

### Project shape

- App code lives in `src-tauri/src/lib.rs` under `#[cfg_attr(mobile, tauri::mobile_entry_point)]`. `main.rs` only calls `app_lib::run()`. https://v2.tauri.app/start/project-structure/
- Tauri is a static host: ship SSG or SPA output. SSR does not run. Fix the dev port (`strictPort: true`) to match `build.devUrl`, and exclude `src-tauri` from the frontend watcher. https://v2.tauri.app/start/frontend/vite/
- Windows renders with Chromium, macOS and Linux with WebKit: test both engines. https://v2.tauri.app/start/frontend/vite/

### Security model

- Every file in `src-tauri/capabilities/` is active until `app.security.capabilities` lists identifiers. Then only the listed ones apply. https://v2.tauri.app/security/capabilities/
- Capabilities match window labels, not titles. A window in several capabilities gets the union of all of them. https://v2.tauri.app/security/capabilities/
- Plugin commands are denied until a capability grants a permission. A permission without a scope still fails: `fs:allow-exists` alone gives `forbidden path`. https://v2.tauri.app/learn/security/using-plugin-permissions/
- App commands registered in `invoke_handler` are allowed for every window unless declared through `AppManifest::commands` in `build.rs`. https://v2.tauri.app/security/capabilities/
- `deny` always wins over `allow`. Wildcards skip Unix dotfolders (`requireLiteralLeadingDot`). https://v2.tauri.app/security/scope/ https://v2.tauri.app/plugin/file-system/
- CSP applies only when set in `app.security.csp`. Prefer the isolation pattern to validate IPC. https://v2.tauri.app/security/csp/ https://v2.tauri.app/concept/inter-process-communication/isolation/
- Core permissions carry the `core:` prefix. `core:default` bundles them. https://v2.tauri.app/start/migrate/from-tauri-2-beta/

### IPC

- JS argument keys are camelCase (`invokeMessage` for `invoke_message`). https://v2.tauri.app/develop/calling-rust/
- Only the last `invoke_handler` call counts: put every command in one `generate_handler!`. https://v2.tauri.app/develop/calling-rust/
- Async commands with borrowed arguments (`&str`, `State<'_, T>`) must return a `Result`. https://v2.tauri.app/develop/calling-rust/
- A wrong `State<T>` type compiles and fails at runtime with `state not managed for field ...`. Do not wrap state in `Arc`. https://v2.tauri.app/develop/state-management/
- Use a `Channel` for streams and ordered data. Events are untyped JSON, slow for large payloads, and webview-targeted events skip global listeners. https://v2.tauri.app/develop/calling-frontend/
- `listen` returns a Promise of the unlisten function: await it and call it on unmount. https://v2.tauri.app/develop/_sections/frontend-listen/

### Config and dependencies

- Platform config files merge objects key by key but REPLACE arrays whole (`bundle.resources`, `app.windows`). https://v2.tauri.app/develop/configuration-files/
- Keep `@tauri-apps/api` and `tauri` on the same minor, and each plugin's npm package and crate on the exact same version. https://v2.tauri.app/develop/updating-dependencies/
- Resolve bundled files with `resolveResource` / `BaseDirectory::Resource`, never by hand: `../` becomes `_up_`. https://v2.tauri.app/develop/resources/

### Shipping

- The updater cannot run unsigned. `TAURI_SIGNING_PRIVATE_KEY` must be a real env var (`.env` is ignored), `pubkey` is key content, and losing the private key ends updates for installed apps. https://v2.tauri.app/plugin/updater/
- Nothing checks for updates unless your code calls `check()`. https://v2.tauri.app/plugin/updater/
- Build Linux packages on the oldest distro you support, or users hit `GLIBC_x.yy not found`. https://v2.tauri.app/distribute/debian/ https://v2.tauri.app/distribute/appimage/
- macOS outside the App Store needs Developer ID signing plus notarization. Without a certificate, sign ad-hoc (`signingIdentity: "-"`). https://v2.tauri.app/distribute/sign/macos/
- The `devtools` feature uses private macOS APIs and blocks App Store review. https://v2.tauri.app/develop/debug/
- Register single-instance first, and fs before persisted-scope. https://v2.tauri.app/plugin/single-instance/ https://v2.tauri.app/plugin/persisted-scope/

### Migrating from Tauri 1

- Run `tauri migrate`, then review: the allowlist becomes capabilities and `tauri::api` becomes plugins. https://v2.tauri.app/start/migrate/from-tauri-1/
- Imports move from `@tauri-apps/api/tauri` to `@tauri-apps/api/core`. https://v2.tauri.app/start/migrate/from-tauri-1/
- On Windows the origin moves from `https://tauri.localhost` to `http://`, which resets LocalStorage and IndexedDB unless `useHttpsScheme: true` is set. https://v2.tauri.app/start/migrate/from-tauri-1/

## Reference files

| File | Read when |
| :-- | :-- |
| [references/getting-started.md](references/getting-started.md) | Installing prerequisites, scaffolding, wiring Vite/Next/Nuxt/SvelteKit/Trunk, running `tauri dev` on desktop or a mobile device |
| [references/migration-from-v1.md](references/migration-from-v1.md) | Upgrading a Tauri 1 or 2.0 beta app: config key moves, removed features, renamed APIs and env vars |
| [references/security-capabilities.md](references/security-capabilities.md) | Writing capabilities, permissions, permission sets or scopes, or debugging a "not allowed" error |
| [references/security-csp-isolation.md](references/security-csp-isolation.md) | Setting CSP, the isolation pattern, the asset protocol scope, response headers, or hardening the build chain |
| [references/ipc-commands-and-state.md](references/ipc-commands-and-state.md) | Defining commands, arguments, errors, async commands, injected parameters and managed state |
| [references/ipc-events-and-channels.md](references/ipc-events-and-channels.md) | Emitting and listening to events, streaming with channels, evaluating JS from Rust |
| [references/configuration-and-resources.md](references/configuration-and-resources.md) | Config formats and platform merges, build variants, version sync, icons, bundled resources, binary size |
| [references/plugins-files-network-data.md](references/plugins-files-network-data.md) | Installing any official plugin, and using fs, dialog, opener, shell, http, upload, websocket, sql, store, stronghold or log |
| [references/plugins-app-and-mobile.md](references/plugins-app-and-mobile.md) | single-instance, deep-link, autostart, cli, global-shortcut, window-state, positioner, notification, and mobile device plugins |
| [references/plugin-authoring.md](references/plugin-authoring.md) | Writing a plugin: naming, config, hooks, command permissions, scopes, Kotlin and Swift code |
| [references/debugging.md](references/debugging.md) | Dev-only code, devtools in release, IDE debugging, Linux blank window or WebGL issues |
| [references/testing.md](references/testing.md) | Mocking IPC in unit tests, WebDriver end-to-end tests, running them in CI |
| [references/distribute-linux.md](references/distribute-linux.md) | Building deb, rpm, AppImage, Flatpak, Snap or AUR packages, and ARM Linux builds |
| [references/distribute-windows-macos.md](references/distribute-windows-macos.md) | Windows MSI/NSIS installers, WebView2 install modes, macOS app bundle and DMG |
| [references/distribute-mobile-and-stores.md](references/distribute-mobile-and-stores.md) | Android signing and Google Play, iOS signing, App Store for iOS and macOS, Microsoft Store |
| [references/code-signing-desktop.md](references/code-signing-desktop.md) | Signing and notarizing on macOS, signing on Windows with a certificate or `signCommand` |
| [references/updater-and-ci.md](references/updater-and-ci.md) | Setting up the updater, update manifests and channels, and the tauri-action release workflow |
| [references/recipes-windows-tray-menu.md](references/recipes-windows-tray-menu.md) | Window labels, custom titlebars, system tray, window and app menus |
| [references/recipes-sidecar-splash-mobile.md](references/recipes-sidecar-splash-mobile.md) | Sidecar binaries, a Node.js sidecar, splashscreens, file associations, multi-window on mobile |

Sources and fetch details: [SOURCES.md](SOURCES.md).
