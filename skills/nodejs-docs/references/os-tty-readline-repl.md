# node:os, node:tty, node:readline, node:repl

## Contents

- os
- tty
- readline
- repl
- Old patterns

## os

- Size worker pools with `os.availableParallelism()`, never `os.cpus().length`. https://nodejs.org/docs/latest/api/os.html#osavailableparallelism
- On Windows `os.loadavg()` is always `[0, 0, 0]`, `nice` is 0, and `os.userInfo()` returns `uid`/`gid` -1 and `shell` null. https://nodejs.org/docs/latest/api/os.html#osloadavg
- `os.homedir()` reads `HOME` / `USERPROFILE` first, so it can differ from `os.userInfo().homedir`. `userInfo()` throws if the user has no name or home. https://nodejs.org/docs/latest/api/os.html#osuserinfooptions
- `os.tmpdir()` honors `TMPDIR`, `TMP`, `TEMP` (POSIX) or `TEMP`, `TMP` (Windows), without a trailing slash. https://nodejs.org/docs/latest/api/os.html#ostmpdir
- Use `os.constants.priority.*` with `os.setPriority()`. On Windows `PRIORITY_HIGHEST` without elevation silently becomes `PRIORITY_HIGH`. https://nodejs.org/docs/latest/api/os.html#ossetprioritypid-priority
- `networkInterfaces()` reports `family` as the string `'IPv4'` / `'IPv6'`. https://nodejs.org/docs/latest/api/os.html

## tty

- Detect a terminal with `process.stdout.isTTY === true` (`Boolean(process.stdout.isTTY)` prints `false` when piped). https://nodejs.org/docs/latest/api/tty.html
- In raw mode Ctrl+C no longer raises `SIGINT`: handle exit yourself and restore with `setRawMode(false)`. `isRaw` is `false` at start even if the terminal is raw. On Windows raw mode needs write access to the console input. https://nodejs.org/docs/latest/api/tty.html#readstreamsetrawmodemode
- `setRawMode('io')` and the `rawMode` property are added in v26.8.0. `'io'` is binary-safe and unsupported on Windows. https://nodejs.org/docs/latest/api/tty.html#readstreamsetrawmodemode
- `getColorDepth()` relies on environment variables and can be wrong. Force with `FORCE_COLOR=0|1|2|3`, disable with `NO_COLOR`. https://nodejs.org/docs/latest/api/tty.html#writestreamgetcolordepthenv

## readline

- Default to `node:readline/promises` (stable since v24.0.0). The process does not exit until `rl.close()`. https://nodejs.org/docs/latest/api/readline.html#promises-api

```js
import * as readline from 'node:readline/promises';
import { stdin as input, stdout as output } from 'node:process';
const rl = readline.createInterface({ input, output });
const answer = await rl.question('What do you think of Node.js? ', { signal: AbortSignal.timeout(10_000) });
rl.close();
```
https://nodejs.org/docs/latest/api/readline.html#rlquestionquery-options

- Read a file by line with `crlfDelay: Infinity`: https://nodejs.org/docs/latest/api/readline.html#example-read-file-stream-line-by-line

```js
const rl = readline.createInterface({ input: fs.createReadStream('input.txt'), crlfDelay: Infinity });
for await (const line of rl) { /* ... */ }
```

- The interface consumes input from creation: any `await` between `createInterface()` and the loop can lose lines. Input stream errors are not forwarded. The `'line'` event is faster than `for await`. https://nodejs.org/docs/latest/api/readline.html#rlsymbolasynciterator
- `rl[Symbol.dispose]()` is an alias for `rl.close()`. https://nodejs.org/docs/latest/api/readline.html
- Without a `'SIGINT'` listener on the interface, Ctrl+C emits `'pause'`. https://nodejs.org/docs/latest/api/readline.html#event-sigint

## repl

- Several REPLs can share one `global` with `useGlobal: true` and separate `input`/`output` (stdin, Unix socket, TCP socket in the page's example). https://nodejs.org/docs/latest/api/repl.html#starting-multiple-repl-instances-in-the-same-process
- Programmatic REPLs have no history file. Call `replServer.setupHistory({ filePath, size })`. https://nodejs.org/docs/latest/api/repl.html#replserversetuphistoryhistoryconfig-callback
- `NODE_REPL_HISTORY=''` disables CLI history, `NODE_REPL_MODE=strict` enables strict mode. https://nodejs.org/docs/latest/api/repl.html#environment-variable-options
- `handleError` in `repl.start()` is added in v25.9.0. https://nodejs.org/docs/latest/api/repl.html#replstartoptions

## Old patterns

| Old pattern | Replacement | Source |
| :-- | :-- | :-- |
| `os.cpus().length` | `os.availableParallelism()` | https://nodejs.org/docs/latest/api/os.html#osavailableparallelism |
| `rl.question(q, callback)` | `await rl.question(q)` from `node:readline/promises` | https://nodejs.org/docs/latest/api/readline.html#promises-api |
| `repl.builtinModules` | `module.builtinModules` | https://nodejs.org/docs/latest/api/repl.html#replbuiltinmodules |
| `repl.REPLServer()` without `new` | `new` (End-of-Life) | https://nodejs.org/docs/latest/api/deprecations.html#DEP0185 |
