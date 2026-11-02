# node:child_process

## Contents

- Choosing the function
- Shells and injection
- Output, buffers and events
- Killing and detaching
- IPC and fork
- Environment and Windows
- Old patterns

## Choosing the function

- Default to `execFile()` or `spawn()`, which run without a shell. Use `exec()` only when you need shell features (redirection, globbing). https://nodejs.org/docs/latest/api/child_process.html#child_processexecfilefile-args-options-callback
- The `*Sync` variants block the event loop: fine for scripts and startup, not for servers. https://nodejs.org/docs/latest/api/child_process.html#synchronous-process-creation
- Each Node child has its own memory and V8 instance: do not spawn many. https://nodejs.org/docs/latest/api/child_process.html#child_processforkmodulepath-args-options
- The promisified `execFile` resolves `{ stdout, stderr }` and rejects on a non-zero exit with `stdout`/`stderr` on the error. https://nodejs.org/docs/latest/api/child_process.html#child_processexecfilefile-args-options-callback

```js
const execFile = promisify(child_process.execFile);
const { stdout } = await execFile('node', ['--version']);
```

## Shells and injection

- Never pass unsanitized input to `exec()`, or to `spawn()`/`execFile()` with `shell` enabled: metacharacters run arbitrary commands. https://nodejs.org/docs/latest/api/child_process.html#child_processexeccommand-options-callback
- With `shell: true`, an `args` array is not escaped, only joined with spaces. This is a runtime deprecation since v24.0.0. https://nodejs.org/docs/latest/api/deprecations.html#DEP0190
- `shell: ''` is almost certainly a bug: use `true` for the default shell, or omit it. https://nodejs.org/docs/latest/api/deprecations.html#DEP0196
- Run `.bat`/`.cmd` files with `exec('my.bat')` or `spawn('cmd.exe', ['/c', 'my.bat'])`, quoting names with spaces. https://nodejs.org/docs/latest/api/child_process.html#spawning-bat-and-cmd-files-on-windows

## Output, buffers and events

- Default pipes have limited capacity: a child that writes more than you read blocks. Consume the output or use `stdio: 'ignore'`. https://nodejs.org/docs/latest/api/child_process.html#child-process
- `exec`/`execFile` `maxBuffer` is 1 MiB, counted in bytes. Exceeding it kills the child and truncates output. https://nodejs.org/docs/latest/api/child_process.html#maxbuffer-and-unicode
- Always attach `'error'`: a missing command or bad `cwd` (`ENOENT`), a failed kill or send all emit it, and `'exit'` may or may not follow. https://nodejs.org/docs/latest/api/child_process.html#event-error

```js
const subprocess = spawn('bad_command');
subprocess.on('error', (err) => { console.error('Failed to start subprocess.'); });
```

- Wait for `'close'`, not `'exit'`, to be sure all output was read. On a signal `code` is `null`: convert with `util.convertProcessSignalToExitCode()`. https://nodejs.org/docs/latest/api/child_process.html#event-close , https://nodejs.org/docs/latest/api/child_process.html#event-exit
- Pass `stdin` as a readable and `stdout`/`stderr` as writables with a real fd (wait for a file stream's `'open'`). The reverse is discouraged and may cause undefined behavior or dropped callbacks. https://nodejs.org/docs/latest/api/child_process.html#optionsstdio

## Killing and detaching

- `kill()` only sends a signal. On Linux, killing a shell-spawned child does not kill its grandchildren. On Windows most signals terminate forcefully. https://nodejs.org/docs/latest/api/child_process.html#subprocesskillsignal
- A detached long-running child needs `detached: true`, `stdio` not connected to the parent, and `unref()`: https://nodejs.org/docs/latest/api/child_process.html#optionsdetached

```js
const out = openSync('./out.log', 'a');
spawn('prg', [], { detached: true, stdio: ['ignore', out, out] }).unref();
```

- Dispose with `using child = spawn(command, args)` (disposal calls `kill()` with `SIGTERM`) rather than an `AbortSignal`. Keep `signal` for cancelling or timing out work. https://nodejs.org/docs/latest/api/deprecations.html#DEP0209

## IPC and fork

- `fork()` ignores `shell`, needs exactly one `'ipc'` in `stdio`, and does not clone the process. https://nodejs.org/docs/latest/api/child_process.html#child_processforkmodulepath-args-options
- A child's IPC channel stays unref'd until it listens for `'message'` or `'disconnect'`. https://nodejs.org/docs/latest/api/child_process.html#optionsstdio
- Messages are serialized and may differ from what was sent. `serialization: 'advanced'` adds BigInt, Map, Set, typed arrays, Error and RegExp, but drops extra properties. Do not send `{ cmd: 'NODE_...' }`. https://nodejs.org/docs/latest/api/child_process.html#advanced-serialization , https://nodejs.org/docs/latest/api/child_process.html#subprocesssendmessage-sendhandle-options-callback
- `send()` returns `false` when the backlog is full: use its callback for flow control. Sockets cannot be sent on Windows. `net.BoundSocket` can be sent since v26.10.0. https://nodejs.org/docs/latest/api/child_process.html#subprocesssendmessage-sendhandle-options-callback

## Environment and Windows

- If `env` lacks `PATH`, Unix lookup uses `/usr/bin:/bin`. `undefined` values are skipped. https://nodejs.org/docs/latest/api/child_process.html#child_processspawncommand-args-options
- On Windows env keys are case-insensitive and the first match wins (`PATH` versus `Path`). https://nodejs.org/docs/latest/api/child_process.html#child-process
- In a Node child `process.argv[0]` is `process.execPath`: read `process.argv0` for the original. https://nodejs.org/docs/latest/api/child_process.html#child_processspawncommand-args-options
- Permission flags granted with `--allow-child-process` reach Node children. https://nodejs.org/docs/latest/api/cli.html#--allow-child-process

## Old patterns

| Old pattern | Replacement | Source |
| :-- | :-- | :-- |
| `spawn(cmd, args, { shell: true })` | `spawn(cmd, args)` without a shell | https://nodejs.org/docs/latest/api/deprecations.html#DEP0190 |
| `spawn(cmd, args, { signal })` as cleanup | `using child = spawn(cmd, args)` | https://nodejs.org/docs/latest/api/deprecations.html#DEP0209 |
| `{ shell: '' }` | `{ shell: true }` or omit | https://nodejs.org/docs/latest/api/deprecations.html#DEP0196 |
| listening only to `'exit'` for output | `'close'` | https://nodejs.org/docs/latest/api/child_process.html#event-close |
