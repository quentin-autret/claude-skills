---
name: nodejs-docs
description: Teaches Node.js practice drawn from the official Node.js API documentation, with pitfalls, surprising defaults, deprecations and replacements, each backed by its nodejs.org URL. Use when writing, reviewing, debugging or upgrading Node.js code, or choosing a built-in module (fs, crypto, streams, http, process, CLI flags, node:test, ESM/CommonJS, package.json exports, TypeScript type stripping, child_process, worker_threads, node:sqlite, deprecations).
metadata:
  version: "v26.13.0 (path latest). Features added after the v24 LTS line are flagged in the body"
  source: https://nodejs.org/docs/latest/api/
  fetched: "2026-11-02"
---

# Node.js API (nodejs.org)

Condenses the Node.js API documentation into what a model tends to get wrong: unsafe defaults, pitfalls, deprecations. Each claim ends with its nodejs.org page.

## Version covered

The documentation read is v26.13.0. Many projects run the v24 LTS line, so the references mark what is newer: "added in v25.x" or "added in v26.x" means missing on v24 unless a backport version is given. https://nodejs.org/docs/latest/api/

Before relying on a flagged feature, check `process.version` or feature-detect (`process.features.typescript`, `process.features.require_module`). https://nodejs.org/docs/latest/api/process.html

Read the rules below, then open the matching reference file.

## Highest-value rules

- Never call `access()`, `exists()` or `stat()` before opening a file: open directly and handle `ENOENT` / `EEXIST` (flag `'wx'` for exclusive create). https://nodejs.org/docs/latest/api/fs.html#fsaccesspath-mode-callback
- Close every `FileHandle` explicitly, ideally with `await using`. A `FileHandle` left to GC throws. https://nodejs.org/docs/latest/api/deprecations.html#DEP0137
- Compose streams with `pipeline()`, not `.pipe()` chains, and respect `write()` returning `false`. https://nodejs.org/docs/latest/api/stream.html#readablepipedestination-options
- Add an `'error'` listener to every emitter, stream, socket, request and child process: an unhandled `'error'` crashes the process. https://nodejs.org/docs/latest/api/events.html#error-events
- Branch on `err.code`, never `err.message`. https://nodejs.org/docs/latest/api/errors.html#errorcode
- Set `process.exitCode` instead of calling `process.exit()`, which can truncate stdout. https://nodejs.org/docs/latest/api/process.html#processexitcode
- Never resume work after `'uncaughtException'`: clean up synchronously and exit. https://nodejs.org/docs/latest/api/process.html#warning-using-uncaughtexception-correctly
- Prefer `queueMicrotask()` to the Legacy `process.nextTick()`. https://nodejs.org/docs/latest/api/process.html#when-to-use-queuemicrotask-vs-processnexttick
- Use `execFile()`/`spawn()` without a shell. Passing `args` with `shell: true` does not escape them. https://nodejs.org/docs/latest/api/deprecations.html#DEP0190
- Consume every HTTP client response body (`res.resume()` at least), attach `req.on('error')`, and remember `timeout` does not abort: call `destroy()`. https://nodejs.org/docs/latest/api/http.html#class-httpclientrequest
- Keep `headersTimeout` non-zero on servers without a reverse proxy. https://nodejs.org/docs/latest/api/http.html#serverheaderstimeout
- Use the WHATWG `URL`. `url.parse()` is deprecated and no CVEs are issued for it. https://nodejs.org/docs/latest/api/deprecations.html#DEP0169
- `path.isAbsolute()` does not stop path traversal, and `path.normalize()` is lexical and may change how the system resolves the path: resolve, then check the result stays inside the root. https://nodejs.org/docs/latest/api/path.html#pathisabsolutepath
- Use `Buffer.alloc()` by default, `Buffer.subarray()` for views, and never `new Buffer()`. https://nodejs.org/docs/latest/api/deprecations.html#DEP0005
- Compare secrets with `crypto.timingSafeEqual()` on equal-length buffers. https://nodejs.org/docs/latest/api/crypto.html#cryptotimingsafeequala-b
- Set `maxOutputLength` when decompressing untrusted data with zlib (default `buffer.kMaxLength`). https://nodejs.org/docs/latest/api/zlib.html#class-options
- Set `"type"` in `package.json`, and put `"default"` last in `"exports"` conditions. https://nodejs.org/docs/latest/api/packages.html#determining-module-system
- `require()` of ESM works for synchronous graphs. Top-level `await` anywhere in the graph throws `ERR_REQUIRE_ASYNC_MODULE`. https://nodejs.org/docs/latest/api/modules.html#loading-ecmascript-modules-using-require
- Type stripping runs `.ts` without type checking: no `enum`, explicit `.ts` extensions, `import type`, and no `tsconfig` `paths`. It is stable since v25.2.0 (backported to v24.12.0), detect it with `process.features.typescript`. https://nodejs.org/docs/latest/api/typescript.html#typescript-features , https://nodejs.org/docs/latest/api/typescript.html#type-stripping
- Register module hooks with `module.registerHooks()`. `module.register()` is deprecated in v26. https://nodejs.org/docs/latest/api/deprecations.html#DEP0205
- Prefer `AsyncLocalStorage` to `async_hooks.createHook()`. https://nodejs.org/docs/latest/api/async_hooks.html
- Use `await t.test()` for subtests in `node:test`. https://nodejs.org/docs/latest/api/test.html#subtests
- `node:vm` and WASI are not sandboxes, and the Permission Model is not a safe way to run untrusted code: never use them for that. https://nodejs.org/docs/latest/api/permissions.html#process-based-permissions
- Dispose of servers, streams and child processes with `using` / `await using`, not an `AbortSignal`. Keep signals for cancellation and timeouts. https://nodejs.org/docs/latest/api/deprecations.html#DEP0209
- `dns.lookup()` is `getaddrinfo` on the threadpool, not a DNS query. https://nodejs.org/docs/latest/api/dns.html#implementation-considerations
- Size pools with `os.availableParallelism()`. https://nodejs.org/docs/latest/api/os.html#osavailableparallelism
- `node:sqlite`: bind parameters, never interpolate into SQL. https://nodejs.org/docs/latest/api/sqlite.html#binding-parameters
- Run CI with `--pending-deprecation` to surface deprecations. https://nodejs.org/docs/latest/api/deprecations.html

## Reference files

| File | Read it for |
| :-- | :-- |
| [references/fs.md](references/fs.md) | TOCTOU, `FileHandle`, `readFile`/`writeFile`, `rm`, `glob`, `fs.watch` |
| [references/path-and-url.md](references/path-and-url.md) | `path`, traversal, `file:` URLs, WHATWG `URL`, `querystring` |
| [references/crypto.md](references/crypto.md) | `KeyObject`, `timingSafeEqual`, hashing, pbkdf2, scrypt, weak algorithms |
| [references/ciphers-and-webcrypto.md](references/ciphers-and-webcrypto.md) | AEAD tags, GCM, key wrap, `subtle`, key formats |
| [references/streams.md](references/streams.md) | `pipeline`, backpressure, `highWaterMark`, `finished`, implementing streams |
| [references/web-streams-and-stream-iter.md](references/web-streams-and-stream-iter.md) | Web Streams, `node:stream/iter` |
| [references/buffer.md](references/buffer.md) | `alloc` versus `allocUnsafe`, encodings, Blob |
| [references/zlib.md](references/zlib.md) | decompression bombs, flush, Zstd |
| [references/http-client.md](references/http-client.md) | `http.request`, agents, keep-alive, proxy |
| [references/http-server.md](references/http-server.md) | timeouts, graceful shutdown, HTTPS servers |
| [references/net-dns-dgram.md](references/net-dns-dgram.md) | TCP, `BlockList`, `dns.lookup` versus `resolve`, UDP |
| [references/tls.md](references/tls.md) | SNI, ciphers, CA store, session resumption |
| [references/http2.md](references/http2.md) | secure server, `allowHTTP1`, hardening |
| [references/process.md](references/process.md) | exit codes, signals, `uncaughtException`, rejections |
| [references/cli-and-env-files.md](references/cli-and-env-files.md) | `NODE_OPTIONS`, `--env-file`, `--watch` |
| [references/permissions.md](references/permissions.md) | `--permission`, known gaps |
| [references/errors.md](references/errors.md) | `error.code`, `cause`, error codes |
| [references/events-and-timers.md](references/events-and-timers.md) | `captureRejections`, `events.once`, `EventTarget`, timers |
| [references/async-context-and-diagnostics-channel.md](references/async-context-and-diagnostics-channel.md) | `AsyncLocalStorage`, `diagnostics_channel` |
| [references/util-and-console.md](references/util-and-console.md) | `util`, `Console` |
| [references/os-tty-readline-repl.md](references/os-tty-readline-repl.md) | `os`, `tty`, `readline`, REPL |
| [references/test-runner.md](references/test-runner.md) | subtests, isolation, coverage, reporters, mocks |
| [references/assert-and-bench.md](references/assert-and-bench.md) | `node:assert/strict`, deep equality, `node:bench` |
| [references/esm-and-commonjs.md](references/esm-and-commonjs.md) | `"type"`, JSON imports, `import.meta`, `require(esm)` |
| [references/packages.md](references/packages.md) | `"exports"`, `"imports"`, conditions, dual packages |
| [references/typescript-and-module-hooks.md](references/typescript-and-module-hooks.md) | type stripping, `registerHooks`, `findPackageJSON` |
| [references/globals.md](references/globals.md) | `fetch`, `AbortSignal`, globals |
| [references/child-process.md](references/child-process.md) | `execFile` versus `exec`, `maxBuffer`, IPC |
| [references/worker-threads-and-cluster.md](references/worker-threads-and-cluster.md) | pools, transfer lists, cluster |
| [references/vm-wasi-vfs.md](references/vm-wasi-vfs.md) | vm, WASI, `node:vfs` |
| [references/performance-and-profiling.md](references/performance-and-profiling.md) | `perf_hooks`, heap snapshots |
| [references/debugging-and-reports.md](references/debugging-and-reports.md) | inspector, diagnostic reports |
| [references/sqlite.md](references/sqlite.md) | `Database`, defaults, binding, type conversion |
| [references/sea-and-native.md](references/sea-and-native.md) | SEA, Node-API rules, `node:ffi` |
| [references/deprecations.md](references/deprecations.md) | DEP codes, removed APIs, codemods |

Pages read, version and fetch date are listed in [SOURCES.md](SOURCES.md).
