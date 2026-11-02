# node:util and console

## Contents

- promisify and callbackify
- debounce and throttle (added in v26.10.0)
- Terminal styling
- Deprecation and debug logging
- Other helpers
- console
- Old patterns

## promisify and callbackify

- Promisified methods lose `this`: bind them. https://nodejs.org/docs/latest/api/util.html#utilpromisifyoriginal

```js
const bar = promisify(foo.bar).bind(foo);
```

- Never promisify a function that already returns a promise: its result is ignored and rejections go unhandled. https://nodejs.org/docs/latest/api/deprecations.html#DEP0174
- Customize with the `util.promisify.custom` symbol. https://nodejs.org/docs/latest/api/util.html#custom-promisified-functions
- With `callbackify`, a throw inside the callback becomes `'uncaughtException'`. A rejection with a falsy value arrives as an `Error` with the value in `err.reason`. https://nodejs.org/docs/latest/api/util.html#utilcallbackifyoriginal

## debounce and throttle (added in v26.10.0)

Added in v26.10.0, so check the target runtime version. https://nodejs.org/docs/latest/api/util.html#utildebouncefn-wait-options

- Both return functions whose calls resolve with `fn`'s result, with `cancel`, `pending` and `ref`/`unref` (`unref()` lets the process exit while a call waits). https://nodejs.org/docs/latest/api/util.html#utildebouncefn-wait-options

```js
const onInactivity = debounce(() => console.log('No activity for 5 seconds'), 5_000).unref();
const request = throttle(async (id) => (await fetch(`https://example.com/items/${id}`)).json(), 2, 1_000);
```

- Debounce: superseded calls resolve with the last call's result. `rejectOnCancel: true` rejects them with `AbortError`. https://nodejs.org/docs/latest/api/util.html#utildebouncefn-wait-options
- Throttle queues extra calls by default (`overflow: 'queue'`). `'drop'` rejects at once with `ERR_THROTTLED` (marked handled). The default window can burst at boundaries: `strict: true` guarantees at most `limit` starts in any rolling interval. https://nodejs.org/docs/latest/api/util.html#utilthrottlefn-limit-interval-options

## Terminal styling

- `util.styleText()` (stable) respects `NO_COLOR`, `FORCE_COLOR` and TTY detection against `process.stdout` by default. Pass `{ stream: process.stderr }` when printing to stderr. Hex colors are added in v26.1.0. https://nodejs.org/docs/latest/api/util.html#utilstyletextformat-text-options

```js
console.error(styleText('red', 'Error! Error!', { stream: process.stderr }));
```

## Deprecation and debug logging

- `util.deprecate(fn, msg, code)` warns once per `code`, shared across wrappers. `--no-deprecation` silences it and `--throw-deprecation` makes it throw. `modifyPrototype: false` (added v24.12.0) leaves the prototype alone. https://nodejs.org/docs/latest/api/util.html#utildeprecatefn-msg-code-options
- `util.debuglog(section)` logs only when `NODE_DEBUG` includes the section (wildcards allowed). Pass a callback to get a logger that skips the check: https://nodejs.org/docs/latest/api/util.html#utildebuglogsection-callback

```js
let log = debuglog('internals', (debug) => { log = debug; });
```

## Other helpers

- `util.parseArgs`, `util.parseEnv`, `util.MIMEType`, `util.isPartialDeepStrictEqual`, `util.getSystemErrorMessage` and `util.convertProcessSignalToExitCode` exist in `node:util`. Check the page for the version that added each. https://nodejs.org/docs/latest/api/util.html
- `util.diff(actual, expected)` (experimental) returns `[op, value]` entries. https://nodejs.org/docs/latest/api/util.html#utildiffactual-expected
- Customize inspection with the `util.inspect.custom` symbol. https://nodejs.org/docs/latest/api/util.html#utilinspectcustom

## console

- The global `console` is neither consistently synchronous nor asynchronous: it depends on the platform and on what stdout is. https://nodejs.org/docs/latest/api/console.html
- Build a file logger with `new Console({ stdout, stderr })`. Write errors on the streams are ignored by default (`ignoreErrors: true`). https://nodejs.org/docs/latest/api/console.html#new-consoleoptions

```js
const logger = new Console({ stdout: createWriteStream('./stdout.log'), stderr: createWriteStream('./stderr.log') });
```

- `console.profile()`, `profileEnd()` and `timeStamp()` do nothing without an attached inspector. https://nodejs.org/docs/latest/api/console.html#inspector-only-methods

## Old patterns

| Old pattern | Replacement | Source |
| :-- | :-- | :-- |
| `util.inherits(Ctor, Super)` | `class Ctor extends Super` | https://nodejs.org/docs/latest/api/util.html#utilinheritsconstructor-superconstructor |
| `util._extend(t, s)` | `Object.assign(t, s)` | https://nodejs.org/docs/latest/api/deprecations.html#DEP0060 |
| `util.isArray(x)` | `Array.isArray(x)` | https://nodejs.org/docs/latest/api/deprecations.html#DEP0044 |
| other `util.isXxx()` | removed (End-of-Life) | https://nodejs.org/docs/latest/api/deprecations.html#DEP0045 |
| `util.types.isNativeError(x)` | `Error.isError(x)` | https://nodejs.org/docs/latest/api/util.html#utiltypesisnativeerrorvalue |
| `util.toUSVString(s)` | `s.toWellFormed()` | https://nodejs.org/docs/latest/api/deprecations.html#DEP0175 |
| `require('node:sys')` | `node:util` | https://nodejs.org/docs/latest/api/deprecations.html#DEP0025 |
