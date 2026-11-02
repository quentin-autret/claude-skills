# Distributing to stores: Google Play, App Store (iOS and macOS), Microsoft Store

## Contents

- Per-store config files
- Android signing
- Google Play
- iOS signing
- App Store: iOS
- App Store: macOS
- Microsoft Store

## Per-store config files

Keep store-only settings in a separate config merged at bundle time, so other builds do not need store assets such as provisioning profiles. https://v2.tauri.app/distribute/ https://v2.tauri.app/distribute/app-store/

```sh
tauri build --no-bundle
tauri bundle --bundles app --target universal-apple-darwin --config src-tauri/tauri.appstore.conf.json
```

https://v2.tauri.app/distribute/app-store/

## Android signing

- Create an upload keystore with `keytool` (if it is not on PATH, use the one in Android Studio's `jbr/bin`). https://v2.tauri.app/distribute/sign/android/

```sh
keytool -genkey -v -keystore ~/upload-keystore.jks -keyalg RSA -keysize 2048 -validity 10000 -alias upload
```

https://v2.tauri.app/distribute/sign/android/

- Write `src-tauri/gen/android/keystore.properties` with `keyAlias`, `password`, `storeFile`, and keep it and the keystore out of public source control. In CI, generate it from secrets: https://v2.tauri.app/distribute/sign/android/

```yml
- name: setup Android signing
  run: |
    cd src-tauri/gen/android
    echo "keyAlias=${{ secrets.ANDROID_KEY_ALIAS }}" > keystore.properties
    echo "password=${{ secrets.ANDROID_KEY_PASSWORD }}" >> keystore.properties
    base64 -d <<< "${{ secrets.ANDROID_KEY_BASE64 }}" > $RUNNER_TEMP/keystore.jks
    echo "storeFile=$RUNNER_TEMP/keystore.jks" >> keystore.properties
```

https://v2.tauri.app/distribute/sign/android/

- Edit `src-tauri/gen/android/app/build.gradle.kts`, the one with a `buildTypes` block, not the other `build.gradle.kts` files. Add `import java.io.FileInputStream`, a `signingConfigs { create("release") { ... } }` block before `buildTypes` reading the properties file, and `signingConfig = signingConfigs.getByName("release")` in `getByName("release")`. https://v2.tauri.app/distribute/sign/android/

## Google Play

- Build an AAB: `tauri android build --aab` (`--apk` for testing or sideloading). All ABIs (aarch64, armv7, i686, x86_64) are built by default; narrow with `--target aarch64 --target armv7`. `--split-per-abi` is only for outside Play. https://v2.tauri.app/distribute/google-play/
- Output: `gen/android/app/build/outputs/bundle/universalRelease/app-universal-release.aab`. https://v2.tauri.app/distribute/google-play/
- `versionCode` is derived as `major*1000000 + minor*1000 + patch` from `version`; override with `bundle > android > versionCode`. https://v2.tauri.app/distribute/google-play/
- Minimum is Android 7.0 (SDK 24); raise it with `bundle > android > minSdkVersion`. https://v2.tauri.app/distribute/google-play/
- The first upload must be done by hand in the Play Console; Tauri has no release automation for Play. https://v2.tauri.app/distribute/google-play/

## iOS signing

- Needs a paid Apple Developer membership, an Apple device, the bundle ID registered in App Store Connect, a certificate and a provisioning profile. https://v2.tauri.app/distribute/sign/ios/
- Default: Xcode automatic signing (local account in Xcode Settings > Accounts). In CI, set `APPLE_API_ISSUER`, `APPLE_API_KEY` and `APPLE_API_KEY_PATH` from an App Store Connect API key with Admin access (the key downloads only once). https://v2.tauri.app/distribute/sign/ios/
- Manual signing: `IOS_CERTIFICATE` (base64 `.p12`), `IOS_CERTIFICATE_PASSWORD`, `IOS_MOBILE_PROVISION` (base64 profile). https://v2.tauri.app/distribute/sign/ios/
- Certificate per export method: Apple Development for debugging, Apple Distribution for `app-store-connect` and `ad-hoc`. https://v2.tauri.app/distribute/sign/ios/

## App Store: iOS

- The App Store Connect Bundle ID must equal `identifier`. Build on macOS. https://v2.tauri.app/distribute/app-store/
- After `tauri ios init`, regenerate icons with `tauri icon /path/to/app-icon.png --ios-color '#fff'`. https://v2.tauri.app/distribute/app-store/
- `CFBundleVersion` comes from `version`; override with `bundle > iOS > bundleVersion`. https://v2.tauri.app/distribute/app-store/
- Build and upload (or archive in Xcode via `tauri ios build --open`): https://v2.tauri.app/distribute/app-store/

```sh
tauri ios build --export-method app-store-connect
xcrun altool --upload-app --type ios --file "src-tauri/gen/apple/build/arm64/$APPNAME.ipa" \
  --apiKey $APPLE_API_KEY_ID --apiIssuer $APPLE_API_ISSUER
```

https://v2.tauri.app/distribute/app-store/

- `altool` looks for `AuthKey_<APPLE_API_KEY_ID>.p8` in `./private_keys`, `~/private_keys`, `~/.private_keys` or `~/.appstoreconnect/private_keys`. Note the names: `APPLE_API_KEY_ID` here, `APPLE_API_KEY` / `APPLE_API_KEY_PATH` for signing and notarization. https://v2.tauri.app/distribute/app-store/

## App Store: macOS

Requirements to pass verification: https://v2.tauri.app/distribute/app-store/

- `bundle > category` set (for example `"Utility"`).
- A "Mac App Store Connect" provisioning profile embedded through `bundle > macOS > files` as `embedded.provisionprofile`.
- `src-tauri/Info.plist` with `ITSAppUsesNonExemptEncryption`.
- `Entitlements.plist` with `com.apple.security.app-sandbox` true, `com.apple.application-identifier` = `$TEAM_ID.$IDENTIFIER` and `com.apple.developer.team-identifier` = `$TEAM_ID`, referenced by `bundle > macOS > entitlements`. The app must work inside the sandbox.
- No `devtools` feature: it uses private macOS APIs that the store rejects. https://v2.tauri.app/develop/debug/

Build a universal app, then a `.pkg` signed with a Mac Installer Distribution certificate (not the app certificate): https://v2.tauri.app/distribute/app-store/

```sh
tauri build --bundles app --target universal-apple-darwin
xcrun productbuild --sign "<identity>" \
  --component "target/universal-apple-darwin/release/bundle/macos/$APPNAME.app" /Applications "$APPNAME.pkg"
xcrun altool --upload-app --type macos --file "$APPNAME.pkg" --apiKey $APPLE_API_KEY_ID --apiIssuer $APPLE_API_ISSUER
```

https://v2.tauri.app/distribute/app-store/

- Apple Silicon only: set `bundle > macOS > minimumSystemVersion` to `12.0`. On an Apple Silicon Mac drop `--target universal-apple-darwin` (output under `target/release`). On an Intel Mac run `rustup target add aarch64-apple-darwin` and use `--target aarch64-apple-darwin`. `bundle > macOS > bundleVersion` overrides `CFBundleVersion`. https://v2.tauri.app/distribute/app-store/

## Microsoft Store

- Tauri produces EXE/MSI only, so register an "EXE or MSI app" pointing to your installer. It must be offline, signed and handle its own updates. https://v2.tauri.app/distribute/microsoft-store/

```json
{ "bundle": { "windows": { "webviewInstallMode": { "type": "offlineInstaller" } } } }
```

```sh
cargo tauri build --no-bundle
cargo tauri bundle --config src-tauri/tauri.microsoftstore.conf.json
```

https://v2.tauri.app/distribute/microsoft-store/

- Silent install is mandatory (rejection `10.2.9.2`): give Partner Center `/S` for NSIS or `/quiet` for MSI. https://v2.tauri.app/distribute/microsoft-store/
- The publisher may not equal the product name. Unset `bundle.publisher` falls back to the second identifier segment, so `productName: "Example"` with `com.example.app` is rejected. Set `bundle.publisher` explicitly. https://v2.tauri.app/distribute/microsoft-store/
