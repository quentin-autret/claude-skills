# Client connections, URLs, pooling, caching and retries

Language-neutral client rules: handshake and RESP3, portable connection URLs, pools versus multiplexing, client-side caching, server-side client limits, error handling and Sentinel discovery. Node.js, Python and Rust specifics live in their own references.

## Contents

- Choosing a client
- Handshake and protocol
- Connection URLs
- Pools and multiplexing
- Client-side caching
- Server-side client limits
- Error handling and retries
- Sentinel-aware clients
- Old patterns

## Choosing a client

- Official clients include redis-py, node-redis, ioredis, Jedis, Lettuce, go-redis, StackExchange.Redis, NRedisStack. redis-rs and Predis are "best effort". node-redis is the recommended JavaScript client, ioredis is older and lacks some newer features and optimizations.
  https://redis.io/docs/latest/develop/clients/
- Use the latest `major.minor` of a client. Only critical bugs and CVEs are backported to older minors.
  https://redis.io/docs/latest/develop/clients/version-support/

## Handshake and protocol

- Open each connection with `HELLO`, which negotiates the protocol, authenticates and names the client in one call. `-NOPROTO` means retry a lower version. A failed `AUTH` leaves the connection on RESP2.

```
HELLO 3 AUTH default <password> SETNAME myapp
```
  https://redis.io/docs/latest/develop/reference/protocol-spec/ and https://redis.io/docs/latest/commands/hello/
- Connections start in RESP2. RESP3 adds maps, sets, doubles and push messages (`HGETALL` returns a map). Future features may require RESP3, and reply types can differ per protocol.
  https://redis.io/docs/latest/develop/reference/protocol-spec/ and https://redis.io/docs/latest/commands/hello/
- Name connections (`CLIENT SETNAME`): names show in `SLOWLOG`. Libraries send `CLIENT SETINFO LIB-NAME`/`LIB-VER` after auth and ignore failures on old servers.
  https://redis.io/docs/latest/develop/reference/clients/ and https://redis.io/docs/latest/commands/client-setinfo/

## Connection URLs

Only `redis[s]://[[user][:password]@][host][:port][/db]` is portable. Query parameters are implementation defined.
https://redis.io/docs/latest/develop/clients/connection-urls/

- Portable form: lowercase scheme, explicit port, explicit user, percent-encoded password, db in the path.

```
rediss://default:p%40ss%3Aw%2Brd@host:6379/0
```
  https://redis.io/docs/latest/develop/clients/connection-urls/
- `REDIS://` is rejected by Lettuce, Predis and sync redis-py, and ioredis enables TLS only for lowercase `rediss://`. Jedis rejects a URL without port.
  https://redis.io/docs/latest/develop/clients/connection-urls/
- `redis://secret@host` (no colon) is a username in redis-py, node-redis, ioredis, go-redis, redis-rs and Predis, a password in Lettuce and redis-rb, and an error in Jedis. For a password only, write `redis://:secret@host`.
  https://redis.io/docs/latest/develop/clients/connection-urls/
- Unknown query parameters are ignored by node-redis, Jedis, Lettuce, redis-rs and redis-rb, passed as options by redis-py, ioredis and Predis, and an error in go-redis. Never set one option in both the URL and the client options. Keep URLs out of logs, they hold the password.
  https://redis.io/docs/latest/develop/clients/connection-urls/

## Pools and multiplexing

- Reuse connections through a pool or a multiplexer, never one connection per request. Total connections = pool size x app instances, so shrink pools when instances are many.
  https://redis.io/docs/latest/develop/clients/pools-and-muxing/ and https://redis.io/docs/latest/develop/reference/errors/common/err-max-clients/
- A multiplexer cannot run blocking pops (`BLPOP`): they stall the connection for every caller. Use a pool or a dedicated connection for them.
  https://redis.io/docs/latest/develop/clients/pools-and-muxing/
- Pool exhaustion is a configuration or concurrency issue: monitor usage, then raise the pool or lower concurrency.
  https://redis.io/docs/latest/develop/clients/error-handling/

## Client-side caching

- Supported from redis-py 5.1.0, node-redis 5.1.0, Jedis 5.2.0, go-redis 9.22.0. Supporting `CLIENT TRACKING` does not mean a client implements the cache.
  https://redis.io/docs/latest/develop/clients/client-side-caching/
- The cache requires RESP3 (`protocol=3` in redis-py, `Protocol: 3` in go-redis, `RESP: 3` in node-redis).
  https://redis.io/docs/latest/develop/clients/redis-py/connect/ and https://redis.io/docs/latest/develop/clients/nodejs/connect/
- When any connection of the client (pool members included) disconnects, the whole local cache is flushed.
  https://redis.io/docs/latest/develop/clients/client-side-caching/
- Not cached: probabilistic and time series commands, non-deterministic commands, `FT.*`. Fields, ranges and differently ordered `MGET`s are separate entries.
  https://redis.io/docs/latest/develop/clients/client-side-caching/
- Read frequently updated keys (counters, scoreboards) over a separate non-caching connection: invalidations would cost more than the cache saves.
  https://redis.io/docs/latest/develop/clients/client-side-caching/
- Hand-built tracking on RESP2 uses two connections, and an invalidation can beat a slow reply: put a placeholder before sending, skip storing if it was invalidated. RESP3 on one connection has no race.
  https://redis.io/docs/latest/develop/reference/client-side-caching/
- Tracking ignores database numbers. Default mode costs server memory per tracked key and client. `BCAST` costs CPU per prefix, and its prefixes cannot overlap.
  https://redis.io/docs/latest/develop/reference/client-side-caching/

## Server-side client limits

- `maxclients` defaults to 10,000, lowered if the file descriptor limit is smaller. Extra clients get an error and are closed. In a cluster, bus connections count too.
  https://redis.io/docs/latest/develop/reference/clients/ and https://redis.io/docs/latest/develop/reference/errors/common/err-max-clients/
- Idle connections are never closed by default. `timeout` closes them but not Pub/Sub clients.
  https://redis.io/docs/latest/develop/reference/clients/
- Client eviction is off by default: set `maxmemory-clients` (for example `5%`) in production, and `CLIENT NO-EVICT on` for monitoring connections (7.0).
  https://redis.io/docs/latest/develop/reference/clients/
- Parse `CLIENT LIST` and `INFO` tolerantly: fields come and go between versions.
  https://redis.io/docs/latest/commands/client-list/ and https://redis.io/docs/latest/commands/info/

## Error handling and retries

- Catch specific error types. Retry timeouts, connection errors and `LOADING`, `BUSY`, `TRYAGAIN` with backoff. Fail fast on auth and command errors such as `WRONGTYPE` or syntax errors.
  https://redis.io/docs/latest/develop/clients/error-handling/ and https://redis.io/docs/latest/develop/clients/nodejs/error-handling/
- Prefer the client's built-in retry settings to hand-written loops.
  https://redis.io/docs/latest/develop/clients/error-handling/
- Cache reads fall back to the source database on connection errors. Cache writes and metrics log and continue. An entry that fails to deserialize is deleted and refetched.
  https://redis.io/docs/latest/develop/clients/error-handling/
- Check the uppercase prefix of an error (`MOVED`, `READONLY`) rather than reading its description. Some module errors have no prefix, and clients then treat the first word as the prefix.
  https://redis.io/docs/latest/develop/reference/errors/
- Geographic failover clients use a circuit breaker: retry transient failures a few times before counting a failure.
  https://redis.io/docs/latest/develop/clients/failover/

## Sentinel-aware clients

- Ask Sentinel for the primary address on every reconnect, then confirm with `ROLE`. If the role is wrong, wait a few hundred milliseconds and restart from the first Sentinel step. Pub/Sub announcements do not replace this check.

```
SENTINEL get-master-addr-by-name master-name
```
  https://redis.io/docs/latest/develop/reference/sentinel-clients/
- Sentinels and data nodes have separate credentials in redis-py (`sentinel_kwargs`) and node-redis (`sentinelClientOptions`, `nodeClientOptions`).
  https://redis.io/docs/latest/develop/clients/redis-py/connect/ and https://redis.io/docs/latest/develop/clients/nodejs/connect/

## Old patterns

| Old | Current | Source |
| --- | --- | --- |
| `AUTH` then other setup commands | `HELLO 3 AUTH user pass SETNAME name` (6.0) | https://redis.io/docs/latest/commands/hello/ |
| Parsing `INFO replication` for the role | `ROLE` | https://redis.io/docs/latest/develop/reference/sentinel-clients/ |
| Hand-rolled invalidation over `PUBLISH` | Server-assisted tracking, `CLIENT TRACKING` (6.0) | https://redis.io/docs/latest/develop/reference/client-side-caching/ |
| ioredis for new Node.js code | node-redis | https://redis.io/docs/latest/develop/clients/ |
