# Distributing on Windows and macOS: installers, WebView2, app bundle, DMG

## Contents

- Windows installer formats
- WebView2 install mode
- NSIS options and hooks
- WiX (MSI) options
- Cross-compiling and other targets
- macOS app bundle
- DMG

## Windows installer formats

- Tauri builds `.msi` (WiX Toolset v3, Windows hosts only) and `-setup.exe` (NSIS). https://v2.tauri.app/distribute/windows-installer/
- MSI builds need the VBSCRIPT optional feature; `failed to run light.exe` means it is off. https://v2.tauri.app/distribute/windows-installer/
- FIPS-compliant MSI: set `TAURI_BUNDLER_WIX_FIPS_COMPLIANT=true` before building. https://v2.tauri.app/distribute/windows-installer/
- The second segment of the identifier (`tauri-apps` in `com.tauri-apps.test`) is used as the company name (`MyCompany`) in the registry key of the WiX fragment example. https://v2.tauri.app/distribute/windows-installer/

## WebView2 install mode

`bundle > windows > webviewInstallMode > type`: https://v2.tauri.app/distribute/windows-installer/

| Mode | Needs internet | Extra size | Use |
| :-- | :-- | :-- | :-- |
| `downloadBootstrapper` | yes | 0 | default |
| `embedBootstrapper` | yes | ~1.8 MB | Windows 7 with `.msi` |
| `offlineInstaller` | no | ~127 MB | offline environments (embeds the WebView2 installer) |
| `fixedRuntime` | no | ~180 MB | pinned WebView2 you must patch yourself |
| `skip` | no | 0 | not recommended: the app will not run without WebView2 |

https://v2.tauri.app/distribute/windows-installer/

- Keep the system runtime when you can: Windows patches it for you. https://v2.tauri.app/distribute/windows-installer/
- `fixedRuntime` points `path` at the folder extracted from the `.cab` (use the real extracted folder name). https://v2.tauri.app/distribute/windows-installer/
- `bundle > windows > minimumWebview2Version` makes the installer update an older runtime (needed for features such as custom URI schemes). https://v2.tauri.app/distribute/windows-installer/
- Windows 7: the default `.msi` fails (bootstrapper download needs TLS 1.2), so use `embedBootstrapper`. https://v2.tauri.app/distribute/windows-installer/

## NSIS options and hooks

- `bundle > windows > nsis > installMode`: the default (current user) installs to `%LOCALAPPDATA%` without admin; `perMachine` installs to Program Files and needs admin; `both` lets the user pick and needs admin. https://v2.tauri.app/distribute/windows-installer/
- Prefer hooks to a full custom template: define `NSIS_HOOK_PREINSTALL`, `NSIS_HOOK_POSTINSTALL`, `NSIS_HOOK_PREUNINSTALL`, `NSIS_HOOK_POSTUNINSTALL` macros in a `.nsh` and set `installerHooks`. https://v2.tauri.app/distribute/windows-installer/

```json
{ "bundle": { "windows": { "nsis": { "installerHooks": "./windows/hooks.nsh" } } } }
```

https://v2.tauri.app/distribute/windows-installer/

- Pitfall: inside a macro, `${__FILEDIR__}` expands to the generated script's dir, not the hook's. Capture it at top level: `!define HOOK_FILE_DIR "${__FILEDIR__}"`. https://v2.tauri.app/distribute/windows-installer/
- Installing a dependency from a hook: check it is missing first, run it with `/passive` or `/quiet` plus `/norestart`, ship it via `bundle > resources` (found at `$INSTDIR\resources\`), and remember other apps may share it at uninstall. https://v2.tauri.app/distribute/windows-installer/
- One NSIS installer holds all `languages`; it follows the OS language unless `displayLanguageSelector: true`. https://v2.tauri.app/distribute/windows-installer/

## WiX (MSI) options

- `wix > template` replaces the `.wxs` template. Fragments go in `src-tauri/windows/fragments`, listed in `fragmentPaths`, and each `ComponentGroup`, `Component`, `FeatureGroup`, `Feature` or `Merge` must also appear in the matching `*Refs` array or it is ignored. https://v2.tauri.app/distribute/windows-installer/
- `wix > language`: a string, an array (one MSI per language) or an object with `localePath` to a `.wxl` whose `Culture` matches. https://v2.tauri.app/distribute/windows-installer/

## Cross-compiling and other targets

- From Linux or macOS only NSIS works, as a last resort: install `nsis`, `lld` and `llvm` (for `llvm-rc`), `rustup target add x86_64-pc-windows-msvc`, `cargo install --locked cargo-xwin`, then: https://v2.tauri.app/distribute/windows-installer/

```sh
cargo tauri build --runner cargo-xwin --target x86_64-pc-windows-msvc
```

https://v2.tauri.app/distribute/windows-installer/

- Only the MSVC target is officially supported. Signing a cross-compiled installer needs a custom `signCommand`. https://v2.tauri.app/distribute/windows-installer/ https://v2.tauri.app/distribute/sign/windows/
- ARM64: `--target aarch64-pc-windows-msvc` with the "C++ ARM64 build tools" installed. The NSIS installer stays x86 (emulated); the app is native ARM64. https://v2.tauri.app/distribute/windows-installer/

## macOS app bundle

- `tauri build --bundles app`. Tauri writes the key `Info.plist` entries; a `src-tauri/Info.plist` is merged on top. Overriding generated keys such as the version can conflict with the config. https://v2.tauri.app/distribute/macos-application-bundle/
- Localize Info.plist with `<lang>.lproj/InfoPlist.strings` shipped as resources, for example `"resources": { "infoplist/**": "./" }`. https://v2.tauri.app/distribute/macos-application-bundle/
- Entitlements: `bundle > macOS > entitlements: "./Entitlements.plist"`, applied only when signing. https://v2.tauri.app/distribute/macos-application-bundle/
- Minimum macOS defaults to 10.13; change `bundle > macOS > minimumSystemVersion`. https://v2.tauri.app/distribute/macos-application-bundle/
- `bundle > macOS > frameworks`: system frameworks by bare name, or local frameworks/dylibs by path relative to `src-tauri`. `bundle > macOS > files` maps paths inside `Contents/`. https://v2.tauri.app/distribute/macos-application-bundle/
- GUI apps do not inherit the shell `$PATH`: use `fix-path-env-rs`. https://v2.tauri.app/distribute/macos-application-bundle/

## DMG

- `tauri build --bundles dmg` on a Mac. Layout under `bundle > macOS > dmg`: `background`, `windowSize` (default 660x400), `windowPosition`, `appPosition`, `applicationFolderPosition`. https://v2.tauri.app/distribute/dmg/
- Known issue: icon positions and sizes are not applied when the DMG is built on CI. https://v2.tauri.app/distribute/dmg/
- Both macOS distribution routes (App Store, direct DMG download) require code signing, and shipping outside the App Store also requires notarization. https://v2.tauri.app/distribute/
