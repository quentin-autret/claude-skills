# Distributing on Linux: deb, rpm, AppImage, Flatpak, Snap, AUR, ARM

## Contents

- Build and bundle flow
- glibc baseline
- Debian package
- RPM
- AppImage
- Flatpak
- Snap
- AUR
- ARM builds

## Build and bundle flow

- `tauri build` bundles every configured format. To customize, split it: `tauri build --no-bundle`, then `tauri bundle --bundles <formats>`. https://v2.tauri.app/distribute/
- GUI apps on Linux and macOS do not inherit `$PATH` from shell dotfiles; the `fix-path-env-rs` crate fixes that. https://v2.tauri.app/distribute/appimage/

## glibc baseline

- Build on the OLDEST distro you support that still ships `libwebkit2gtk-4.1-dev` (Ubuntu 22.04 and Debian 12 are the given baselines). Building on a newer system raises the minimum glibc and users see `version 'GLIBC_2.33' not found`. Build in Docker or CI. https://v2.tauri.app/distribute/debian/ https://v2.tauri.app/distribute/appimage/

## Debian package

- The stock `.deb` declares `libwebkit2gtk-4.1-0` and `libgtk-3-0`, plus `libappindicator3-1` if the app uses a tray. Add dependencies under `bundle > linux > deb`. https://v2.tauri.app/distribute/debian/
- Extra files map destination to source (relative to `tauri.conf.json`): https://v2.tauri.app/distribute/debian/

```json
{ "bundle": { "linux": { "deb": { "files": {
  "/usr/share/README.md": "../README.md",
  "/usr/share/assets": "../assets/"
} } } } }
```

https://v2.tauri.app/distribute/debian/

## RPM

- Config under `bundle > linux > rpm`: `depends`, `conflicts`, `provides`, `obsoletes`, `files`, `desktopTemplate`, `release`, `epoch`, and `preInstallScript` / `postInstallScript` / `preRemoveScript` / `postRemoveScript` (script paths). https://v2.tauri.app/distribute/rpm/
- Packages in `obsoletes` are removed when yours installs. Avoid `epoch` unless needed: it changes version comparison. https://v2.tauri.app/distribute/rpm/
- License from `Cargo.toml` `license`, or `bundle.license` / `bundle.licenseFile`. https://v2.tauri.app/distribute/rpm/
- Sign by exporting a GPG key before the build: https://v2.tauri.app/distribute/rpm/

```bash
export TAURI_SIGNING_RPM_KEY=$(cat /home/johndoe/my_super_private.key)
export TAURI_SIGNING_RPM_KEY_PASSPHRASE=password
```

https://v2.tauri.app/distribute/rpm/

- Check with `rpm -v --checksig <file>.rpm` after importing the public key and configuring `~/.rpmmacros`. https://v2.tauri.app/distribute/rpm/

## AppImage

- Bundles all dependencies, so size jumps from a few MB to 70+ MB. Prefer distro packages when possible. https://v2.tauri.app/distribute/appimage/
- Media playback: `bundle > linux > appimage > bundleMediaFramework` includes GStreamer. Fully supported only on Ubuntu build hosts, which must have every plugin the app needs. GStreamer `ugly` plugins may not be redistributable. https://v2.tauri.app/distribute/appimage/
- Extra files: `bundle > linux > appimage > files`; destinations must start with `/usr/`. https://v2.tauri.app/distribute/appimage/
- Signing: set `SIGN=1`, optional `SIGN_KEY`, and `APPIMAGETOOL_SIGN_PASSPHRASE` (mandatory in CI or gpg opens a dialog). Signing failures still produce an AppImage unless `APPIMAGETOOL_FORCE_SIGN=1`. https://v2.tauri.app/distribute/sign/linux/
- AppImage never checks its own signature: users run the AppImageUpdate `validate` tool, and you publish the key ID on an authenticated channel. https://v2.tauri.app/distribute/sign/linux/

## Flatpak

- Flow: build the `.deb`, write an AppStream MetaInfo file, then a manifest that unpacks the deb (`ar -x`, `tar -xf data.tar.gz`) and installs the binary, `.desktop`, icons and metainfo into `/app`. https://v2.tauri.app/distribute/flatpak/
- Runtime: `org.gnome.Platform` covers the standard Tauri dependencies (the page shows both 46 and 47: check before use). https://v2.tauri.app/distribute/flatpak/
- `finish-args`: `--socket=wayland` and `--socket=fallback-x11` (both needed to show a window), `--share=ipc`, `--device=dri` when OpenGL is needed. Tray: `--talk-name=org.kde.StatusNotifierWatcher` and `--filesystem=xdg-run/tray-icon:create`, or move the tray temp dir into the cache dir: https://v2.tauri.app/distribute/flatpak/

```rust
TrayIconBuilder::new()
  .icon(app.default_window_icon().unwrap().clone())
  .temp_dir_path(app.path().app_cache_dir().unwrap())
  .build()
  .unwrap();
```

https://v2.tauri.app/distribute/flatpak/

- Rewrite `Icon=` in the desktop file to the Flatpak id and install it as `<identifier>.desktop`. https://v2.tauri.app/distribute/flatpak/
- Do not ship a prebuilt local `.so`: build extra libraries from source inside the Flatpak. https://v2.tauri.app/distribute/flatpak/
- Open-source builds run offline: generate sources with `flatpak-node-generator` and `flatpak-cargo-generator.py` (on `src-tauri/Cargo.lock`). https://v2.tauri.app/distribute/flatpak/
- Test: `flatpak-builder --force-clean --user --disable-cache --repo flatpak-repo flatpak flatpak-builder.yaml`, then `flatpak run <id>`. Submit through a PR to `flathub/flathub` on the `new-pr` branch. https://v2.tauri.app/distribute/flatpak/
- single-instance under Flatpak needs `--talk-name` and `--own-name` for its DBus name. https://v2.tauri.app/plugin/single-instance/

## Snap

- `snapcraft.yaml`: `base: core22`, `confinement: strict`, `extensions: [gnome]`, and a `layout` binding `/usr/lib/$SNAPCRAFT_ARCH_TRIPLET/webkit2gtk-4.1`. The gnome extension supplies the desktop, wayland, x11 and opengl plugs; add others such as `network`. `name` must equal the name registered on snapcraft.io. https://v2.tauri.app/distribute/snapcraft/
- The part builds with `npm run tauri build -- --bundles deb`, unpacks it with `dpkg -x` into `$SNAPCRAFT_PART_INSTALL/` and fixes the `Icon=` line. https://v2.tauri.app/distribute/snapcraft/
- single-instance needs a dbus `slots`/`plugs` pair named `org.<id_with_underscores>.SingleInstance`. https://v2.tauri.app/distribute/snapcraft/
- Publish: `snapcraft login`, `snapcraft upload --release=stable <file>.snap`. https://v2.tauri.app/distribute/snapcraft/

## AUR

- `depends` must include `cairo desktop-file-utils gdk-pixbuf2 glib2 gtk3 hicolor-icon-theme libsoup pango webkit2gtk-4.1`. https://v2.tauri.app/distribute/aur/
- Binary package: point `source_x86_64` / `source_aarch64` at released `.deb` files and extract `data.tar.gz` into `${pkgdir}`, with `options=('!strip' '!debug')`. Refresh icon and desktop caches in a `.install` file. https://v2.tauri.app/distribute/aur/
- Regenerate `.SRCINFO` with `makepkg --printsrcinfo > .SRCINFO` before pushing. https://v2.tauri.app/distribute/aur/

## ARM builds

- AppImage cannot be cross-compiled for ARM (linuxdeploy limitation): build on ARM hardware. GitHub's `ubuntu-22.04-arm` / `ubuntu-24.04-arm` runners (public repositories) work unchanged; QEMU emulation is extremely slow. https://v2.tauri.app/distribute/appimage/ https://v2.tauri.app/distribute/pipelines/github/
- The Debian page documents manual cross-compilation (deb bundle) from a Debian/Ubuntu host: https://v2.tauri.app/distribute/debian/
  1. `rustup target add aarch64-unknown-linux-gnu` and `sudo apt install gcc-aarch64-linux-gnu`.
  2. In `.cargo/config.toml`: `[target.aarch64-unknown-linux-gnu] linker = "aarch64-linux-gnu-gcc"`.
  3. `sudo dpkg --add-architecture arm64`. On Ubuntu, also add `ports.ubuntu.com` sources tagged `[arch=armhf,arm64]` and tag existing lines `[arch=amd64]`, or apt breaks.
  4. `sudo apt install libwebkit2gtk-4.1-dev:arm64`; on missing OpenSSL headers, `libssl-dev:arm64` or `openssl-sys` with `features = ["vendored"]`.
  5. `export PKG_CONFIG_SYSROOT_DIR=/usr/aarch64-linux-gnu/`, then `cargo tauri build --target aarch64-unknown-linux-gnu`.
- ARMv7 uses the same steps with `armv7-unknown-linux-gnueabihf`, `gcc-arm-linux-gnueabihf` and `armhf`. https://v2.tauri.app/distribute/debian/
