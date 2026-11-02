# Single executables, native addons, FFI, ICU

## Contents

- Single executable applications
- Node-API addons
- C++ addons and embedding
- FFI (added in v26.1.0)
- Internationalization builds
- Old patterns

## Single executable applications

Active development (Stability 1.1). https://nodejs.org/docs/latest/api/single-executable-applications.html

- Build in one step with `node --build-sea sea-config.json` (added v25.5.0). The Node version producing the blob must match the binary it is injected into. On Windows `output` must end in `.exe`. https://nodejs.org/docs/latest/api/single-executable-applications.html#generating-single-executable-applications-with---build-sea

```json
{ "main": "hello.js", "output": "sea" }
```

- Bundle the app into one JavaScript file first: in the injected main script `require()` and `import` load only built-ins. For disk modules use `require = require('node:module').createRequire(__filename)`. https://nodejs.org/docs/latest/api/single-executable-applications.html#module-loading-in-the-injected-main-script , https://nodejs.org/docs/latest/api/single-executable-applications.html#require-in-the-injected-main-script
- `__filename` is `process.execPath` and `__dirname` its directory, so `__dirname`-relative asset reads fail unless `useVfs` is on. Read assets with `sea.getAsset()` (a copy). Never write to `sea.getRawAsset()` memory. https://nodejs.org/docs/latest/api/single-executable-applications.html
- Set `useSnapshot` and `useCodeCache` to `false` when building for another platform, or the executable may crash at startup. With `useSnapshot`, `main` runs at build time and must call `v8.startupSnapshot.setDeserializeMainFunction()`. https://nodejs.org/docs/latest/api/single-executable-applications.html
- Bake flags with `execArgv`. Set `execArgvExtension: "none"` to ignore `NODE_OPTIONS` and lock them. https://nodejs.org/docs/latest/api/single-executable-applications.html#execution-argument-extension
- Native addons: embed the `.node` as an asset, write it to a temp file, `process.dlopen()` it, then delete it: https://nodejs.org/docs/latest/api/single-executable-applications.html#using-native-addons-in-the-injected-main-script

```js
const addonPath = path.join(os.tmpdir(), 'myaddon.node');
fs.writeFileSync(addonPath, new Uint8Array(getRawAsset('myaddon.node')));
const myaddon = { exports: {} };
process.dlopen(myaddon, addonPath);
fs.rmSync(addonPath);
```

- `useVfs` and `vfsArchive` (added in v26.9.0, early development) mount assets read-only. They cannot combine with `useSnapshot` or `useCodeCache`, and `getAsset()` does not serve `vfsArchive` files. https://nodejs.org/docs/latest/api/single-executable-applications.html#virtual-file-system-vfs-for-assets
- Sign on macOS (`codesign --sign - sea`). Silence the runtime warning with `disableExperimentalSEAWarning: true`. CI covers Windows, macOS arm64 and Linux except Alpine and s390x. https://nodejs.org/docs/latest/api/single-executable-applications.html#platform-support

## Node-API addons

- Prefer Node-API: ABI-stable across majors without recompiling. https://nodejs.org/docs/latest/api/addons.html
- Check the `napi_status` of every call. `napi_pending_exception` means a JS exception is pending. For any other non-ok status call `napi_is_exception_pending`. https://nodejs.org/docs/latest/api/n-api.html#return-values
- Pin the API level with `#define NAPI_VERSION 3` before `#include <node_api.h>`. Opt into experimental APIs with `NAPI_EXPERIMENTAL`. https://nodejs.org/docs/latest/api/n-api.html#node-api-version-matrix , https://nodejs.org/docs/latest/api/n-api.html#usage
- Open a handle scope per iteration in loops over large arrays. https://nodejs.org/docs/latest/api/n-api.html#making-handle-lifespan-shorter-than-that-of-the-native-method
- Delete persistent references with `napi_delete_reference`, or memory leaks permanently. https://nodejs.org/docs/latest/api/n-api.html#references-to-values-with-a-lifespan-longer-than-that-of-the-native-method
- Never cache `napi_env` or share it across threads. Register cleanup hooks to release resources in order at teardown (for example `worker.terminate()`). https://nodejs.org/docs/latest/api/n-api.html#napi_env , https://nodejs.org/docs/latest/api/n-api.html#finalization-on-the-exit-of-the-nodejs-environment
- In async work, `execute` runs off the main thread and must not call JavaScript-touching Node-API: do that in `complete`. Other threads call JS through thread-safe functions. https://nodejs.org/docs/latest/api/n-api.html#napi_async_execute_callback , https://nodejs.org/docs/latest/api/n-api.html#asynchronous-thread-safe-function-calls
- Ship prebuilt binaries (prebuildify, node-pre-gyp, prebuild), or every user needs a C/C++ toolchain. https://nodejs.org/docs/latest/api/n-api.html#building

## C++ addons and embedding

- `NODE_MODULE()` addons cannot load in several contexts or threads: make them context-aware or use Node-API. Keep no static data holding JS references. https://nodejs.org/docs/latest/api/addons.html#context-aware-addons
- `ObjectWrap` destructors do not run at process or worker shutdown: track and destroy objects yourself. https://nodejs.org/docs/latest/api/addons.html#wrapping-c-objects
- `import addon from './hello.node'` needs `--experimental-addon-modules` and only a default import works. https://nodejs.org/docs/latest/api/addons.html#loading-addons-using-import
- The embedder API may break on every major without warning: pin the Node version. https://nodejs.org/docs/latest/api/embedding.html

## FFI (added in v26.1.0)

Experimental, `node:` scheme only, `--allow-ffi` under the Permission Model. Not on v24. https://nodejs.org/docs/latest/api/ffi.html

- Wrong pointers, signatures or lifetimes crash or corrupt memory: Node tracks none of them. Prefer copied values over zero-copy views. https://nodejs.org/docs/latest/api/ffi.html , https://nodejs.org/docs/latest/api/ffi.html#ffitobufferpointer-length-copy

```js
import { dlopen, suffix } from 'node:ffi';
using handle = dlopen(`./mylib.${suffix}`, {
  add_i32: { arguments: ['int32', 'int32'], return: 'int32' },
});
handle.functions.add_i32(20, 22);
```

- 64-bit integer and pointer return values are always `bigint`. Never resize or detach a buffer passed to a call. Callbacks run on the same thread and must not throw or return promises. https://nodejs.org/docs/latest/api/ffi.html#libraryregistercallbacksignature-callback

## Internationalization builds

- Official binaries ship full ICU. With `small-icu`, locale data is English-only, so other locales generally do not work. With `none`, `Intl` is missing and `normalize()` is a no-op. https://nodejs.org/docs/latest/api/intl.html#options-for-building-nodejs
- Detect full ICU: https://nodejs.org/docs/latest/api/intl.html#detecting-internationalization-support

```js
const hasFullICU = (() => { try {
  return new Intl.DateTimeFormat('es', { month: 'long' }).format(new Date(9e8)) === 'enero';
} catch { return false; } })();
```

## Old patterns

| Old pattern | Replacement | Source |
| :-- | :-- | :-- |
| `--experimental-sea-config` plus `postject` | `--build-sea` | https://nodejs.org/docs/latest/api/single-executable-applications.html#dumping-the-preparation-blob-to-disk |
| `nan` or raw V8 addons | Node-API | https://nodejs.org/docs/latest/api/addons.html |
| `napi_open_callback_scope` `resource_object` | `async_resource` of `napi_async_init` | https://nodejs.org/docs/latest/api/n-api.html#napi_open_callback_scope |
