# Code signing on desktop: macOS (signing, notarization) and Windows

## Contents

- When signing is required
- macOS certificates
- macOS signing identity, local and CI
- macOS notarization
- Ad-hoc signing
- Windows certificates and SmartScreen
- Windows local certificate
- Windows signCommand (Azure and others)
- Old patterns

## When signing is required

- Most platforms require signing. macOS needs it for the App Store and to avoid a warning that the app is broken when downloaded from the browser. Notarization is required with a Developer ID Application certificate. Windows needs it for the Microsoft Store and to avoid SmartScreen. Linux does not require it. https://v2.tauri.app/distribute/ https://v2.tauri.app/distribute/sign/macos/ https://v2.tauri.app/distribute/sign/windows/ https://v2.tauri.app/distribute/sign/linux/
- Windows 11 Smart App Control blocks unknown unsigned apps even when not downloaded, and users can only disable the feature entirely. https://v2.tauri.app/distribute/sign/windows/

## macOS certificates

- `Developer ID Application` to ship outside the App Store, `Apple Distribution` for the App Store. https://v2.tauri.app/distribute/sign/macos/
- Only the Account Holder can create Developer ID Application certificates. A free account cannot notarize. https://v2.tauri.app/distribute/sign/macos/
- A certificate not tied to your Apple ID is invalid and does not show in `security find-identity`. Download it into the "login" keychain. https://v2.tauri.app/distribute/sign/macos/

## macOS signing identity, local and CI

- Find the identity with `security find-identity -v -p codesigning`, then set `bundle > macOS > signingIdentity` or `APPLE_SIGNING_IDENTITY`. https://v2.tauri.app/distribute/sign/macos/
- CI: export the certificate as `.p12`, base64 it (`openssl base64 -A -in certificate.p12 -out certificate-base64.txt`), store it in `APPLE_CERTIFICATE` and the export password in `APPLE_CERTIFICATE_PASSWORD`, then import into a temporary keychain: https://v2.tauri.app/distribute/sign/macos/

```sh
echo $APPLE_CERTIFICATE | base64 --decode > certificate.p12
security create-keychain -p "$KEYCHAIN_PASSWORD" build.keychain
security default-keychain -s build.keychain
security unlock-keychain -p "$KEYCHAIN_PASSWORD" build.keychain
security set-keychain-settings -t 3600 -u build.keychain
security import certificate.p12 -k build.keychain -P "$APPLE_CERTIFICATE_PASSWORD" -T /usr/bin/codesign
security set-key-partition-list -S apple-tool:,apple:,codesign: -s -k "$KEYCHAIN_PASSWORD" build.keychain
```

https://v2.tauri.app/distribute/sign/macos/

## macOS notarization

- Required with a Developer ID Application certificate. Provide one credential set, then rebuild (for example `tauri build --bundles dmg`): https://v2.tauri.app/distribute/sign/macos/
  - App Store Connect API key (Developer access): `APPLE_API_ISSUER`, `APPLE_API_KEY` (Key ID), `APPLE_API_KEY_PATH` (downloaded private key).
  - Apple ID: `APPLE_ID`, `APPLE_PASSWORD` (app-specific password), `APPLE_TEAM_ID`.
- `--skip-stapling` skips stapling on a first notarization pass. https://v2.tauri.app/distribute/sign/macos/

## Ad-hoc signing

- Without an Apple identity, set `"signingIdentity": "-"`. Apple Silicon requires code signing for all apps from the Internet, and the GitHub pipeline page says ad-hoc signing can avoid macOS treating Apple Silicon builds downloaded from GitHub releases as damaged. https://v2.tauri.app/distribute/sign/macos/ https://v2.tauri.app/distribute/pipelines/github/
- Users still have to allow an ad-hoc app in Privacy & Security settings. https://v2.tauri.app/distribute/sign/macos/

## Windows certificates and SmartScreen

- Use a code signing certificate; TLS/SSL certificates do not work. https://v2.tauri.app/distribute/sign/windows/
- EV no longer gives instant SmartScreen reputation: EV and OV build reputation the same way, so new releases may still warn. Sign every release with the same certificate to keep reputation. https://v2.tauri.app/distribute/sign/windows/

## Windows local certificate

- The thumbprint flow below only applies to OV certificates acquired before June 1st 2023. For EV and current OV certificates, follow the issuer's docs and use `signCommand`. https://v2.tauri.app/distribute/sign/windows/
- Convert to `.pfx` (`openssl pkcs12 -export -in cert.cer -inkey private-key.key -out certificate.pfx`), import with `Import-PfxCertificate -CertStoreLocation Cert:\CurrentUser\My`, read the thumbprint in `certmgr.msc`. https://v2.tauri.app/distribute/sign/windows/

```json
{ "bundle": { "windows": {
  "certificateThumbprint": "A1B1A2B2A3B3A4B4A5B5A6B6A7B7A8B8A9B9A0B0",
  "digestAlgorithm": "sha256",
  "timestampUrl": "http://timestamp.comodoca.com"
} } }
```

https://v2.tauri.app/distribute/sign/windows/

- Success prints `Successfully signed: <path>`. In CI, store the base64 pfx (`certutil -encode`) in `WINDOWS_CERTIFICATE` with `WINDOWS_CERTIFICATE_PASSWORD`, decode and import it before the build step. https://v2.tauri.app/distribute/sign/windows/

## Windows signCommand (Azure and others)

- `bundle > windows > signCommand` runs any signing tool; `%1` is the file. It is mandatory when building Windows installers on Linux or macOS. https://v2.tauri.app/distribute/sign/windows/
- Azure Artifact Signing (formerly Trusted Signing): `cargo install artifact-signing-cli`, set `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_CLIENT_SECRET`: https://v2.tauri.app/distribute/sign/windows/

```json
{ "bundle": { "windows": {
  "signCommand": "artifact-signing-cli -e https://wus2.codesigning.azure.net -a MyAccount -c MyProfile -d MyApp %1"
} } }
```

https://v2.tauri.app/distribute/sign/windows/

- `-d` sets the description; for an `.msi` it is the name in the UAC prompt (random if unset). https://v2.tauri.app/distribute/sign/windows/
- Azure Key Vault: use `relic` with a `src-tauri/relic.conf` and `"signCommand": "relic sign --file %1 --key azure --config relic.conf"`. The Entra app needs the "Key Vault Certificate User" and "Key Vault Crypto User" roles. https://v2.tauri.app/distribute/sign/windows/

## Old patterns

- The page tells you to edit the `tauri` -> `bundle` -> `windows` section, a Tauri 1 layout: its own `signCommand` reference uses `bundle > windows` at the top level. https://v2.tauri.app/distribute/sign/windows/
- The sample workflow on the Windows signing page uses outdated actions and `webkit2gtk-4.0`: base workflows on the GitHub pipeline page instead. https://v2.tauri.app/distribute/sign/windows/ https://v2.tauri.app/distribute/pipelines/github/
