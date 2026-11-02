# Deprecations

## Contents

- Kinds and flags
- Deprecations (v24 to v26)
- Security-relevant entries
- Removed with replacement
- Revoked
- Old patterns

## Kinds and flags

- 4 kinds: documentation-only, application (warns for code outside `node_modules`), runtime (warns for all code) and End-of-Life (removed or about to be). https://nodejs.org/docs/latest/api/deprecations.html
- Run CI with `--pending-deprecation` to make documentation-only entries that support it warn and to cover `node_modules`. `--throw-deprecation` turns runtime warnings into errors. https://nodejs.org/docs/latest/api/deprecations.html
- Many entries ship a codemod: `npx codemod@latest @nodejs/<recipe>`, for example `@nodejs/node-url-to-whatwg-url`, `@nodejs/rmdir`, `@nodejs/util-is`, `@nodejs/crypto-fips-to-getFips`. https://nodejs.org/docs/latest/api/deprecations.html#DEP0147

## Deprecations (v24 to v26)

Each code links as `https://nodejs.org/docs/latest/api/deprecations.html#DEPxxxx`.

| Code | What | Use instead | Since |
| :-- | :-- | :-- | :-- |
| DEP0205 | `module.register()` | `module.registerHooks()` | runtime, v26.0.0 |
| DEP0203 | `CryptoKey` passed to `node:crypto` functions | none named (will throw later) | runtime, v26.0.0 |
| DEP0204 | `KeyObject.from()` with a non-extractable `CryptoKey` | none named (will throw later) | runtime, v26.0.0 |
| DEP0201 | `Duplex.toWeb({ type })` | `readableType` | runtime, v26.0.0 |
| DEP0206 | second `hmac.digest()` call | call it once | doc-only, v26.2.0 |
| DEP0209 | `AbortSignal` to dispose servers, streams, children | `using` / `await using` | doc-only, v26.8.2 |
| DEP0210, DEP0211 | `DatabaseSync`, `StatementSync` | `Database`, `Statement` | doc-only, v26.11.0 |
| DEP0202 | HTTP/2 `Http1IncomingMessage` options | `http1Options` | doc-only, v25.7.0 |
| DEP0198 | SHAKE digests without `outputLength` | pass `outputLength` | runtime, v25.0.0 |
| DEP0200 | `fs.Dir` closed by GC | `dir.close()` or `await using` | doc-only, v24.9.0 |
| DEP0199 | `node:_http_*` internals | `node:http` | doc-only, v24.6.0 |
| DEP0190 | `args` array with `shell: true` | none named (values are not escaped) | runtime, v24.0.0 |
| DEP0187 | `fs.existsSync()` with invalid types | validate the argument | runtime, v24.0.0 |
| DEP0169 | `url.parse()` | WHATWG `URL` | application, v24.0.0 |
| DEP0184 | zlib classes without `new` | `new` | runtime, v24.0.0 |

Sources: https://nodejs.org/docs/latest/api/deprecations.html#DEP0205 , https://nodejs.org/docs/latest/api/deprecations.html#DEP0203 , https://nodejs.org/docs/latest/api/deprecations.html#DEP0209 , https://nodejs.org/docs/latest/api/deprecations.html#DEP0210 , https://nodejs.org/docs/latest/api/deprecations.html#DEP0190 , https://nodejs.org/docs/latest/api/deprecations.html#DEP0169

- DEP0209 example: https://nodejs.org/docs/latest/api/deprecations.html#DEP0209

```js
// Deprecated
const ac = new AbortController();
server.listen({ port: 3000, signal: ac.signal });
// Use this instead
await using server = http.createServer(handler);
server.listen(3000);
```

Keep `AbortSignal` for cancelling actions, propagating cancellation and timeouts.

## Security-relevant entries

- DEP0190: with `shell: true`, `args` are only joined with spaces, not escaped: shell injection. https://nodejs.org/docs/latest/api/deprecations.html#DEP0190
- DEP0169: `url.parse()` is error-prone and no CVEs are issued for it. `url.format(string)` and `url.resolve()` use it too. https://nodejs.org/docs/latest/api/deprecations.html#DEP0169
- DEP0005: `Buffer()` has usability issues that lead to accidental security issues. Use `Buffer.alloc()`, `Buffer.allocUnsafe()` or `Buffer.from()`. https://nodejs.org/docs/latest/api/deprecations.html#DEP0005
- DEP0139: `process.umask()` with no argument races between threads. https://nodejs.org/docs/latest/api/deprecations.html#DEP0139
- DEP0167: DH groups `modp1`, `modp2`, `modp5` are weak. https://nodejs.org/docs/latest/api/deprecations.html#DEP0167

## Removed with replacement

| Removed | Replacement | Source |
| :-- | :-- | :-- |
| `crypto.createCipher()` | KDF with random salt, then `createCipheriv()` | https://nodejs.org/docs/latest/api/deprecations.html#DEP0106 |
| GCM tags under 16 bytes without `authTagLength` | pass `authTagLength` | https://nodejs.org/docs/latest/api/deprecations.html#DEP0182 |
| `pbkdf2` without digest | pass the digest | https://nodejs.org/docs/latest/api/deprecations.html#DEP0009 |
| `fs.rmdir(p, { recursive: true })` | `fs.rm(p, { recursive: true, force: true })` | https://nodejs.org/docs/latest/api/deprecations.html#DEP0147 |
| `fs.F_OK` and friends | `fs.constants.*` | https://nodejs.org/docs/latest/api/deprecations.html#DEP0176 |
| `dirent.path` | `dirent.parentPath` | https://nodejs.org/docs/latest/api/deprecations.html#DEP0178 |
| `FileHandle` closed by GC | close it explicitly | https://nodejs.org/docs/latest/api/deprecations.html#DEP0137 |
| thenables in stream methods | callbacks | https://nodejs.org/docs/latest/api/deprecations.html#DEP0157 |
| non-integer `process.exit()` codes | integers | https://nodejs.org/docs/latest/api/deprecations.html#DEP0164 |
| unhandled rejections that only warn | the process exits | https://nodejs.org/docs/latest/api/deprecations.html#DEP0018 |
| `assert.CallTracker` | none | https://nodejs.org/docs/latest/api/deprecations.html#DEP0173 |
| `Module.createRequireFromPath()` | `module.createRequire()` | https://nodejs.org/docs/latest/api/deprecations.html#DEP0130 |

## Revoked

- DEP0033 `events.listenerCount(emitter, name)` (v25.4.0), DEP0163 `channel.subscribe()` (v24.8.0), DEP0089 `require('node:assert')`, DEP0116 legacy URL API (folded into DEP0169). Identifiers are never renumbered. https://nodejs.org/docs/latest/api/deprecations.html#DEP0033

## Old patterns

| Old pattern | Replacement | Source |
| :-- | :-- | :-- |
| `require('node:sys')` | `node:util` | https://nodejs.org/docs/latest/api/deprecations.html#DEP0025 |
| `require('node:constants')` | the `constants` property of the relevant module, e.g. `fs.constants`, `os.constants` | https://nodejs.org/docs/latest/api/deprecations.html#DEP0008 |
| `require('node:punycode')` | a userland package | https://nodejs.org/docs/latest/api/deprecations.html#DEP0040 |
| `http.ClientRequest()`, `ServerResponse()` etc. without `new` | `new` | https://nodejs.org/docs/latest/api/deprecations.html#DEP0195 |
| `process.binding()` | public APIs | https://nodejs.org/docs/latest/api/deprecations.html#DEP0111 |
