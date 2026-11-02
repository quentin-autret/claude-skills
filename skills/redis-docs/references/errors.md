# Error replies: meaning, cause and fix

The common Redis error prefixes, what triggers them and whether to retry, from the Redis error reference.

## Contents

- Reading errors
- Retry or fail fast
- Cluster errors
- Memory and persistence errors
- Replication errors
- Scripting and transaction errors
- Auth and ACL errors
- Command and argument errors
- Streams and restore errors

## Reading errors

- Match on the uppercase first word (`WRONGTYPE`, `MOVED`, `READONLY`), never on the text. Most Search errors start with a code such as `SEARCH_INDEX_NOT_FOUND`. Some JSON, probabilistic type and time series errors have no prefix at all (for example `Bad capacity`), and clients then use the first word.
  https://redis.io/docs/latest/develop/reference/errors/

## Retry or fail fast

| Error | Action |
| --- | --- |
| `LOADING`, `TRYAGAIN`, `BUSY` (temporary) | retry with backoff, bounded |
| `CLUSTERDOWN` | fix the cluster (restore the failed node), then resend |
| `MOVED`, `ASK` | follow the redirect (cluster-aware client) |
| `NOSCRIPT` | `SCRIPT LOAD`, then retry `EVALSHA` |
| `READONLY` | rediscover the primary |
| `WRONGTYPE`, syntax and arity errors, `NOPERM`, `WRONGPASS` | fail fast, fix the code or config |

https://redis.io/docs/latest/develop/reference/errors/common/loading/ and https://redis.io/docs/latest/develop/reference/errors/common/tryagain/ and https://redis.io/docs/latest/develop/clients/nodejs/error-handling/ and https://redis.io/docs/latest/develop/reference/errors/common/clusterdown/ and https://redis.io/docs/latest/develop/reference/errors/common/noscript/ and https://redis.io/docs/latest/develop/reference/errors/common/readonly/ and https://redis.io/docs/latest/develop/clients/error-handling/ and https://redis.io/docs/latest/develop/reference/errors/common/noperm/ and https://redis.io/docs/latest/develop/reference/errors/common/wrongpass/

## Cluster errors

- `MOVED`: the slot lives elsewhere. Update the slot map and resend. Cluster-aware clients (redis-py `RedisCluster`, `redis-cli -c`) do this. Plain clients surface it as an error.
  https://redis.io/docs/latest/develop/reference/errors/common/moved/
- `ASK`: the slot is migrating. Send `ASKING` then the command to the named node, once, without updating the map. Without `ASKING` the target answers `MOVED` back.
  https://redis.io/docs/latest/develop/reference/errors/common/ask/
- `CROSSSLOT`: a multi-key command, transaction or script spans slots. Put a hash tag around the shared part (`user:{42}:profile`, `user:{42}:cart`), or split per slot.
  https://redis.io/docs/latest/develop/reference/errors/common/crossslot/
- `TRYAGAIN`: a multi-key command during slot migration. redis-py gives up with `ClusterError: TTL exhausted.` A stuck migration shows as open slots in `redis-cli --cluster check`.
  https://redis.io/docs/latest/develop/reference/errors/common/tryagain/
- `CLUSTERDOWN`: by default any uncovered slot fails the whole cluster, even keys on healthy nodes. Give every primary a replica. `cluster-require-full-coverage no` keeps covered slots serving.
  https://redis.io/docs/latest/develop/reference/errors/common/clusterdown/

## Memory and persistence errors

- `OOM`: over `maxmemory` with `noeviction` (the default), writes fail while reads and deletes work. A `volatile-*` policy with no TTL keys frees nothing and also gives OOM. For a cache set `allkeys-lru`.
  https://redis.io/docs/latest/develop/reference/errors/common/oom/
- `MISCONF`: an RDB save or AOF write failed, so all writes are refused. Causes: full disk, unwritable dir, `fork()` failing on low memory. Check `INFO persistence` (`rdb_last_bgsave_status`, `aof_last_write_status`) and `CONFIG GET dir`.
  https://redis.io/docs/latest/develop/reference/errors/common/misconf/
- Recovery: RDB needs one successful `BGSAVE`, AOF retries every second. `stop-writes-on-bgsave-error no` only hides the failure and does not apply to AOF.
  https://redis.io/docs/latest/develop/reference/errors/common/misconf/
- `LOADING`: during startup or a replica full sync, most commands fail, PING included. `INFO` works (`loading:1`, `loading_eta_seconds`).
  https://redis.io/docs/latest/develop/reference/errors/common/loading/
- `ERR max number of clients reached`: `maxclients` (default 10,000) is full. Reuse one client or pool per app, and set `timeout` so idle connections close. Raising `maxclients` also needs a higher OS file descriptor limit.
  https://redis.io/docs/latest/develop/reference/errors/common/err-max-clients/

## Replication errors

- `READONLY`: a write reached a replica, often a demoted primary after failover. Use Sentinel-aware clients. Never set `replica-read-only no` as a workaround: data diverges.
  https://redis.io/docs/latest/develop/reference/errors/common/readonly/
- `NOREPLICAS`: fewer than `min-replicas-to-write` replicas reported within `min-replicas-max-lag`. Reads still work.
  https://redis.io/docs/latest/develop/reference/errors/common/noreplicas/
- `MASTERDOWN`: a replica with `replica-serve-stale-data no` lost its primary link. The default `yes` serves possibly stale data. A password-protected primary needs `masterauth` (plus `masteruser` with ACL users).
  https://redis.io/docs/latest/develop/reference/errors/common/masterdown/

## Scripting and transaction errors

- `BUSY`: a script or function runs past `busy-reply-threshold` (5 s by default), and almost every command (PING included) gets `BUSY`. Only `SCRIPT KILL` (`FUNCTION KILL` for a function) or `SHUTDOWN NOSAVE` work. A long module command gives a similar `BUSY` error. A kill fails with `UNKILLABLE` once the script wrote.
  https://redis.io/docs/latest/develop/reference/errors/common/busy/
- `NOSCRIPT`: the script cache was emptied (restart, failover, `SCRIPT FLUSH`). Functions avoid it because they are persisted and replicated.
  https://redis.io/docs/latest/develop/reference/errors/common/noscript/
- Scripts with a `#!lua` shebang and functions are rejected up front under OOM unless flagged `allow-oom` or `no-writes`, and on replicas unless `no-writes`.
  https://redis.io/docs/latest/develop/reference/errors/common/oom/ and https://redis.io/docs/latest/develop/reference/errors/common/readonly/
- `EXECABORT`: a command failed while queueing, so EXEC discards everything. A missing EXEC permission gives `EXECABORT ... NOPERM`.
  https://redis.io/docs/latest/develop/reference/errors/common/execabort/

## Auth and ACL errors

- `NOAUTH`: authentication is required and almost every command fails, PING included.
  https://redis.io/docs/latest/develop/reference/errors/common/noauth/
- `WRONGPASS`: identical for a wrong password, an unknown user and a disabled user (`ACL SETUSER u on` re-enables). Usual causes: another environment's password, a trailing newline, a missing username (AUTH then checks `default`).
  https://redis.io/docs/latest/develop/reference/errors/common/wrongpass/
- `NOPERM`: the authenticated user lacks the permission to run the command or use the key. Diagnose with `ACL LOG` and `ACL WHOAMI`.
  https://redis.io/docs/latest/develop/reference/errors/common/noperm/
- `DENIED`: protected mode (on by default) accepts only loopback when the default user has no password. Some clients only report "Connection closed by server". Set a password or ACL users rather than disabling protected mode.
  https://redis.io/docs/latest/develop/reference/errors/common/denied/

## Command and argument errors

- `WRONGTYPE`: the key holds another type. Check with `TYPE key`. JSON says `Existing key has wrong Redis type`. A leftover key from an old app version is a common cause.
  https://redis.io/docs/latest/develop/reference/errors/common/wrongtype/
- `ERR value is not an integer or out of range`: `INCR` family on text or decimals (use `INCRBYFLOAT`), or numbers with units (`10s`). An increment past the signed 64-bit maximum gives `ERR increment or decrement would overflow` instead.
  https://redis.io/docs/latest/develop/reference/errors/common/err-not-an-integer/
- `ERR syntax error` never names the bad option. Look for exclusive options (`SET k v NX XX`), a misplaced option (`ZADD myset 1 "a" NX`), or an option newer than the server.
  https://redis.io/docs/latest/develop/reference/errors/common/err-syntax-error/
- `ERR unknown command`: typo, command newer than the server (`INFO server`), module not loaded (`MODULE LIST`), or renamed through `rename-command`.
  https://redis.io/docs/latest/develop/reference/errors/common/err-unknown-command/
- Wrong number of arguments: a variadic command with zero keys, or `AUTH user pass` on a server older than Redis 6.
  https://redis.io/docs/latest/develop/reference/errors/common/err-wrong-number-of-arguments/

## Streams and restore errors

- `BUSYGROUP`: the group exists. Often harmless when apps create the group at every start: catch it and carry on. Move the read position with `XGROUP SETID`, not by recreating: `XGROUP DESTROY` drops the pending list.
  https://redis.io/docs/latest/develop/reference/errors/common/busygroup/
- `NOGROUP`: the group or the stream is missing, including a deleted or expired stream key. Names are case sensitive.
  https://redis.io/docs/latest/develop/reference/errors/common/nogroup/
- `BUSYKEY`: `RESTORE` does not overwrite without `REPLACE`.
  https://redis.io/docs/latest/develop/reference/errors/common/busykey/
