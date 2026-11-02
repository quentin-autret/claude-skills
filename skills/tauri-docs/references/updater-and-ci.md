# Updater plugin and release CI (tauri-action)

## Contents

- Keys and build env
- Updater config
- Update manifests
- Checking and installing
- Runtime options and channels
- GitHub Actions release workflow
- Old patterns

## Keys and build env

- Signature checks are mandatory and cannot be disabled. Generate keys with `tauri signer generate -w ~/.tauri/myapp.key`. https://v2.tauri.app/plugin/updater/
- The public key goes in the config. Never share the private key, and back it up: if it is lost, installed apps can never be updated again. https://v2.tauri.app/plugin/updater/
- At build time set `TAURI_SIGNING_PRIVATE_KEY` (path or content) and optionally `TAURI_SIGNING_PRIVATE_KEY_PASSWORD` as real environment variables: `.env` files do NOT work. https://v2.tauri.app/plugin/updater/

## Updater config

```json
{
  "bundle": { "createUpdaterArtifacts": true },
  "plugins": { "updater": {
    "pubkey": "CONTENT FROM PUBLICKEY.PEM",
    "endpoints": [
      "https://releases.myapp.com/{{target}}/{{arch}}/{{current_version}}",
      "https://github.com/user/repo/releases/latest/download/latest.json"
    ]
  } }
}
```

https://v2.tauri.app/plugin/updater/

- `pubkey` is the key content, not a file path. https://v2.tauri.app/plugin/updater/
- `createUpdaterArtifacts`: `true`, or `"v1Compatible"` while users still run a Tauri 1 build; switch to `true` once all are on v2 (the option goes away in v3). https://v2.tauri.app/plugin/updater/
- Artifacts with `true`: Linux AppImage + `.sig`, macOS `.app.tar.gz` + `.sig`, Windows NSIS `-setup.exe` and `.msi`, each + `.sig`. https://v2.tauri.app/plugin/updater/
- Endpoints need TLS in production. The next URL is tried only on a non-2XX status. Variables: `{{current_version}}`, `{{target}}` (`linux`, `windows`, `darwin`), `{{arch}}` (`x86_64`, `i686`, `aarch64`, `armv7`). https://v2.tauri.app/plugin/updater/
- Windows `plugins.updater.windows.installMode`: `"passive"` by default (small progress window). `"quiet"` cannot ask for admin rights, so it only suits per-user installs. https://v2.tauri.app/plugin/updater/

## Update manifests

- Static JSON: `version` (SemVer, `v` prefix allowed), `platforms.<os>-<arch>.url` and `.signature`; optional `notes`, `pub_date` (RFC 3339). `signature` is the `.sig` file CONTENT, not a URL or path. https://v2.tauri.app/plugin/updater/
- Pitfall: the whole file is validated before `version` is compared, so one incomplete platform entry breaks updates for every platform. https://v2.tauri.app/plugin/updater/
- tauri-action generates this file for GitHub Releases. https://v2.tauri.app/plugin/updater/
- Dynamic server: `204 No Content` when there is no update; `200 OK` with `version`, `url`, `signature` when there is. https://v2.tauri.app/plugin/updater/

## Checking and installing

```js
import { check } from '@tauri-apps/plugin-updater';
import { relaunch } from '@tauri-apps/plugin-process';
const update = await check();
if (update) {
  await update.downloadAndInstall((event) => { /* Started | Progress | Finished */ });
  await relaunch();
}
```

https://v2.tauri.app/plugin/updater/

- Grant `"updater:default"` in a capability under `src-tauri/capabilities/`, and `"process:default"` for `relaunch()`: plugin commands are blocked until then. https://v2.tauri.app/plugin/updater/ https://v2.tauri.app/plugin/process/
- Rust: `app.updater()?.check().await?`, then `update.download_and_install(on_chunk, on_finish).await?` and `app.restart()`, spawned with `tauri::async_runtime::spawn` from `setup`. https://v2.tauri.app/plugin/updater/
- Relaunching right away is optional: you can wait for the user. On Windows the app exits automatically when the installer runs; hook `updater_builder().on_before_exit(...)` to clean up. https://v2.tauri.app/plugin/updater/
- Progress to the UI: wrap fetch and install in commands that take a `Channel`, keeping the pending `Update` in managed state. https://v2.tauri.app/plugin/updater/
- Request options: JS `check({ proxy, timeout, headers })`; Rust `updater_builder().timeout(..).proxy(..).header(k, v)`. https://v2.tauri.app/plugin/updater/

## Runtime options and channels

- For security some runtime APIs are Rust-only: `updater_builder().endpoints(vec![url])?` (release channels) and `.pubkey(...)` (key rotation). A custom target also works in JS (`check({ target: 'macos-universal' })`) and in Rust (`.target(..)`); it is matched against the manifest `platforms` key. https://v2.tauri.app/plugin/updater/
- Inside `format!()` escape the braces twice: `{{{{target}}}}`. https://v2.tauri.app/plugin/updater/
- Default only installs a higher version. Allow downgrades or rollbacks with `version_comparator(|current, update| update.version != current)`. https://v2.tauri.app/plugin/updater/

## GitHub Actions release workflow

- `tauri-apps/tauri-action` builds, uploads and drafts a GitHub release; keep it as the LAST step. Set `projectPath` when the app is not at the repo root. https://v2.tauri.app/distribute/pipelines/github/
- Matrix: `macos-latest` twice (`--target aarch64-apple-darwin`, `--target x86_64-apple-darwin`), `ubuntu-22.04`, `windows-latest`, and `ubuntu-22.04-arm` in public repos. Add macOS Rust targets only on macOS runners. https://v2.tauri.app/distribute/pipelines/github/
- Linux runner deps: `libwebkit2gtk-4.1-dev libappindicator3-dev librsvg2-dev patchelf xdg-utils`. https://v2.tauri.app/distribute/pipelines/github/

```yaml
- uses: tauri-apps/tauri-action@v1
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
  with:
    tagName: app-v__VERSION__
    releaseName: 'App v__VERSION__'
    releaseDraft: true
    prerelease: false
    args: ${{ matrix.args }}
```

https://v2.tauri.app/distribute/pipelines/github/

- The workflow needs `permissions: contents: write`. "Resource not accessible by integration" means the token is read-only: enable "Read and write permissions" under Settings > Actions > Workflow permissions. https://v2.tauri.app/distribute/pipelines/github/
- Build the frontend in the workflow only when `beforeBuildCommand` is not set. https://v2.tauri.app/distribute/pipelines/github/
- macOS signing secrets: `APPLE_CERTIFICATE` (base64 `.p12`), `APPLE_CERTIFICATE_PASSWORD`, `APPLE_SIGNING_IDENTITY`, plus `APPLE_ID`, `APPLE_PASSWORD` (app-specific password) and `APPLE_TEAM_ID` for notarization. Without an Apple certificate, use ad-hoc signing (`signingIdentity: "-"`). https://v2.tauri.app/distribute/sign/macos/ https://v2.tauri.app/distribute/pipelines/github/
- Emulated ARM (`pguyot/arm-runner-action`) takes about an hour uncached: prefer native ARM runners. https://v2.tauri.app/distribute/pipelines/github/
- CrabNebula Cloud is an official partner offering a CDN and update server, with the `crabnebula-dev/cloud-release` action. https://v2.tauri.app/distribute/crabnebula-cloud/

## Old patterns

- v1 updater: `tauri > updater` with `active` and `dialog`, and a built-in prompt. Now: `plugins > updater`, no dialog, and your code must call `check()`. https://v2.tauri.app/start/migrate/from-tauri-1/
- `TAURI_PRIVATE_KEY` / `TAURI_KEY_PASSWORD` are now `TAURI_SIGNING_PRIVATE_KEY` / `TAURI_SIGNING_PRIVATE_KEY_PASSWORD`. https://v2.tauri.app/start/migrate/from-tauri-1/
- v1Compatible artifacts (`.AppImage.tar.gz`, `.nsis.zip`, `.msi.zip`) exist only for migration. https://v2.tauri.app/plugin/updater/
