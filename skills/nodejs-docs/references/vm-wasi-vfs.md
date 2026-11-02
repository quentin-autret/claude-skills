# node:vm, node:wasi and node:vfs

None of these is a security boundary. Isolate untrusted code with OS users, containers or sandboxes. https://nodejs.org/docs/latest/api/vm.html , https://nodejs.org/docs/latest/api/wasi.html#security , https://nodejs.org/docs/latest/api/vfs.html#security

## Contents

- vm contexts
- vm timeouts and microtasks
- vm modules and dynamic import
- WASI (experimental)
- VFS (added in v26.4.0)
- Old patterns

## vm contexts

- "The node:vm module is not a security mechanism. Do not use it to run untrusted code." https://nodejs.org/docs/latest/api/vm.html
- A contextified object acts as the global, and `var` writes land on it. The inner `globalThis` is not the same object and cannot be frozen. https://nodejs.org/docs/latest/api/vm.html#what-does-it-mean-to-contextify-an-object
- Prefer `vm.createContext(vm.constants.DONT_CONTEXTIFY)` for fewer quirks: the returned object is the inner `globalThis` and can be frozen. https://nodejs.org/docs/latest/api/vm.html#vmconstantsdont_contextify

```js
const ctx = vm.createContext(vm.constants.DONT_CONTEXTIFY);
vm.runInContext('Object.freeze(globalThis);', ctx);
```

- `codeGeneration: { strings: false, wasm: false }` makes `eval`, `Function` and Wasm compilation throw inside the context. https://nodejs.org/docs/latest/api/vm.html#vmcreatecontextcontextobject-options

## vm timeouts and microtasks

- `timeout` and `breakOnSigint` are escaped by promises: microtasks run after the stack unwinds. Set `microtaskMode: 'afterEvaluate'` on the context. Callbacks from the outer context and timers exposed inside still escape. https://nodejs.org/docs/latest/api/vm.html#timeout-interactions-with-asynchronous-tasks-and-promises

```js
vm.runInNewContext('Promise.resolve().then(() => loop());', { loop, console },
  { timeout: 5, microtaskMode: 'afterEvaluate' });
```

- With `'afterEvaluate'` the context has its own microtask queue: awaiting an inner promise from outside hangs until another `runInContext()` drains it. https://nodejs.org/docs/latest/api/vm.html#when-microtaskmode-is-afterevaluate-beware-sharing-promises-between-contexts

## vm modules and dynamic import

- `vm.Module`, `SourceTextModule`, `SyntheticModule` and `measureMemory()` are experimental. Modules need `--experimental-vm-modules`. https://nodejs.org/docs/latest/api/vm.html#class-vmmodule
- Without `importModuleDynamically`, `import()` in compiled code rejects with `ERR_VM_DYNAMIC_IMPORT_CALLBACK_MISSING`. The option is part of the experimental modules API and not recommended in production. https://nodejs.org/docs/latest/api/vm.html#support-of-dynamic-import-in-compilation-apis
- `vm.constants.USE_MAIN_CONTEXT_DEFAULT_LOADER` gives sandboxed code the main loader, hence `fs` and `http`. Its objects are not `instanceof` the context's built-ins. https://nodejs.org/docs/latest/api/vm.html#when-importmoduledynamically-is-vmconstantsuse_main_context_default_loader

## WASI (experimental)

- `version` is required (`'preview1'` or `'unstable'`). `returnOnExit` defaults to `true`, so `proc_exit` returns the code from `start()`. https://nodejs.org/docs/latest/api/wasi.html#new-wasioptions

```js
const wasi = new WASI({ version: 'preview1', args: argv, env, preopens: { '/local': '/some/real/path' } });
const instance = await WebAssembly.instantiate(wasm, wasi.getImportObject());
wasi.start(instance);
```

- `start()` is for commands (`_start` and exported `memory`), `initialize()` for reactors (`_initialize`). Each can be called once. https://nodejs.org/docs/latest/api/wasi.html
- Preopens are not a sandbox, they can be escaped. https://nodejs.org/docs/latest/api/wasi.html#security

## VFS (added in v26.4.0)

Experimental, needs `--experimental-vfs` and the `node:` scheme. https://nodejs.org/docs/latest/api/vfs.html

- `mount()` returns a runtime-chosen mount point inside a reserved namespace: never hard-code it. Module resolution of a missing path under the mount fails with `ENOENT` without consulting the disk. https://nodejs.org/docs/latest/api/vfs.html#module-loader-integration

```js
using myVfs = vfs.create();
myVfs.writeFileSync('/data.txt', 'Hello');
const mountPoint = myVfs.mount();
fs.readFileSync(`${mountPoint}/data.txt`, 'utf8');
```

- Only `node:fs` calls under the mount are redirected. `require()` and `import` work there, and the mount acts as a root for `node_modules` lookup. https://nodejs.org/docs/latest/api/vfs.html#module-loader-integration
- Providers: `MemoryProvider`, `RealFSProvider(root)` (its escape check is not a security boundary), `ZipProvider` (v26.9.0), `ComposableProvider` (v26.12.0). Custom providers go through `vfs.registerProvider()` (v26.10.0, selected by `--vfs-load`) and must be registered before the source is mounted. https://nodejs.org/docs/latest/api/vfs.html

## Old patterns

| Old pattern | Replacement | Source |
| :-- | :-- | :-- |
| `new vm.Script(code, { produceCachedData: true })` | `script.createCachedData()` | https://nodejs.org/docs/latest/api/deprecations.html#DEP0110 |
| `sourceTextModule.dependencySpecifiers` | `sourceTextModule.moduleRequests` | https://nodejs.org/docs/latest/api/vm.html#sourcetextmoduledependencyspecifiers |
| `new WASI()` without `version` | pass `version` (required) | https://nodejs.org/docs/latest/api/wasi.html#new-wasioptions |
