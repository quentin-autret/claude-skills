# redis-py and redis-rs

Defaults and traps of redis-py (sync and asyncio) and redis-rs: timeouts, retries, pools, pipelines, WATCH, Sentinel, caching, failover.

## Contents

- redis-py basics
- redis-py timeouts and retries
- redis-py asyncio
- redis-py pipelines and WATCH
- redis-py Sentinel, caching and failover
- redis-py search and vectors
- redis-rs connections
- redis-rs timeouts and blocking commands
- redis-rs pipelines, errors and features
- Old patterns

## redis-py basics

- Replies are bytes unless `decode_responses=True`. In a URL, `?decode_responses=False` is the string "False", which is truthy: pass booleans as keyword arguments.
  https://redis.io/docs/latest/develop/clients/redis-py/connect/ and https://redis.io/docs/latest/develop/clients/connection-urls/
- Production uses a pool. The documented form shares one pool through `from_pool`:

```python
pool = redis.ConnectionPool().from_url("redis://localhost")
r1 = redis.Redis().from_pool(pool)
```
  https://redis.io/docs/latest/develop/clients/redis-py/connect/
- Iterate with `scan_iter(match=..., count=..., _type=...)` and `hscan_iter(key, no_values=True)` for field names only.
  https://redis.io/docs/latest/develop/clients/redis-py/scaniter/
- Exceptions from `redis.exceptions`: `ConnectionError` and `TimeoutError` are recoverable, `ResponseError` is a bug to fix, `DataError` is only partly recoverable (log, invalidate the cache, fetch fresh data).
  https://redis.io/docs/latest/develop/clients/redis-py/error-handling/
- In `RedisCluster`, `mget_nonatomic()`/`mset_nonatomic()` split cross-slot calls (no longer atomic). `RedisCluster.from_url()` accepts db 0 only.
  https://redis.io/docs/latest/develop/reference/errors/common/crossslot/ and https://redis.io/docs/latest/develop/clients/connection-urls/

## redis-py timeouts and retries

- Connect and command timeouts default to 10 s. Too short retries commands that would have succeeded, too long hangs on a reply that never comes.
  https://redis.io/docs/latest/develop/clients/redis-py/produsage/
- Since 6.0.0 the default retry is 3 attempts with exponential backoff and jitter, on `ConnectionError` and `TimeoutError` only. Extend with `retry_on_error`. `ExponentialBackoff()` alone has no jitter.

```python
r = Redis(retry=Retry(ExponentialBackoff(), 8), retry_on_error=[BusyLoadingError],
          health_check_interval=3, socket_connect_timeout=15, socket_timeout=5)
```
  https://redis.io/docs/latest/develop/clients/redis-py/produsage/
- Cluster retries always cover `TimeoutError`, `ConnectionError`, `ClusterDownError`, and the list is not configurable.
  https://redis.io/docs/latest/develop/clients/redis-py/produsage/
- `health_check_interval` sends a PING before using a connection idle longer than the interval.
  https://redis.io/docs/latest/develop/clients/redis-py/produsage/

## redis-py asyncio

- Create ONE `redis.asyncio.Redis` at startup, share it across tasks, close it at shutdown (`await r.aclose()` or `async with`). It owns its pool and connects lazily.
  https://redis.io/docs/latest/develop/clients/redis-py/async/
- Never pass one `ConnectionPool` to several `Redis` objects: closing any of them closes the pool under the others.
  https://redis.io/docs/latest/develop/clients/redis-py/async/
- Size `max_connections` to peak concurrency. `BlockingConnectionPool` waits instead of raising on exhaustion.
  https://redis.io/docs/latest/develop/clients/redis-py/async/
- Pipelines and `PubSub` objects are not task safe: one per task. A forgotten `await` returns a coroutine, the most common mistake.
  https://redis.io/docs/latest/develop/clients/redis-py/async/
- A cancelled command (`asyncio.wait_for`) disconnects that connection to avoid reply misalignment.
  https://redis.io/docs/latest/develop/clients/redis-py/async/

## redis-py pipelines and WATCH

- `pipeline()` is a transaction by default. Pass `transaction=False` for a plain pipeline.
  https://redis.io/docs/latest/develop/clients/redis-py/transpipe/
- After `pipe.watch(...)` commands run immediately until `pipe.multi()`. `r.transaction(func, *keys)` retries on conflict, but `func` must call `pipe.multi()` itself.

```python
def watched_sequence(pipe):
    current = pipe.get("shellpath")
    pipe.multi()
    pipe.set("shellpath", current + ":/usr/mycmds/")
r.transaction(watched_sequence, "shellpath")
```
  https://redis.io/docs/latest/develop/clients/redis-py/transpipe/
- In async pipelines, buffered commands are not awaited. Only `watch`, reads before `multi()` and `execute()` are.
  https://redis.io/docs/latest/develop/clients/redis-py/async/

## redis-py Sentinel, caching and failover

- Sentinel credentials go in `sentinel_kwargs`, data-node credentials in `master_for()`. With `sentinel_kwargs`, Sentinel connections no longer inherit `socket_timeout`: repeat it there.
  https://redis.io/docs/latest/develop/clients/redis-py/connect/
- redis-py ignores failover announcements: it keeps using the old primary until it rejects writes, and writes accepted in between are lost. `slave_for()` reads can be stale.
  https://redis.io/docs/latest/develop/clients/redis-py/connect/
- Client-side caching: `Redis(protocol=3, cache_config=CacheConfig())`, redis-py 5.1.0+ and Redis 7.4+. Opt-in, opt-out and broadcast modes are not implemented.
  https://redis.io/docs/latest/develop/clients/redis-py/connect/
- `MultiDBClient` (geographic failover): after all attempts it raises `NoValidDatabaseException` and does not recover by itself, so rebuild the client. Use it for publishers and subscribers alike. `add_database` skips the initial health check by default.
  https://redis.io/docs/latest/develop/clients/redis-py/failover/

## redis-py search and vectors

- From redis-py 6.0.0, `ft().search()` requests dialect 2 explicitly, overriding the server default. `dropindex()` on a missing index and creating an index under an existing name raise `ResponseError`.
  https://redis.io/docs/latest/develop/clients/redis-py/queryjson/, https://redis.io/docs/latest/develop/clients/redis-py/vecsearch/ and https://redis.io/docs/latest/develop/clients/redis-py/produsage/
- Hash vectors are bytes (`.astype(np.float32).tobytes()`), JSON vectors are lists (`.tolist()`), and the query parameter is bytes in both cases.
  https://redis.io/docs/latest/develop/clients/redis-py/vecsearch/
- `himport_prepare()` then `himport_set()` in a `transaction=False` pipeline (Redis Open Source 8.10 and redis-py 8.1.0+, experimental, replaces existing keys).
  https://redis.io/docs/latest/develop/clients/redis-py/queryjson/

## redis-rs connections

- redis-rs is third party, not supported by Redis. Async needs the `tokio-comp` or `smol-comp` feature.
  https://redis.io/docs/latest/develop/clients/rust/
- Async default: `get_multiplexed_async_connection()`. Clones share one connection, so no pool is needed.
  https://redis.io/docs/latest/develop/clients/rust/connect/
- A `MultiplexedConnection` never reconnects. `ConnectionManager` (feature `connection-manager`) reconnects in the background, but the command that met the disconnect still fails: retry it.
  https://redis.io/docs/latest/develop/clients/rust/connect/
- Sync pools use r2d2 (`max_size` default 10, all opened at build).
  https://redis.io/docs/latest/develop/clients/rust/connect/
- Protocol defaults to RESP2. RESP3 with `?protocol=resp3`. Client-side caching (experimental) needs RESP3, async connections, the `cache-aio` feature and Redis 7.4+.
  https://redis.io/docs/latest/develop/clients/rust/connect/
- Sentinel connections do not reconnect and ignore failover announcements: call `get_connection()` again on error.
  https://redis.io/docs/latest/develop/clients/rust/connect/
- rustls picks a crypto provider by itself only when your dependencies enable exactly one, otherwise connecting panics unless you install one first (`rustls::crypto::aws_lc_rs::default_provider().install_default()`). The `#insecure` URL fragment on a `rediss` URL skips certificate verification.
  https://redis.io/docs/latest/develop/clients/rust/connect/ and https://redis.io/docs/latest/develop/clients/connection-urls/

## redis-rs timeouts and blocking commands

- Async defaults: 1 s connect, 500 ms response. The response timeout also hits blocking commands, and while a BLPOP blocks, other commands on that multiplexed connection time out too. Send blocking commands on their own connection with a longer or no response timeout.

```rust
let config = AsyncConnectionConfig::new().set_response_timeout(None);
let mut con = client.get_multiplexed_async_connection_with_config(&config).await?;
```
  https://redis.io/docs/latest/develop/clients/rust/connect/
- Sync connections have NO timeouts by default: set `set_read_timeout()` and `set_write_timeout()`.
  https://redis.io/docs/latest/develop/clients/rust/connect/

## redis-rs pipelines, errors and features

- `exec()` discards results, `query()` returns them, `.ignore()` drops one, `atomic()` makes a transaction. `redis::transaction(&mut con, &[keys], |con, pipe| ...)` retries until the watched keys are untouched: read through `con`, write through `pipe`.
  https://redis.io/docs/latest/develop/clients/rust/transpipe/
- Match `e.kind()`: `Io` is retryable, `AuthenticationFailed` and `UnexpectedReturnType` are not, `Parse` is a bug.
  https://redis.io/docs/latest/develop/clients/rust/error-handling/
- Only Bloom (`bf_*`) and HyperLogLog have helpers. Cuckoo, Count-min sketch, t-digest and Top-K have no dedicated methods and are called with low-level commands. JSON needs the `json` feature.
  https://redis.io/docs/latest/develop/clients/rust/prob/ and https://redis.io/docs/latest/develop/clients/rust/json/

## Old patterns

| Old | Current | Source |
| --- | --- | --- |
| async `close()` | `await r.aclose()` | https://redis.io/docs/latest/develop/clients/redis-py/async/ |
