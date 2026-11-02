# Lua scripts and Redis Functions

When to use Functions or EVAL, key declaration, the script cache, flags, blocking and the Lua to RESP conversion traps in Redis Open Source 8.10.

## Contents

- Functions or EVAL
- Declaring keys and arguments
- Script cache and NOSCRIPT
- Flags and read-only scripts
- Blocking, timeouts and memory
- Cluster
- Lua sandbox and type conversion
- Old patterns

## Functions or EVAL

- Default on Redis 7+: Functions (`FUNCTION LOAD` + `FCALL`). They are named, persisted to AOF, replicated with the data, and callable by every client. Loading them is a deployment task, like loading a module.
  https://redis.io/docs/latest/develop/programmability/
- Escape hatch: `EVAL` for light scripts that belong to the application. Every app instance then holds the source and must be able to reload it at any time.
  https://redis.io/docs/latest/develop/programmability/
- Functions solve EVAL's problems: every client needing every script, cached scripts failing inside transactions, meaningless SHA1s in MONITOR, scripts unable to call other scripts.
  https://redis.io/docs/latest/develop/programmability/functions-intro/
- A library loads as a whole: update with `FUNCTION LOAD REPLACE`. The payload starts with `#!lua name=<lib>`. Loading fails if no function is registered, if the library exists without REPLACE, or if a function name exists in another library.

```
cat mylib.lua | redis-cli -x FUNCTION LOAD REPLACE
FCALL myfunc 0 hello
```
  https://redis.io/docs/latest/commands/function-load/ and https://redis.io/docs/latest/develop/programmability/functions-intro/
- In functions, keys and args arrive as the callback's parameters `(keys, args)`. The `KEYS` and `ARGV` globals exist only in EVAL scripts. Functions cannot use the Lua debugger.
  https://redis.io/docs/latest/develop/programmability/lua-api/ and https://redis.io/docs/latest/develop/programmability/functions-intro/

## Declaring keys and arguments

- Pass every key the script touches as a key argument, everything else as an argument. Never build key names in the script or read them from stored data: it breaks cluster routing and correctness.

```
EVAL "return { KEYS[1], KEYS[2], ARGV[1], ARGV[2], ARGV[3] }" 2 key1 key2 arg1 arg2 arg3
```
  https://redis.io/docs/latest/commands/eval/ and https://redis.io/docs/latest/develop/programmability/eval-intro/
- Never embed values in the script text. Each variant becomes a new cached script and eats memory. Since 7.4, EVAL scripts are evicted LRU when the cache grows (`evicted_scripts` in INFO).
  https://redis.io/docs/latest/commands/eval/ and https://redis.io/docs/latest/develop/programmability/eval-intro/
- Never build script bodies from untrusted strings. Normal arguments are binary safe and carry no injection risk.
  https://redis.io/docs/latest/operate/oss_and_stack/management/security/

## Script cache and NOSCRIPT

- The script cache is never persisted: restart, replica promotion and `SCRIPT FLUSH` empty it, so `EVALSHA` can return `NOSCRIPT` after any failover.
  https://redis.io/docs/latest/develop/programmability/eval-intro/ and https://redis.io/docs/latest/develop/reference/errors/common/noscript/
- On `NOSCRIPT`, run `SCRIPT LOAD` and retry `EVALSHA`. Most clients do this (redis-py `register_script()` objects reload themselves).
  https://redis.io/docs/latest/develop/reference/errors/common/noscript/
- A NOSCRIPT inside a pipeline cannot be handled: `SCRIPT LOAD` first, or send plain parameterized `EVAL` in the pipeline.
  https://redis.io/docs/latest/develop/programmability/eval-intro/

## Flags and read-only scripts

- Flags: `no-writes`, `allow-oom`, `allow-stale`, `no-cluster`, `allow-cross-slot-keys`. EVAL scripts declare them in a shebang, functions in `register_function`.

```
#!lua flags=no-writes,allow-stale
local x = redis.call('get','x')
return x
```
  https://redis.io/docs/latest/develop/programmability/eval-intro/ and https://redis.io/docs/latest/develop/programmability/lua-api/
- A shebang changes the defaults even with no flags: a script starting with `#!` can no longer touch keys in different slots, and is rejected up front on OOM or on a read-only replica unless flagged.
  https://redis.io/docs/latest/develop/programmability/eval-intro/ and https://redis.io/docs/latest/develop/reference/errors/common/oom/
- A function is assumed to write unless flagged `no-writes`: `FCALL_RO` refuses it and replicas deny it.
  https://redis.io/docs/latest/develop/programmability/functions-intro/
- Read-only scripts never fail with OOM, always run on replicas, can always be killed and keep running during write pauses. `PUBLISH`, `SPUBLISH` and `PFCOUNT` count as writes (7.0.1).
  https://redis.io/docs/latest/develop/programmability/
- Default for read-only logic: the normal commands with `no-writes`. Use `EVAL_RO`/`EVALSHA_RO`/`FCALL_RO` when ACLs must limit a user to read-only scripts or a client routes them to replicas.
  https://redis.io/docs/latest/develop/programmability/

## Blocking, timeouts and memory

- A running script blocks every client. Past `busy-reply-threshold` (default 5 s) Redis does not kill it: others get `BUSY`, even for PING.
  https://redis.io/docs/latest/develop/programmability/ and https://redis.io/docs/latest/develop/reference/errors/common/busy/
- `SCRIPT KILL`/`FUNCTION KILL` work only before the first write. After a write the only exit is `SHUTDOWN NOSAVE`, which loses unsaved data. Split big jobs into short calls.
  https://redis.io/docs/latest/develop/reference/errors/common/busy/
- Over `maxmemory`, the first write that needs memory aborts the script. If the first write frees memory, the rest runs to keep atomicity and can exceed the limit.
  https://redis.io/docs/latest/develop/programmability/eval-intro/
- Handle a command error inside the script with `redis.pcall`, which returns an error table. `redis.call` raises it to the client.
  https://redis.io/docs/latest/develop/programmability/lua-api/

## Cluster

- Functions are NOT propagated between cluster nodes, only to replicas. Load them on every primary; `redis-cli --cluster add-node` copies them to a new node.

```
redis-cli --cluster-only-masters --cluster call host:port FUNCTION LOAD ...
```
  https://redis.io/docs/latest/develop/programmability/functions-intro/
- Cache-only instances must preload functions before accepting clients after a restart: `redis-cli --functions-rdb` extracts them into an RDB.
  https://redis.io/docs/latest/develop/programmability/functions-intro/
- `allow-cross-slot-keys` is discouraged, and declared keys must still hash to one slot.
  https://redis.io/docs/latest/develop/programmability/lua-api/

## Lua sandbox and type conversion

- No global variables (creating or reading an undefined one errors), no `require`, no file, network or system calls. `os` exposes only `os.clock()` (7.4). Keep state in the keyspace, not in Lua globals.
  https://redis.io/docs/latest/develop/programmability/lua-api/
- Lua to RESP: a number becomes an integer (decimals dropped, so return floats as strings), an array is cut at the first nil, associative entries are dropped, `false` becomes a null bulk in RESP2.
  https://redis.io/docs/latest/develop/programmability/lua-api/
- RESP to Lua: a RESP2 null (bulk or multi-bulk) becomes `false` (after `redis.setresp(3)` a RESP3 null becomes `nil`), a status becomes `{ok=...}`, an error `{err=...}`. After `redis.setresp(3)` a map becomes `{map={...}}`, a double `{double=n}`.
  https://redis.io/docs/latest/develop/programmability/lua-api/
- `redis.set_repl` can break replica and AOF consistency and is unavailable in functions.
  https://redis.io/docs/latest/develop/programmability/lua-api/
- Debug with `redis-cli --ldb --eval script.lua key , arg` on a dev server. `--ldb-sync-mode` blocks the whole server.
  https://redis.io/docs/latest/develop/programmability/lua-debugging/

## Old patterns

| Old | Current | Source |
| --- | --- | --- |
| EVAL/EVALSHA for server-side app logic | Functions, `FUNCTION LOAD` + `FCALL` (7.0) | https://redis.io/docs/latest/develop/programmability/functions-intro/ |
| Verbatim script replication, `redis.replicate_commands()` | Effects replication only (7.0), call not needed | https://redis.io/docs/latest/develop/programmability/eval-intro/ |
| Random seed passed as argument, sorted SMEMBERS rules | Not needed with effects replication | https://redis.io/docs/latest/develop/programmability/eval-intro/ |
| `redis.REPL_SLAVE` | `redis.REPL_REPLICA` | https://redis.io/docs/latest/develop/programmability/lua-api/ |
| Lua `print()` | Unsupported since 7.2, the docs log with `redis.log(redis.LOG_WARNING, msg)` | https://redis.io/docs/latest/develop/whats-new/7-2/ and https://redis.io/docs/latest/develop/programmability/lua-api/ |
