# Security: capabilities, permissions, scopes, runtime authority

## Contents

- Trust boundary
- Capabilities
- Permissions and permission sets
- Scopes
- Using plugin permissions
- Runtime authority
- What capabilities do not protect

## Trust boundary

- Rust code, plugins included, has full system access. Webview code only reaches what the IPC layer exposes: capabilities gate access and each command enforces finer limits. Strongly type and check all data crossing that line. https://v2.tauri.app/security/
- Let the Core own global state and secrets, keep business logic in Rust, never handle secrets in the frontend. https://v2.tauri.app/concept/process-model/
- Frontend assets can be extracted from the binary fairly easily: do not rely on obscurity. https://v2.tauri.app/security/future/

## Capabilities

- A capability grants or denies permissions to windows or webviews. Files are JSON or TOML in `src-tauri/capabilities/`. https://v2.tauri.app/security/capabilities/
- Default: every file in `capabilities/` is enabled. Once `app.security.capabilities` lists identifiers in `tauri.conf.json`, only those are used. Default to separate files referenced by identifier. https://v2.tauri.app/security/capabilities/
- A window in several capabilities gets the union of all of them. https://v2.tauri.app/security/capabilities/
- Capabilities match window labels, not titles. Expose window creation only to higher-privileged windows. https://v2.tauri.app/security/capabilities/
- Remote pages cannot reach the API unless a capability lists them in `remote.urls` (with `remote-schema.json`). On Linux and Android an embedded `<iframe>` cannot be told apart from its window, so grant remote access very carefully. https://v2.tauri.app/security/capabilities/
- `platforms` narrows a capability (`linux`, `macOS`, `windows`, `iOS`, `android`); default is all. Put desktop-only permissions (`global-shortcut:allow-register`) and mobile-only ones (`nfc:allow-scan`, `biometric:allow-authenticate`) in separate files. https://v2.tauri.app/security/capabilities/
- `tauri-build` generates schemas in `gen/schemas`: point `$schema` at `desktop-schema.json`, `mobile-schema.json` or `remote-schema.json`. https://v2.tauri.app/security/capabilities/

```json
{
  "$schema": "../gen/schemas/desktop-schema.json",
  "identifier": "main-capability",
  "description": "Capability for the main window",
  "windows": ["main"],
  "permissions": [
    "core:path:default",
    "core:event:default",
    "core:window:default",
    "core:app:default",
    "core:resources:default",
    "core:menu:default",
    "core:tray:default",
    "core:window:allow-set-title"
  ]
}
```

https://v2.tauri.app/security/capabilities/ https://v2.tauri.app/start/migrate/from-tauri-2-beta/

- Surprising default: every app command registered in `invoke_handler` is allowed for every window. To change that, declare them with `AppManifest::commands` in `build.rs`: https://v2.tauri.app/security/capabilities/

```rust
fn main() {
    tauri_build::try_build(
        tauri_build::Attributes::new()
            .app_manifest(tauri_build::AppManifest::new().commands(&["your_command"])),
    )
    .unwrap();
}
```

https://v2.tauri.app/security/capabilities/

- Mobile multi-window: iOS scene windows get labels such as `main-1`, so list `"windows": ["main", "main-*"]`. https://v2.tauri.app/learn/mobile-multiwindow/

## Permissions and permission sets

- A permission allows or denies commands, defines scopes, or both. It has no effect until a capability references it. https://v2.tauri.app/security/permissions/
- Identifiers: `<plugin>:default` and `<plugin>:<permission>` (for example `fs:allow-mkdir`), where `<plugin>` is the crate name without `tauri-plugin-`. App permissions use no prefix. Identifiers are limited to ASCII lowercase letters (maximum length 116); the page's own examples also use hyphens. https://v2.tauri.app/security/permissions/
- Capabilities may be JSON, JSON5 or TOML, but app permissions are defined only in TOML: `src-tauri/permissions/<identifier>.toml`. https://v2.tauri.app/security/permissions/

```toml
[[permission]]
identifier = "my-identifier"
description = "This describes the impact and more."
commands.allow = ["read_file"]

[[permission.scope.allow]]
my-scope = "$HOME/*"

[[permission.scope.deny]]
my-scope = "$HOME/secret"

[[set]]
identifier = "allow-home-read-extended"
description = "This allows non-recursive read access to files and to create directories in the `$HOME` folder."
permissions = ["fs:read-files", "fs:scope-home", "fs:allow-mkdir"]
```

https://v2.tauri.app/security/permissions/

- A plugin's `default` permission is added to the app automatically only when the plugin is added with the Tauri CLI (`tauri add`). https://v2.tauri.app/security/permissions/

## Scopes

- A scope narrows what an allowed command may do. `deny` always wins over `allow`. https://v2.tauri.app/security/scope/
- The scope is injected into the command, and the command must enforce it. Command authors must audit scope checks so no bypass exists. https://v2.tauri.app/security/scope/
- A scope permission applies to all commands of a plugin (global scope), or only to the commands it is combined with in a set. https://v2.tauri.app/security/scope/
- fs ships platform denies, `deny-webview-data-linux` (`$APPLOCALDATA/**`) and `deny-webview-data-windows` (`$APPLOCALDATA/EBWebView/**`), because webview data lives there. A permission can carry `platforms = ["linux"]`. https://v2.tauri.app/security/scope/

## Using plugin permissions

- Plugin commands that touch anything sensitive are blocked until a capability allows them. https://v2.tauri.app/plugin/autostart/
- Pick one command permission plus the narrowest scope (a single file rather than all of `$HOME`). Scoped entries use the object form: https://v2.tauri.app/learn/security/using-plugin-permissions/

```json
{ "identifier": "fs:allow-write-text-file", "allow": [{ "path": "$HOME/test.txt" }] }
```

https://v2.tauri.app/learn/security/using-plugin-permissions/

- A command permission without a scope still fails: `fs:allow-exists` alone gives `forbidden path` at runtime. https://v2.tauri.app/plugin/file-system/
- An error like `fs.write_text_file not allowed. Permissions associated with this command: ...` means the permission or scope is missing. https://v2.tauri.app/learn/security/using-plugin-permissions/
- For a community plugin, read its `permissions/default.toml` to see what `default` grants. Available permissions are in the plugin docs table or `permissions/autogenerated`. https://v2.tauri.app/learn/security/using-plugin-permissions/
- `fs:default` is `["read-all", "scope-app-recursive", "deny-default"]`: all reads inside `$APP` (which the app must create first), webview data denied. https://v2.tauri.app/learn/security/using-plugin-permissions/
- Listing only the commands you use, instead of `default` sets, also lets `build.removeUnusedCommands` shrink the binary. https://v2.tauri.app/concept/size/

## Runtime authority

On each invoke, the runtime authority checks that the origin may use the command and that a capability allows it, injects the matching scopes, and only then calls the command. A denied request never reaches the command. https://v2.tauri.app/security/runtime-authority/

## What capabilities do not protect

They limit a compromised frontend, accidental exposure of local interfaces and frontend-to-backend escalation. They do not protect against malicious or buggy Rust code, scopes that are too lax, wrong scope checks in commands, intentional bypasses in Rust, webview 0-days, supply chain attacks or a compromised dev machine. https://v2.tauri.app/security/capabilities/
