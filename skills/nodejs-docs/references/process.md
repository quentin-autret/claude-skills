# node:process: exit, signals, errors, I/O

## Contents

- Exiting
- Signals
- Uncaught exceptions and rejections
- process.env
- stdout, stderr and console
- Scheduling and timing
- Feature detection and newer APIs
- Old patterns

## Exiting

- Set `process.exitCode = 1` and let the loop drain instead of calling `process.exit()`: writes to `stdout` can be async and `exit()` can cut them off. https://nodejs.org/docs/latest/api/process.html#processexitcode

```js
if (someConditionNotMet()) { printUsageToStdout(); process.exitCode = 1; }
```

- `exit()` and `exitCode` accept only integers or integer strings (DEP0164). https://nodejs.org/docs/latest/api/deprecations.html#DEP0164
- In a Worker, `process.exit()` ends the thread only. https://nodejs.org/docs/latest/api/process.html#processexitcode
- `'exit'` listeners must be synchronous: timers and I/O scheduled there are abandoned. https://nodejs.org/docs/latest/api/process.html#event-exit
- `'beforeExit'` is not emitted on `process.exit()` or an uncaught exception, and a listener that schedules async work keeps the process alive. https://nodejs.org/docs/latest/api/process.html#event-beforeexit
- Exit codes to know: 1 uncaught exception, 9 invalid argument, 13 unsettled top-level `await`, 124 `--process-timeout` (added in v26.11.0), above 128 killed by a signal (128 + number). https://nodejs.org/docs/latest/api/process.html#exit-codes
- `process.finalization.register(obj, fn)` runs `fn` at exit only if `obj` was not collected. Pass a function that does not close over unnecessary objects, which would keep them alive. Release candidate since v26.12.0. https://nodejs.org/docs/latest/api/process.html#processfinalizationregisterref-callback

```js
function onFinalize(obj, event) { obj.dispose(); }
finalization.register(myDisposableObject, onFinalize);
```

## Signals

- Adding any `SIGINT` or `SIGTERM` listener removes the default behavior (exit with `128 + signal number` on non-Windows), so the handler decides whether to end the process. Listeners are async and may not run if the loop is already empty. https://nodejs.org/docs/latest/api/process.html#signal-events
- `SIGKILL` and `SIGSTOP` cannot be caught. `SIGUSR1` is reserved for the debugger (`--disable-sigusr1` turns that off). Workers receive no signals. https://nodejs.org/docs/latest/api/process.html#signal-events , https://nodejs.org/docs/latest/api/cli.html#--disable-sigusr1
- On Windows, `SIGTERM` is not supported (it can only be listened on), `SIGHUP` terminates the process about 10 s later even with a listener, and `kill()` with SIGINT, SIGTERM or SIGKILL terminates unconditionally. `process.kill(pid, 0)` tests existence everywhere. https://nodejs.org/docs/latest/api/process.html#signal-events

## Uncaught exceptions and rejections

- Never resume normal operation from `'uncaughtException'`: the process is in an undefined state. Do synchronous cleanup, exit, and let an external supervisor restart it. https://nodejs.org/docs/latest/api/process.html#warning-using-uncaughtexception-correctly
- With an `'uncaughtException'` handler and no `exitCode`, the process exits with 0, not 1. https://nodejs.org/docs/latest/api/process.html#event-uncaughtexception
- To log without changing crash behavior, use `'uncaughtExceptionMonitor'`. https://nodejs.org/docs/latest/api/process.html#event-uncaughtexceptionmonitor

```js
process.on('uncaughtExceptionMonitor', (err, origin) => { MyMonitoringTool.logSync(err, origin); });
```

- Unhandled rejections are emitted as `'unhandledRejection'`, or raised as an uncaught exception when no hook is set (`--unhandled-rejections=throw`, default). A promise stored without a handler for a turn (for example `this.loaded = Promise.reject(...)` in a constructor) triggers it: attach `.catch(() => {})` to opt out. https://nodejs.org/docs/latest/api/process.html#event-unhandledrejection
- `--unhandled-rejections` modes: `throw` (default), `strict`, `warn`, `warn-with-error-code`, `none`. https://nodejs.org/docs/latest/api/cli.html#--unhandled-rejections=mode
- `process.addUncaughtExceptionCaptureCallback()` is experimental, added in v25.9.0. https://nodejs.org/docs/latest/api/process.html#processadduncaughtexceptioncapturecallbackfn

## process.env

- Changes are local to the process. Only the main thread's changes reach the OS and addons, and each Worker gets a copy. https://nodejs.org/docs/latest/api/process.html#processenv
- Assign strings only and remove keys with `delete`: `env.X = undefined` stores `'undefined'`, and non-string values will throw in future. https://nodejs.org/docs/latest/api/deprecations.html#DEP0104
- On Windows names are case-insensitive on the main thread but case-sensitive in a Worker copy. https://nodejs.org/docs/latest/api/process.html#processenv
- Keep secrets out of diagnostic reports with `process.report.excludeEnv` or `--report-exclude-env`. https://nodejs.org/docs/latest/api/process.html#processreportexcludeenv , https://nodejs.org/docs/latest/api/cli.html#--report-exclude-env

## stdout, stderr and console

- Writes are sync for files, sync for TTYs on POSIX but async on Windows, async for pipes and sockets on POSIX but sync on Windows. Sync writes to a slow or unread pipe block the loop. https://nodejs.org/docs/latest/api/process.html#a-note-on-process-io
- `console` inherits this: it is neither consistently sync nor async. https://nodejs.org/docs/latest/api/console.html
- Detect a terminal with `process.stdout.isTTY` (falsy when piped). https://nodejs.org/docs/latest/api/tty.html
- Warnings go to stderr. Silence one code with `--disable-warning=CODE`, show stacks with `--trace-warnings`. https://nodejs.org/docs/latest/api/process.html#event-warning , https://nodejs.org/docs/latest/api/cli.html#--disable-warningcode-or-type

## Scheduling and timing

- Default to `queueMicrotask()`. `process.nextTick()` is legacy, kept for extra arguments or for running before promise callbacks in CommonJS. In ESM, microtasks run first. Recursive `nextTick` starves the loop. https://nodejs.org/docs/latest/api/process.html#when-to-use-queuemicrotask-vs-processnexttick

```js
queueMicrotask(deferred.bind(undefined, 1, 2));
```

- Measure with `process.hrtime.bigint()`. https://nodejs.org/docs/latest/api/process.html#processhrtimetime

## Feature detection and newer APIs

- Detect with `process.features.typescript` and `process.features.require_module`. https://nodejs.org/docs/latest/api/process.html
- `process.constrainedMemory()` reports container limits, `process.availableMemory()` free memory. `process.getBuiltinModule(id)` loads a built-in without import. https://nodejs.org/docs/latest/api/process.html
- `process.execve()` (experimental, POSIX) replaces the process image. https://nodejs.org/docs/latest/api/process.html#processexecvefile-args-env
- `process.permission` is available through `--permission` or `--permission-audit`. https://nodejs.org/docs/latest/api/process.html#processpermission

## Old patterns

| Old pattern | Replacement | Source |
| :-- | :-- | :-- |
| `process.exit(1)` after output | `process.exitCode = 1` | https://nodejs.org/docs/latest/api/process.html#processexitcode |
| `process.nextTick(fn)` | `queueMicrotask(fn)` | https://nodejs.org/docs/latest/api/process.html#processnexttickcallback-args |
| `process.hrtime()` | `process.hrtime.bigint()` | https://nodejs.org/docs/latest/api/process.html#processhrtimetime |
| `process.mainModule` | `require.main`, or `import.meta.main` in ESM | https://nodejs.org/docs/latest/api/deprecations.html#DEP0138 |
| `process.umask()` with no argument | pass a mask, there is no safe read | https://nodejs.org/docs/latest/api/deprecations.html#DEP0139 |
| `process.setSourceMapsEnabled()` | `module.setSourceMapsSupport()` | https://nodejs.org/docs/latest/api/process.html#processsetsourcemapsenabledval |
| `process.features.ipv6`, `.uv` | drop the check, always `true` | https://nodejs.org/docs/latest/api/process.html#processfeaturesipv6 |
| `process.features.tls_alpn`, `.tls_ocsp`, `.tls_sni` | `process.features.tls` | https://nodejs.org/docs/latest/api/process.html#processfeaturestls_alpn |
| `process.binding()` | public APIs | https://nodejs.org/docs/latest/api/deprecations.html#DEP0111 |
| `process.on('multipleResolves')` | removed | https://nodejs.org/docs/latest/api/deprecations.html#DEP0160 |
