# Permission Model

## Contents

- What it is and is not
- Enabling and auditing
- File system grants
- Known gaps
- Dropping permissions at run time
- Old patterns

## What it is and is not

- The Permission Model is stable since v23.5.0 (backported to v22.13.0). Most `--allow-*` sub-flags are in active development (1.1), `--allow-fs-read`/`--allow-fs-write` are stable. https://nodejs.org/docs/latest/api/permissions.html#permission-model
- It is a seat belt for trusted code, not a sandbox: it gives no guarantee against malicious code. Never rely on it to run untrusted code. https://nodejs.org/docs/latest/api/permissions.html#process-based-permissions
- `node:vm`, `node:wasi` and `node:vfs` are not security boundaries either: isolate untrusted code with OS users, containers or sandboxes. https://nodejs.org/docs/latest/api/vm.html , https://nodejs.org/docs/latest/api/wasi.html#security , https://nodejs.org/docs/latest/api/vfs.html#security

## Enabling and auditing

- `--permission` enforces: a denied operation throws `ERR_ACCESS_DENIED` with `permission` and `resource`. Restricted: fs, network, child processes, workers, addons, WASI, FFI, OpenSSL STORE loaders and the inspector. https://nodejs.org/docs/latest/api/permissions.html#permission-model
- Start with `--permission-audit` (added in v25.8.0): violations are published on `diagnostics_channel` and execution continues. `--permission` wins if both are given. https://nodejs.org/docs/latest/api/permissions.html#audit-mode

```js
require('node:diagnostics_channel').channel('node:permission-model:fs')
  .subscribe((msg) => console.log(`Permission denied: ${msg.permission} on ${msg.resource}`));
```

Channels: `node:permission-model:` followed by `fs`, `net`, `child`, `worker`, `inspector`, `wasi`, `addon`, `ffi`. https://nodejs.org/docs/latest/api/permissions.html#audit-mode

- Grants: `--allow-fs-read`, `--allow-fs-write`, `--allow-net` (added in v25.0.0), `--allow-child-process`, `--allow-worker`, `--allow-addons`, `--allow-wasi`, `--allow-inspector` (added in v25.0.0, early development), `--allow-ffi` (added in v26.1.0), `--allow-openssl-store` (added in v26.7.0), `--allow-fs-vfs` (added in v26.9.0, gates `vfs.mount()`). https://nodejs.org/docs/latest/api/cli.html#--allow-fs-vfs
- In a config file, the presence of a `permission` object turns `--permission` on: https://nodejs.org/docs/latest/api/permissions.html#configuration-file-support

```json
{ "permission": { "allow-fs-read": ["./foo"], "allow-fs-write": ["./bar"], "allow-child-process": true } }
```

- `--allow-child-process` grants are inherited by Node children: `fork()` passes them as arguments, `spawn()` via `NODE_OPTIONS`. https://nodejs.org/docs/latest/api/cli.html#--allow-child-process
- With `npx`, grant read access to the global prefix or npx cache: https://nodejs.org/docs/latest/api/permissions.html#using-the-permission-model-with-npx

```bash
npx --node-options="--permission --allow-fs-read=$(npm prefix -g)" package-name
```

## File system grants

- Repeat the flag for each path (comma lists were removed). Values are `*`, cwd-relative or absolute paths. The entry point and `--require` modules are readable implicitly. https://nodejs.org/docs/latest/api/permissions.html#file-system-permissions
- An existing directory gets an automatic trailing wildcard. A directory that does not exist yet does not: write `/my-path/folder/*`. https://nodejs.org/docs/latest/api/permissions.html#file-system-permissions
- Everything after a `*` is ignored: `/home/*.js` grants `/home/*`. `/home/test*` also matches `/home/test2`. https://nodejs.org/docs/latest/api/permissions.html#file-system-permissions

## Known gaps

- Symlinks are followed outside granted paths. Make sure granted directories contain no relative symlinks. https://nodejs.org/docs/latest/api/permissions.html#limitations-and-known-issues
- Existing file descriptors bypass the model through `node:fs`, while `fchmod`, `fchown`, `futimes`, `fsync` on them throw. https://nodejs.org/docs/latest/api/permissions.html#file-system-permissions , https://nodejs.org/docs/latest/api/permissions.html#permission-model-constraints
- `node:sqlite` file access is not covered by fs grants. https://nodejs.org/docs/latest/api/permissions.html#file-system-permissions
- `--env-file`, `--openssl-config` and `v8.setFlagsFromString()` act outside the model. https://nodejs.org/docs/latest/api/permissions.html#permission-model-constraints
- `process._debugProcess(pid)` is not gated: a restricted process can open the inspector of any same-user Node process. https://nodejs.org/docs/latest/api/permissions.html#processdebugprocess-and-cross-process-inspector-activation
- A `Worker` with an explicit `execArgv` (even `[]`) loses inherited permission flags. Omit `execArgv` or repeat the flags. https://nodejs.org/docs/latest/api/permissions.html#permission-model-constraints
- `--allow-openssl-store` grants broad authority outside fs and net scopes. https://nodejs.org/docs/latest/api/permissions.html#permission-model

## Dropping permissions at run time

- `process.permission.has(scope, ref)` checks a grant. `process.permission.drop(scope, ref)` (added in v26.3.0, backported to v24.20.0, active development) removes it irreversibly for future checks only: open fds, sockets, children and workers stay. Only the exact granted resource can be dropped. https://nodejs.org/docs/latest/api/permissions.html#permissiondropscope-reference

```js
const config = fs.readFileSync('/etc/myapp/config.json', 'utf8');
process.permission.drop('fs.read', '/etc/myapp');
process.permission.drop('child');
```
https://nodejs.org/docs/latest/api/process.html#processpermissiondropscope-reference

## Old patterns

| Old pattern | Replacement | Source |
| :-- | :-- | :-- |
| `--allow-fs-read=/a,/b` | `--allow-fs-read=/a --allow-fs-read=/b` | https://nodejs.org/docs/latest/api/permissions.html#file-system-permissions |
| `Worker` with `execArgv: []` under `--permission` | omit `execArgv` to inherit grants | https://nodejs.org/docs/latest/api/worker_threads.html#new-workerfilename-options |
| `process.binding()` | unavailable under the model, use public APIs | https://nodejs.org/docs/latest/api/deprecations.html#DEP0111 |
