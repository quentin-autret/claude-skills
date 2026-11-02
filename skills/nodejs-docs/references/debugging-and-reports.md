# Inspector, debugger, diagnostic reports, trace events

## Contents

- Inspector
- node inspect and probe mode
- Diagnostic reports
- Trace events
- Old patterns

## Inspector

- Never bind the inspector to a public IP and port: the docs call it insecure. `inspector.open(port, host, wait)` equals `--inspect` at run time and returns a disposable. https://nodejs.org/docs/latest/api/inspector.html#inspectoropenport-host-wait
- `node:inspector/promises` is experimental. https://nodejs.org/docs/latest/api/inspector.html#promises-api

```js
import { Session } from 'node:inspector/promises';
const session = new Session();
session.connect();
await session.post('Profiler.enable');
await session.post('Profiler.start');
// ... business logic
const { profile } = await session.post('Profiler.stop');
fs.writeFileSync('./profile.cpuprofile', JSON.stringify(profile));
```
https://nodejs.org/docs/latest/api/inspector.html#example-usage

- Never set breakpoints through a same-thread `session.connect()`: the paused program is the debugger. Use `connectToMainThread()` from a Worker or an external debugger. https://nodejs.org/docs/latest/api/inspector.html#support-of-breakpoints
- With a `Session`, console output objects are kept until `Runtime.DiscardConsoleEntries`. https://nodejs.org/docs/latest/api/inspector.html#new-inspectorsession
- `session.disconnect()` drops all inspector state. https://nodejs.org/docs/latest/api/inspector.html#sessiondisconnect
- DevTools network integration (`inspector.Network.*`) is in active development. https://nodejs.org/docs/latest/api/inspector.html#integration-with-devtools
- Under the Permission Model, a restricted process can still open another same-user process's inspector via `process._debugProcess`. https://nodejs.org/docs/latest/api/permissions.html#process_debugprocess-and-cross-process-inspector-activation

## node inspect and probe mode

- `node inspect` stops on the first line. `NODE_INSPECT_RESUME_ON_START=1` runs to the first `debugger` statement. https://nodejs.org/docs/latest/api/debugger.html#interactive-mode
- Probe mode (`node inspect --probe`, added in v26.1.0, experimental) is non-interactive printf debugging that prints one final report. It only launches a new process. https://nodejs.org/docs/latest/api/debugger.html#probe-mode

```console
$ node inspect --probe app.js:10 --expr "user" --probe src/utils.js:5:15 --expr "config.options" --json -- app.js
```

- Each `--probe file:line[:col]` takes exactly one `--expr`. `--cond` and `--max-hit` follow their pair. `--timeout` defaults to 30000 ms. `--` passes Node flags to the child. https://nodejs.org/docs/latest/api/debugger.html#probe-mode
- Do not probe a `let`/`const` on its declaration line: you get a TDZ `ReferenceError`. https://nodejs.org/docs/latest/api/debugger.html#probe-mode
- The JSON report is schema `v: 2` and always ends with a terminal entry (`completed`, `miss`, `timeout`, `error`). The exit code is non-zero for probe failures and timeouts. https://nodejs.org/docs/latest/api/debugger.html#probe-mode

## Diagnostic reports

- Reports are meant for production too: JS and native stacks, heap, resources. https://nodejs.org/docs/latest/api/report.html

```bash
node --report-uncaught-exception --report-on-signal --report-on-fatalerror app.js
```

- Reports include environment variables and network interfaces by default. Use `--report-exclude-env` and `--report-exclude-network` when reports leave the host. The latter also avoids slow reverse DNS. https://nodejs.org/docs/latest/api/report.html#usage
- `--report-on-signal` uses `SIGUSR2` and is unsupported on Windows. `--report-compact` writes one-line JSON for log pipelines. https://nodejs.org/docs/latest/api/report.html#usage
- Pass the caught error to keep its stack: `process.report.writeReport(err)` (filename first when both). https://nodejs.org/docs/latest/api/report.html#usage

```js
try { process.chdir('/non-existent-path'); } catch (err) { process.report.writeReport(err); }
```

- Check `header.reportVersion` before parsing reports across Node versions. https://nodejs.org/docs/latest/api/report.html#report-version
- Reports include a `workers` section for child Workers. https://nodejs.org/docs/latest/api/report.html#interaction-with-workers

## Trace events

- Trace events are experimental and unavailable in workers. Enable with `--trace-event-categories node.perf,node.fs.sync` or `trace_events.createTracing({ categories }).enable()`. Open the file in `chrome://tracing`. https://nodejs.org/docs/latest/api/tracing.html
- Install `SIGINT`/`SIGTERM` handlers that exit cleanly, or the log may not be written: `process.on('SIGINT', () => { process.exit(130); })`. https://nodejs.org/docs/latest/api/tracing.html
- `tracing.disable()` keeps categories enabled by other `Tracing` objects or the CLI. https://nodejs.org/docs/latest/api/tracing.html

## Old patterns

| Old pattern | Replacement | Source |
| :-- | :-- | :-- |
| `--trace-events-enabled` (no longer required) | `--trace-event-categories` (the old flag only enabled the default `node`, `node.async_hooks`, `v8`) | https://nodejs.org/docs/latest/api/tracing.html |
| `console.log` instrumentation edits | `node inspect --probe` (v26.1.0+) | https://nodejs.org/docs/latest/api/debugger.html#probe-mode |
