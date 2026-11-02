# Official plugins: app integration (desktop) and device features (mobile)

## Contents

- Platform availability
- single-instance
- deep-link
- autostart, cli, global-shortcut
- window-state, positioner
- notification, process, os, clipboard
- localhost
- Mobile device plugins

## Platform availability

- Desktop only: autostart, cli, global-shortcut, positioner, single-instance, window-state. Mobile only: barcode-scanner, biometric, geolocation, haptics, nfc. https://v2.tauri.app/plugin/autostart/ https://v2.tauri.app/plugin/barcode-scanner/ https://v2.tauri.app/plugin/positioner/ https://v2.tauri.app/plugin/nfc/
- Keep platform-specific permissions in capability files with `platforms` set, using `mobile-schema.json` or `desktop-schema.json`. https://v2.tauri.app/plugin/barcode-scanner/

## single-instance

- Register it FIRST, before every other plugin. https://v2.tauri.app/plugin/single-instance/
- `init(|app, args, cwd| {})` runs in the running instance when a second one starts; the second one is closed. Default does nothing: focus the window yourself. https://v2.tauri.app/plugin/single-instance/

```rust
.plugin(tauri_plugin_single_instance::init(|app, args, cwd| {
    let _ = app.get_webview_window("main").expect("no main window").set_focus();
}))
```

https://v2.tauri.app/plugin/single-instance/

- Linux uses DBus `org.{id}.SingleInstance`, `{id}` being the identifier with `.` and `-` turned into `_`. Snap needs a dbus `slots`/`plugs` pair, Flatpak `--talk-name` and `--own-name` finish args. https://v2.tauri.app/plugin/single-instance/
- No JS API, no capability needed. https://v2.tauri.app/plugin/single-instance/

## deep-link

- On Windows and Linux a deep link starts a NEW process with the URL as an argument. Keep one instance with single-instance and its `deep-link` feature: https://v2.tauri.app/plugin/deep-linking/

```toml
tauri-plugin-single-instance = { version = "2.0.0", features = ["deep-link"] }
```

https://v2.tauri.app/plugin/deep-linking/

- Config: `plugins.deep-link.desktop.schemes` and `plugins.deep-link.mobile: [{ scheme, host, pathPrefix, appLink }]`. https://v2.tauri.app/plugin/deep-linking/

```json
{ "plugins": { "deep-link": { "desktop": { "schemes": ["my-tauri-app"] } } } }
```

https://v2.tauri.app/plugin/deep-linking/

- JS: `getCurrent()` for a cold start, `onOpenUrl(cb)` while running (URLs come as a list). Permissions: `deep-link:default` plus `core:event:default`. https://v2.tauri.app/plugin/deep-linking/
- Caution: anyone can launch the app with a fake URL argument. Validate the URL. Schemes registered at runtime are not matched for you: check `Env::args_os` yourself, also in the single-instance callback. https://v2.tauri.app/plugin/deep-linking/
- Schemes register at install time only. `app.deep_link().register_all()` also works in dev on Linux and Windows (`#[cfg(any(target_os = "linux", all(debug_assertions, windows)))]`). macOS cannot register at runtime: test the bundle installed in `/Applications`. https://v2.tauri.app/plugin/deep-linking/
- Moving an AppImage breaks its registered link (absolute path). https://v2.tauri.app/plugin/deep-linking/
- Verified links: Android App Links need `.well-known/assetlinks.json` with `package_name` = identifier with `-` turned into `_`. iOS Universal Links need `.well-known/apple-app-site-association` served over HTTPS as `application/json`, with `appIDs: ["$DEVELOPMENT_TEAM_ID.$APP_BUNDLE_ID"]`. Custom schemes need neither. https://v2.tauri.app/plugin/deep-linking/
- Test: Windows `start <scheme>://url`, Linux `xdg-open`, iOS simulator `xcrun simctl openurl booted <url>`, Android `adb shell am start -a android.intent.action.VIEW -d <url> <identifier>`. https://v2.tauri.app/plugin/deep-linking/

## autostart, cli, global-shortcut

- autostart: `tauri_plugin_autostart::init(MacosLauncher::LaunchAgent, Some(vec!["--flag1"]))`; JS `enable()`, `isEnabled()`, `disable()`. https://v2.tauri.app/plugin/autostart/
- cli: clap-based args declared in `plugins.cli` (`args`, `subcommands`); read with `getMatches()` or `app.cli().matches()`. On Windows, release apps cannot print back to the calling console by default. https://v2.tauri.app/plugin/cli/
- global-shortcut: the Rust handler fires on both `ShortcutState::Pressed` and `Released`, so check `event.state()`. JS accelerators look like `'CommandOrControl+Shift+C'`. https://v2.tauri.app/plugin/global-shortcut/

```rust
tauri_plugin_global_shortcut::Builder::new()
    .with_handler(|_app, shortcut, event| { /* match event.state() */ })
    .build()
```

https://v2.tauri.app/plugin/global-shortcut/

## window-state, positioner

- window-state saves every window on close and restores it on launch. Create windows with `visible: false` to avoid a flash: the plugin shows them once restored. https://v2.tauri.app/plugin/window-state/
- positioner: `moveWindow(Position.TopRight)`. Tray-relative positions need its `tray-icon` feature and `tauri_plugin_positioner::on_tray_event(app, &event)` in `on_tray_icon_event`. https://v2.tauri.app/plugin/positioner/

## notification, process, os, clipboard

- notification: check `isPermissionGranted()`, else `requestPermission()`, then `sendNotification({ title, body })`. https://v2.tauri.app/plugin/notification/
- Create channels before sending notifications that reference them: invalid channel IDs prevent display (channels are mainly Android). Actions are mobile only. https://v2.tauri.app/plugin/notification/
- Windows 7 notifications need the plugin's `windows7-compat` feature. https://v2.tauri.app/distribute/windows-installer/
- process: `exit(code)` and `relaunch()`; use shell for child processes. https://v2.tauri.app/plugin/process/
- os: `platform()` is fixed at compile time. https://v2.tauri.app/plugin/os-info/
- clipboard: crate `tauri-plugin-clipboard-manager`, `tauri add clipboard-manager`; JS `writeText`, `readText`. https://v2.tauri.app/plugin/clipboard/

## localhost

Serves assets over a localhost port instead of the custom protocol. The docs flag considerable security risk: keep the default protocol unless you know why you need it. https://v2.tauri.app/plugin/localhost/

## Mobile device plugins

- iOS usage strings go in `src-tauri/Info.ios.plist`: `NSCameraUsageDescription` (barcode-scanner), `NSFaceIDUsageDescription` (biometric), `NSLocationWhenInUseUsageDescription` (geolocation), `NFCReaderUsageDescription` (nfc). https://v2.tauri.app/plugin/barcode-scanner/ https://v2.tauri.app/plugin/biometric/ https://v2.tauri.app/plugin/geolocation/ https://v2.tauri.app/plugin/nfc/
- barcode-scanner: `scan({ windowed: true, formats: [Format.QRCode] })` makes the webview transparent over the camera, so the UI must show through. https://v2.tauri.app/plugin/barcode-scanner/
- biometric: `checkStatus()` then `authenticate(reason, { allowDeviceCredential })`, which rejects on failure. https://v2.tauri.app/plugin/biometric/
- geolocation: `checkPermissions()`; on `prompt` or `prompt-with-rationale` call `requestPermissions(['location'])`; then `getCurrentPosition()` or `watchPosition(opts, cb)`. Add `<uses-feature android:name="android.hardware.location.gps" android:required="true" />` only if GPS is mandatory (Play filters devices on it). https://v2.tauri.app/plugin/geolocation/
- haptics: Android has no vibration standard, so feedback may do nothing on cheaper phones. https://v2.tauri.app/plugin/haptics/
- nfc: check `isAvailable()` first. iOS needs iOS 14+ and the NFC Tag Reading capability (entitlement `com.apple.developer.nfc.readersession.formats`). URI, MIME and tech filters work only on Android, so always check the tag contents. https://v2.tauri.app/plugin/nfc/
- nfc: without `keepSessionAlive: true` on `scan`, a later `write()` makes the user rescan the tag. https://v2.tauri.app/plugin/nfc/
