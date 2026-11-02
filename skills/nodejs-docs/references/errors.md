# Errors

## Contents

- Identifying errors
- Stack traces
- How errors propagate
- Error codes worth knowing
- Domains (deprecated)
- Old patterns

## Identifying errors

- Branch on `error.code`, never on `error.message`: messages may change in any version, codes only in a major. For `DOMException` use `.name`. https://nodejs.org/docs/latest/api/errors.html#errorcode
- Chain with `new Error(msg, { cause })`. `util.inspect()` prints causes recursively. https://nodejs.org/docs/latest/api/errors.html#errorcause

```js
const cause = new Error('The remote HTTP server responded with a 500 status');
const symptom = new Error('The message failed to send', { cause });
```

- Test for built-in errors with `Error.isError(x)`. https://nodejs.org/docs/latest/api/errors.html#errorstack
- User code may throw non-Errors. Some failures (C++ `abort()`) always crash and cannot be caught. https://nodejs.org/docs/latest/api/errors.html#exceptions-vs-errors
- Errors from `crypto` and `tls` may carry `opensslErrorStack`, `library`, `reason`. https://nodejs.org/docs/latest/api/errors.html#openssl-errors

## Stack traces

- Traces stop at `Error.stackTraceLimit` (default 10) frames. A negative or non-number value captures none, and changes affect only later traces. https://nodejs.org/docs/latest/api/errors.html#errorstacktracelimit
- `error.stack` lives in a hidden slot of built-in errors: if the accessor runs with a non-built-in `this` such as a Proxy, the getter returns `undefined` and the setter does nothing. Changing `message` after reading `stack` may not update it. https://nodejs.org/docs/latest/api/errors.html#errorstack , https://nodejs.org/docs/latest/api/errors.html#errormessage
- Hide internal frames without computing the stack twice: https://nodejs.org/docs/latest/api/errors.html#errorcapturestacktracetargetobject-constructoropt

```js
const { stackTraceLimit } = Error;
Error.stackTraceLimit = 0;
const error = new Error();
Error.stackTraceLimit = stackTraceLimit;
Error.captureStackTrace(error, b); // frames above and including b omitted
throw error;
```

- Native addon frames do not appear in traces. https://nodejs.org/docs/latest/api/errors.html#errorstack
- `util.getCallSites()` (active development) reads call sites without `Error.prepareStackTrace`. Use `columnNumber`, not the deprecated `column`. https://nodejs.org/docs/latest/api/util.html#utilgetcallsitesframecount-options

## How errors propagate

- Sync APIs throw, promise APIs reject, callback APIs pass the error first, emitters emit `'error'`. A few "typically asynchronous" methods still throw: read each method's docs. https://nodejs.org/docs/latest/api/errors.html#error-propagation-and-interception
- An `'error'` emitted asynchronously cannot be caught by `try/catch`, and with no listener it crashes the process. https://nodejs.org/docs/latest/api/errors.html#error-propagation-and-interception
- A callback called twice raises `ERR_MULTIPLE_CALLBACK`. https://nodejs.org/docs/latest/api/errors.html#err_multiple_callback
- An exception in an `EventTarget` listener becomes an uncaught exception on the next tick. Do not rely on `process.on('error')`, which sees it first: the documentation says that behavior will change. https://nodejs.org/docs/latest/api/events.html#eventtarget-error-handling
- Node-API callback exceptions are suppressed by default. `--force-node-api-uncaught-exceptions-policy` emits `'uncaughtException'` instead. https://nodejs.org/docs/latest/api/deprecations.html#DEP0168

## Error codes worth knowing

| Code | Meaning | Source |
| :-- | :-- | :-- |
| `ERR_UNHANDLED_ERROR` | `'error'` emitted with no listener | https://nodejs.org/docs/latest/api/errors.html#err_unhandled_error |
| `ERR_STREAM_PREMATURE_CLOSE` | `stream.finished()` or `stream.pipeline()` ended non gracefully with no explicit error | https://nodejs.org/docs/latest/api/errors.html#err_stream_premature_close |
| `ERR_REQUIRE_ASYNC_MODULE` | `require()` of ESM with top-level `await` | https://nodejs.org/docs/latest/api/errors.html#err_require_async_module |
| `ERR_REQUIRE_CYCLE_MODULE` | CJS and ESM `require()` cycle, require lazily inside a function | https://nodejs.org/docs/latest/api/errors.html#err_require_cycle_module |
| `ERR_REQUIRE_ESM_RACE_CONDITION` | `require()` while `import()` of the same ESM is in flight (added in v26.1.0) | https://nodejs.org/docs/latest/api/errors.html#err_require_esm_race_condition |
| `ERR_MISSING_ARGS` | strict spec check, most APIs use `ERR_INVALID_ARG_TYPE` | https://nodejs.org/docs/latest/api/errors.html#err_missing_args |
| `ERR_UNCAUGHT_EXCEPTION_CAPTURE_ALREADY_SET` | capture callback set twice without resetting to `null` | https://nodejs.org/docs/latest/api/errors.html#err_uncaught_exception_capture_already_set |
| `EMFILE` | too many open files, raise `ulimit -n` | https://nodejs.org/docs/latest/api/errors.html#common-system-errors |
| `ENOTFOUND` | DNS failure, not a POSIX error | https://nodejs.org/docs/latest/api/errors.html#common-system-errors |
| `ETIMEDOUT` | often a missing `socket.end()` | https://nodejs.org/docs/latest/api/errors.html#common-system-errors |

- Since v26.5.0 `ERR_REQUIRE_ASYNC_MODULE` carries `requireStack` and `topLevelAwaitLocations`. `topLevelAwaitLocations` is only populated with `--experimental-print-required-tla`, which also prints the locations. https://nodejs.org/docs/latest/api/errors.html#err_require_async_module

## Domains (deprecated)

- `node:domain` is deprecated: do not use it. After a throw there is almost never a safe way to resume. https://nodejs.org/docs/latest/api/domain.html
- For a server, answer the failing request with an error, stop accepting in that worker and let the cluster primary fork a replacement: https://nodejs.org/docs/latest/api/domain.html#warning-dont-ignore-errors

```js
const killtimer = setTimeout(() => { process.exit(1); }, 30000);
killtimer.unref();
server.close();
cluster.worker.disconnect(); // primary: cluster.on('disconnect', () => cluster.fork())
res.statusCode = 500;
res.end('Oops, there was a problem!\n');
```

## Old patterns

| Old pattern | Replacement | Source |
| :-- | :-- | :-- |
| `err.message.includes(...)` | `err.code === '...'` | https://nodejs.org/docs/latest/api/errors.html#errorcode |
| `util.types.isNativeError(x)` | `Error.isError(x)` | https://nodejs.org/docs/latest/api/deprecations.html#DEP0197 |
| `ERR_REQUIRE_ESM` handling | `require()` loads sync ESM, handle `ERR_REQUIRE_ASYNC_MODULE` | https://nodejs.org/docs/latest/api/errors.html#err_require_esm |
| `domain.create()` | process isolation, cluster restart | https://nodejs.org/docs/latest/api/deprecations.html#DEP0032 |
| `ERR_INVALID_CALLBACK` from timers | `ERR_INVALID_ARG_TYPE` | https://nodejs.org/docs/latest/api/timers.html#setimmediatecallback-args |
