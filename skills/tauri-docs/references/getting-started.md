# Getting started: prerequisites, project layout, frontend, dev loop (Tauri 2)

## Contents

- Prerequisites by platform
- Project layout
- Frontend rules
- Framework setups (Vite, Next.js, Nuxt, SvelteKit, Trunk/Leptos)
- Dev loop
- Mobile dev server

## Prerequisites by platform

- Linux: Tauri 2 needs `webkit2gtk-4.1`. Debian packages: https://v2.tauri.app/start/prerequisites/

```sh
sudo apt install libwebkit2gtk-4.1-dev build-essential curl wget file \
  libxdo-dev libssl-dev libayatana-appindicator3-dev librsvg2-dev
```

https://v2.tauri.app/start/prerequisites/

- Alpine containers ship no fonts: install at least one font package (for example `font-dejavu`) or no text renders. https://v2.tauri.app/start/prerequisites/
- Alpine (musl) links system libs statically: on linker errors for libs pkg-config reports present, install the `*-static` packages (`openssl-libs-static cairo-static harfbuzz-static glib-static wayland-static zlib-static`). https://v2.tauri.app/start/prerequisites/
- macOS: desktop-only work can use Command Line Tools (`xcode-select --install`). iOS needs full Xcode, launched once to finish setup. https://v2.tauri.app/start/prerequisites/
- Windows: Microsoft C++ Build Tools ("Desktop development with C++") plus WebView2, preinstalled on Windows 10 1803+ and later. https://v2.tauri.app/start/prerequisites/
- Windows: the MSVC toolchain must be the default host triple. Fix with `rustup default stable-msvc`. https://v2.tauri.app/start/prerequisites/
- Windows: building `.msi` needs the optional Windows feature VBSCRIPT. Symptom when missing: `failed to run light.exe`. https://v2.tauri.app/start/prerequisites/
- Android: Android Studio, `JAVA_HOME` pointing at Studio's bundled `jbr`, SDK Platform, Platform-Tools, NDK (Side by side), Build-Tools, Command-line Tools, then `ANDROID_HOME` and `NDK_HOME`. https://v2.tauri.app/start/prerequisites/

```sh
rustup target add aarch64-linux-android armv7-linux-androideabi i686-linux-android x86_64-linux-android
```

https://v2.tauri.app/start/prerequisites/

- On Windows, env vars set with `SetEnvironmentVariable(..., "User")` are not seen by running terminals and IDEs: restart them. https://v2.tauri.app/start/prerequisites/
- iOS (macOS only): `rustup target add aarch64-apple-ios x86_64-apple-ios aarch64-apple-ios-sim` and `brew install cocoapods`. https://v2.tauri.app/start/prerequisites/

## Project layout

- A Tauri project is a Cargo project in `src-tauri/` plus an optional JS project at the top level. `tauri.conf.json` is also the marker the CLI uses to find the Rust project. https://v2.tauri.app/start/project-structure/
- `src-tauri/capabilities/` is read by default: a command must be allowed there to be callable from JS. https://v2.tauri.app/start/project-structure/
- App code lives in `src/lib.rs` with `#[cfg_attr(mobile, tauri::mobile_entry_point)]`, because mobile builds compile the app as a library. `src/main.rs` only calls `app_lib::run()`, where `app_lib` matches `[lib] name` in `Cargo.toml`. Edit `lib.rs`, not `main.rs`. https://v2.tauri.app/start/project-structure/
- `build.rs` calls `tauri_build::build()`. https://v2.tauri.app/start/project-structure/
- Manual setup on an existing frontend: add `@tauri-apps/cli` as a dev dependency, then `tauri init` (prompts for app name, window title, assets dir, dev URL, dev and build commands). https://v2.tauri.app/start/create-project/
- The bundle identifier (for example `com.tauri-app.app`) must be unique. https://v2.tauri.app/start/create-project/
- Cargo users install the CLI with `cargo install tauri-cli --version "^2.0.0" --locked`. https://v2.tauri.app/start/create-project/

## Frontend rules

- Tauri acts as a static web host: ship SSG, SPA or classic MPA output. SSR is not supported natively. Talk to remote APIs as a normal client. https://v2.tauri.app/start/frontend/
- Windows renders with Chromium (WebView2), macOS and Linux with WebKit. Set build targets accordingly (`chrome105` vs `safari13`). https://v2.tauri.app/start/frontend/vite/
- Tauri expects a fixed dev port that matches `build.devUrl`, so set `strictPort: true`. https://v2.tauri.app/start/frontend/vite/
- Exclude `src-tauri` from the frontend watcher, so the frontend tooling does not watch Rust files (Nuxt: `ignore: ['**/src-tauri/**']` avoids `EMFILE: too many open files`). https://v2.tauri.app/start/create-project/ https://v2.tauri.app/start/frontend/nuxt/

## Framework setups

Vite (recommended for React, Vue, Svelte, Solid, plain JS):

```json
{ "build": { "beforeDevCommand": "npm run dev", "beforeBuildCommand": "npm run build",
  "devUrl": "http://localhost:5173", "frontendDist": "../dist" } }
```

https://v2.tauri.app/start/frontend/vite/

```js
const host = process.env.TAURI_DEV_HOST;
export default defineConfig({
  clearScreen: false, // keep Rust errors visible
  server: {
    port: 5173, strictPort: true, host: host || false,
    hmr: host ? { protocol: 'ws', host, port: 1421 } : undefined,
    watch: { ignored: ['**/src-tauri/**'] },
  },
  envPrefix: ['VITE_', 'TAURI_ENV_*'],
  build: {
    target: process.env.TAURI_ENV_PLATFORM == 'windows' ? 'chrome105' : 'safari13',
    minify: !process.env.TAURI_ENV_DEBUG,
    sourcemap: !!process.env.TAURI_ENV_DEBUG,
  },
});
```

https://v2.tauri.app/start/frontend/vite/

- Next.js: `output: 'export'`, `images: { unoptimized: true }`, `frontendDist: "../out"`, and `assetPrefix` in dev or assets do not resolve. https://v2.tauri.app/start/frontend/nextjs/

```ts
const isProd = process.env.NODE_ENV === 'production';
const internalHost = process.env.TAURI_DEV_HOST || 'localhost';
export default { output: 'export', images: { unoptimized: true },
  assetPrefix: isProd ? undefined : `http://${internalHost}:3000` };
```

https://v2.tauri.app/start/frontend/nextjs/

- Nuxt: `ssr: false`, `devServer: { host: '0' }` for physical iOS devices, `vite.server.strictPort: true`, `vite.clearScreen: false`, build with the `generate` script, `frontendDist: "../dist"`. https://v2.tauri.app/start/frontend/nuxt/
- SvelteKit: `@sveltejs/adapter-static` with `fallback: 'index.html'`, `export const ssr = false` in `src/routes/+layout.ts`, `frontendDist: "../build"`. With prerendering, `load` functions cannot reach Tauri APIs at build time, so prefer SPA mode. https://v2.tauri.app/start/frontend/sveltekit/
- Trunk/Leptos (Rust WASM): SSG (server-based solutions are not officially supported), `app.withGlobalTauri: true` to reach `window.__TAURI__` through `wasm-bindgen`, and `[serve] ws_protocol = "ws"` so hot reload works on mobile. https://v2.tauri.app/start/frontend/trunk/ https://v2.tauri.app/start/frontend/leptos/

## Dev loop

- With a dev server, set `build.devUrl` and `build.beforeDevCommand`. Without one, point `build.frontendDist` at a folder with `index.html`: the CLI serves it itself. https://v2.tauri.app/develop/
- Caution: that built-in dev server has no authentication or encryption. Never run it on an untrusted network. https://v2.tauri.app/develop/
- `tauri dev` watches `src-tauri` and workspace crates and restarts the app. Opt out with `--no-watch`. Ignore paths with a `.taurignore` (gitignore syntax). https://v2.tauri.app/develop/
- Commit `src-tauri/Cargo.lock`, never `src-tauri/target`. https://v2.tauri.app/develop/
- Tauri APIs only work inside the app window, not in a system browser tab. To use browser devtools, bridge calls with `tauri-invoke-http`. https://v2.tauri.app/develop/

## Mobile dev server

- Run `tauri android dev` / `tauri ios dev`. Pass a device name to skip the prompt: `tauri ios dev 'iPhone 15'`. https://v2.tauri.app/develop/
- For physical iOS devices the dev server must listen on `TAURI_DEV_HOST` (check that variable, not `TAURI_ENV_PLATFORM`). The `internal-ip` package is no longer needed. https://v2.tauri.app/start/migrate/from-tauri-2-beta/
- Physical iOS devices default to the public network address. To use the tunnel instead: open Xcode with the device connected, run `tauri ios dev --force-ip-prompt`, pick the address ending in `::2`. https://v2.tauri.app/start/migrate/from-tauri-2-beta/ https://v2.tauri.app/develop/
- To work in Xcode or Android Studio, run `tauri [android|ios] dev --open` and keep the CLI process alive until the IDE closes. With `--open` on a physical iOS device, also pass `--host`. https://v2.tauri.app/develop/
- The Xcode build phase runs the CLI in Xcode's environment, so PATH tweaks such as Node version managers may be missing. https://v2.tauri.app/develop/
- First iOS run may ask for local network permission: allow it and restart the app. https://v2.tauri.app/develop/
- Inspectors: iOS through Safari's Develop menu (enable Web Inspector on the device), Android through `chrome://inspect`. https://v2.tauri.app/develop/
