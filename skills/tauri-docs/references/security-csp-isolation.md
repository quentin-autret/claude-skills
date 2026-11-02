# Security: CSP, isolation pattern, asset protocol, headers, lifecycle

## Contents

- Content Security Policy
- Isolation pattern
- Asset protocol scope
- HTTP headers
- Lifecycle threats

## Content Security Policy

- CSP is only enforced when set in the Tauri config (`app.security.csp`). Make it as strict as possible and allow only hosts you trust. https://v2.tauri.app/security/csp/
- At compile time Tauri adds hashes for local scripts and nonces for styles and external scripts: configure only what is specific to your app. https://v2.tauri.app/security/csp/
- Avoid remote scripts such as CDNs: every untrusted file is an attack vector. https://v2.tauri.app/security/csp/
- Rust/WASM frontends need `'wasm-unsafe-eval'` in `script-src`. https://v2.tauri.app/security/csp/
- Example from Tauri's API demo, to tailor per app (note `ipc:` and `asset:` sources): https://v2.tauri.app/security/csp/

```json
"csp": {
  "default-src": "'self' customprotocol: asset:",
  "connect-src": "ipc: http://ipc.localhost",
  "font-src": ["https://fonts.gstatic.com"],
  "img-src": "'self' asset: http://asset.localhost blob: data:",
  "style-src": "'unsafe-inline' 'self' https://fonts.googleapis.com"
}
```

https://v2.tauri.app/security/csp/

## Isolation pattern

- Default pattern is `brownfield` (no interception). Isolation routes all IPC, events included, through a sandboxed `<iframe>` app that can check or rewrite each message before the Core sees it. Tauri highly recommends it whenever possible. https://v2.tauri.app/concept/inter-process-communication/brownfield/ https://v2.tauri.app/concept/inter-process-communication/isolation/
- Use the hook to validate inputs: file paths stay in expected places, HTTP calls only set the expected Origin. It mainly guards against compromised build tooling and deep dependency trees. https://v2.tauri.app/concept/inter-process-communication/isolation/
- Messages are encrypted with AES-GCM using a key regenerated at each start. In headless setups (WebDriver CI) low entropy can stall key generation: install an entropy daemon such as `haveged`. https://v2.tauri.app/concept/inter-process-communication/isolation/
- Pitfall: on Windows, sandboxed iframes cannot load external files, so Tauri inlines scripts at build time. A classic `<script src="index.js">` works, ES modules do not load. https://v2.tauri.app/concept/inter-process-communication/isolation/
- Keep the isolation app tiny, with minimal dependencies and build steps. https://v2.tauri.app/concept/inter-process-communication/isolation/

```json
{ "app": { "security": { "pattern": {
  "use": "isolation", "options": { "dir": "../dist-isolation" } } } } }
```

```javascript
// ../dist-isolation/index.js
window.__TAURI_ISOLATION_HOOK__ = (payload) => {
  console.log('hook', payload); // validate or modify here
  return payload;
};
```

https://v2.tauri.app/concept/inter-process-communication/isolation/

## Asset protocol scope

- The asset protocol (used by `convertFileSrc`) serves disk files to the webview. It needs both `enable: true` and a `scope`, otherwise loads fail with "asset protocol not configured to allow the path". https://v2.tauri.app/security/asset-protocol/

```json
{ "app": { "security": { "assetProtocol": {
  "enable": true,
  "scope": { "allow": ["$APPCACHE/**/*"], "deny": ["$APPCACHE/**/secrets/**"] }
} } } }
```

https://v2.tauri.app/security/asset-protocol/

- Resolved paths are absolute, so `["*/**"]` typically does not match on Linux. Start patterns with a base-directory variable (`$HOME`, `$CACHE`, `$APPCACHE`, `$APPDATA`, `$RESOURCE`) or a leading `/`. `deny` wins over `allow`. Prefer `**/*` over a bare `**`. https://v2.tauri.app/security/asset-protocol/
- Unix default `requireLiteralLeadingDot: true`: wildcards skip dotfolders, so `$HOME/**` does not match `~/.cache/...`, while `$HOME/.cache/myapp/**` does. Only the object form of `scope` can set it to `false`, which widens exposure. https://v2.tauri.app/security/asset-protocol/
- Prefer narrow roots (`$APPCACHE`, `$RESOURCE`, one app folder) over `$HOME/**/*`. https://v2.tauri.app/security/asset-protocol/
- Paths picked by the user at runtime are not in the static scope. Persist them with `tauri-plugin-persisted-scope` and its `protocol-asset` feature, registered after the fs plugin. https://v2.tauri.app/security/asset-protocol/

```toml
tauri-plugin-persisted-scope = { version = "2", features = ["protocol-asset"] }
```

https://v2.tauri.app/security/asset-protocol/

## HTTP headers

- `app.security.headers` (since 2.1.0) adds headers to webview responses, not to IPC or error responses. CSP does not go here. https://v2.tauri.app/security/http-headers/
- Only a fixed list is accepted: `Access-Control-Allow-Credentials`, `-Allow-Headers`, `-Allow-Methods`, `-Expose-Headers`, `-Max-Age`, `Cross-Origin-Embedder-Policy`, `Cross-Origin-Opener-Policy`, `Cross-Origin-Resource-Policy`, `Permissions-Policy`, `Service-Worker-Allowed`, `Timing-Allow-Origin`, `X-Content-Type-Options`, `Tauri-Custom-Header` (not for production). https://v2.tauri.app/security/http-headers/
- Values: string as is, array joined by `, `, object joined as `key value` pairs separated by `; `, `null` ignored. https://v2.tauri.app/security/http-headers/
- COOP `same-origin` plus COEP `require-corp` enable `SharedArrayBuffer`. https://v2.tauri.app/security/http-headers/

```json
"headers": {
  "Cross-Origin-Opener-Policy": "same-origin",
  "Cross-Origin-Embedder-Policy": "require-corp"
}
```

https://v2.tauri.app/security/http-headers/

- Pitfall: Tauri cannot inject headers into the dev server, and frameworks drop them at build. Define them twice: in the dev server (Vite `server.headers`, Nuxt `vite.server.headers`, Next `headers()`, Angular serve `headers`, Trunk `[serve] headers`) and in the Tauri config. https://v2.tauri.app/security/http-headers/

## Lifecycle threats

- Keep Tauri, `rustc` and Node updated. Audit dependencies with `npm audit` and `cargo audit` (`cargo-vet`, `cargo crev` and cargo supply chain help). For critical dependencies, pin git revisions by hash, or at least by tag. https://v2.tauri.app/security/lifecycle/
- Dev servers, Tauri's included, are unauthenticated and unencrypted and expose the frontend to the local network, so an attacker there can push code to dev devices. Develop on trusted networks only. https://v2.tauri.app/security/lifecycle/
- CI holds source and secrets: pin third-party action versions and sign binaries for each platform. Keys on hardware tokens cannot leak, but a compromised CI can still sign malicious builds. https://v2.tauri.app/security/lifecycle/
- Do not count on reproducible builds: Rust and many frontend bundlers do not guarantee them. https://v2.tauri.app/security/lifecycle/
- Whoever controls the update manifest server, build server or binary host controls updates. https://v2.tauri.app/security/lifecycle/
- `cargo-auditable` embeds an SBOM of exact crate versions in the binary. https://v2.tauri.app/security/future/
- The webview is treated as insecure: CSP limits its traffic and capabilities keep untrusted content away from the API. https://v2.tauri.app/security/lifecycle/
- Report Tauri vulnerabilities privately (GitHub disclosure or security@tauri.app), never in public issues. https://v2.tauri.app/security/
