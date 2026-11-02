# Configuration, versions, icons, resources, app size

## Contents

- Config file formats
- Platform files and the merge trap
- Build variants with --config
- Versions and dependency sync
- Icons
- Bundled resources
- App size

## Config file formats

- `tauri.conf.json` is the default. JSON5 or TOML (`Tauri.toml`, kebab-case keys allowed) need the `config-json5` / `config-toml` feature on BOTH `tauri` and `tauri-build`. Keys are case-sensitive in all formats. https://v2.tauri.app/develop/configuration-files/

```toml
[build-dependencies]
tauri-build = { version = "2.0.0", features = ["config-json5"] }
[dependencies]
tauri = { version = "2.0.0", features = ["config-json5"] }
```

https://v2.tauri.app/develop/configuration-files/

- `tauri dev` and `tauri build` set the `tauri` crate `features` automatically from the config. https://v2.tauri.app/develop/configuration-files/
- The `tauri` crate reads the config at compile time; `tauri-codegen` embeds, hashes and compresses assets at the same time. https://v2.tauri.app/concept/architecture/

## Platform files and the merge trap

- `tauri.linux.conf.json`, `tauri.windows.conf.json`, `tauri.macos.conf.json`, `tauri.android.conf.json`, `tauri.ios.conf.json` (or `Tauri.<platform>.toml`) are merged with JSON Merge Patch (RFC 7396). https://v2.tauri.app/develop/configuration-files/
- Pitfall: objects merge key by key, but arrays are replaced whole, nested ones too (`bundle.resources`, `app.windows`). A platform `app.windows` entry replaces the base entry, and omitted fields fall back to defaults, not to the base values. Repeat everything you want to keep. https://v2.tauri.app/develop/configuration-files/

```json
{ "bundle": { "resources": ["./linux-assets"] } }
```

This Linux file drops a base `["./resources", "./shared-assets"]` entirely. https://v2.tauri.app/develop/configuration-files/

## Build variants with --config

- `--config <json or path>` extends the config for `dev`, `build`, `bundle` and the `android` / `ios` variants. Use it for a beta app with its own `productName` and `identifier`, or per-store settings. https://v2.tauri.app/develop/configuration-files/ https://v2.tauri.app/distribute/

```sh
npm run tauri build -- --config src-tauri/tauri.beta.conf.json
```

https://v2.tauri.app/develop/configuration-files/

## Versions and dependency sync

- App version: set `version` in `tauri.conf.json`; if absent, Tauri uses `package.version` from `src-tauri/Cargo.toml`. https://v2.tauri.app/distribute/
- Generally keep `tauri`, `tauri-build` and the CLI on the latest version of the same minor (not strictly required); when the app fails to run, check that `tauri` and `tauri-cli` are on the latest patch of their minor. https://v2.tauri.app/develop/configuration-files/
- Keep npm `@tauri-apps/api` and crate `tauri` on the same minor. https://v2.tauri.app/develop/updating-dependencies/
- Keep each plugin's npm package and crate on the exact same version (for example `@tauri-apps/plugin-fs` 2.2.1 with `tauri-plugin-fs` 2.2.1): such changes ship in patch releases. https://v2.tauri.app/develop/updating-dependencies/
- Update: `npm install @tauri-apps/cli@latest @tauri-apps/api@latest`, then `cargo update` in `src-tauri` (pin with `=2.0.0` for exact versions). https://v2.tauri.app/develop/updating-dependencies/ https://v2.tauri.app/develop/configuration-files/

## Icons

- The default icon set is Tauri's logo. Generate yours with `tauri icon [INPUT]` (default `./app-icon.png`; a square PNG or SVG with transparency). It writes desktop icons to `src-tauri/icons` and mobile icons straight into the Xcode and Android Studio projects. https://v2.tauri.app/develop/icons/
- Options: `-o` output dir, `-p` custom PNG sizes (disables the defaults), `--ios-color` iOS background (default `#fff`). https://v2.tauri.app/develop/icons/
- Keep all generated icons even when targeting fewer platforms: PNGs are reused elsewhere. https://v2.tauri.app/develop/icons/
- Manual icons: `.ico` must contain 16, 24, 32, 48, 64 and 256 px layers, 32 px layer first for optimal display in development. PNGs square RGBA 32 bpp. iOS icons must have no transparency. https://v2.tauri.app/develop/icons/
- Run `tauri icon` again after `tauri android init` to update the Android Studio project icons; it also produces Microsoft Store icons. https://v2.tauri.app/distribute/google-play/ https://v2.tauri.app/distribute/microsoft-store/

## Bundled resources

- List files in `bundle.resources`. Array form keeps paths under `$RESOURCE`, with `..` rewritten to `_up_` and an absolute root to `_root_`. https://v2.tauri.app/develop/resources/

```json
{ "bundle": { "resources": ["./path/to/some-file.txt", "/absolute/path/to/textfile.txt",
  "../relative/path/to/jsonfile.json", "some-folder/", "resources/**/*.md"] } }
```

https://v2.tauri.app/develop/resources/

- Map form picks the destination: `{ "resources/": "", "docs/**/*md": "website-docs/" }`. Globs in map form flatten matches into the target folder. https://v2.tauri.app/develop/resources/
- Source syntax: `dir/` recursive, `dir/*` non-recursive, `dir/**/*` all files recursively. `dir/**` and `dir/**/**` are errors (`**` only matches directories). https://v2.tauri.app/develop/resources/
- Resolve paths through the API, never by hand. Rust `app.path().resolve("lang/de.json", BaseDirectory::Resource)?`; JS `resolveResource('lang/de.json')` from `@tauri-apps/api/path`. https://v2.tauri.app/develop/resources/
- Reading from JS needs a command or the fs plugin. Because of `_up_` and `_root_` subfolders, scope with `$RESOURCE/**/*`: https://v2.tauri.app/develop/resources/

```json
{ "permissions": ["core:default", "fs:allow-read-text-file", "fs:allow-resource-read-recursive"] }
```

https://v2.tauri.app/develop/resources/

- Android stores resources as APK assets: resolved paths use `asset://localhost/` and are not file system paths. Read them with the fs plugin, or copy them out first when a real file is needed. https://v2.tauri.app/develop/resources/

## App size

- Recommended release profile: https://v2.tauri.app/concept/size/

```toml
[profile.dev]
incremental = true
[profile.release]
codegen-units = 1
lto = true
opt-level = "s"  # "3" for speed, "z" for smallest
panic = "abort"
strip = true
```

https://v2.tauri.app/concept/size/

- `build.removeUnusedCommands: true` drops commands that no capability allows. It needs `tauri` 2.4, `tauri-build` 2.1, `tauri-plugin` 2.1 and `tauri-cli` 2.4, and ignores ACLs added at runtime. Listing exact commands instead of `default` sets maximizes the gain. https://v2.tauri.app/concept/size/
