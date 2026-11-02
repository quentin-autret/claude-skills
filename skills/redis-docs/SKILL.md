---
name: redis-docs
description: Pitfalls, surprising defaults and recommended patterns of Redis Open Source 8.10, distilled from the official redis.io docs. Use when choosing a Redis data structure, naming keys, setting TTL and expiration (including hash field expiration with HEXPIRE and HSETEX), iterating with SCAN instead of KEYS, pipelining, transactions with MULTI/EXEC and WATCH, compare-and-set and locks, Lua scripts and Redis Functions, streams and consumer groups, pub/sub and sharded pub/sub, keyspace notifications, client-side caching, connection pooling or multiplexing, configuring node-redis, ioredis, redis-py or redis-rs, error handling, retries and failover, maxmemory and eviction policies, RDB and AOF persistence and backups, replication, Sentinel, Redis Cluster hash slots and CROSSSLOT errors, ACL users and TLS, latency and memory optimization, JSON, time series, probabilistic types, vector sets, Redis Search FT.* queries and vector search, or upgrading to Redis 8.
metadata:
  version: "8.10"
  source: https://redis.io/docs/latest/
  fetched: "2026-11-02"
---

# Redis from the official docs

## Overview

This skill holds what the Redis docs say that is easy to get wrong: commands that block the single main thread, TTLs silently dropped by `SET`, transactions without rollback, scripts lost on failover, at-most-once Pub/Sub, eviction policies that never evict, cluster multi-key rules, client defaults that replay or lose writes, and the commands Redis 8 added to replace old workarounds. Generic database and caching advice is left out. Every claim carries the docs URL it comes from.

## Version covered

Redis Open Source 8.10, as documented on redis.io/docs/latest. Since Redis 8.0, Redis Stack and the community edition are merged into Redis Open Source, which ships JSON, time series, probabilistic types, Redis Search and vector sets without separate modules. Clients covered: node-redis, ioredis, redis-py and redis-rs. Redis Cloud, Redis Software and Kubernetes are out of scope. Before reviewing code written for Redis 7 or older, read "Old patterns" below.

https://redis.io/docs/latest/develop/whats-new/8-0/

## How to navigate

1. Read the quick start below. It covers the rules that matter in most tasks.
2. Open the one reference file whose domain matches the task, from the table at the end. Each file stands alone.
3. Every claim ends with its source URL. Reopen it when a detail decides the outcome.

## Quick start

### Data modeling

- Documents: Hash for simple objects with few fields and no nesting, JSON for complex nested ones (JSON also has more powerful Search features than hashes). A String holds a chunk whose internal structure your application manages.
  https://redis.io/docs/latest/develop/data-types/compare-data-types/
- Sorted set scores are doubles: integers are exact only within plus or minus 2^53.
  https://redis.io/docs/latest/commands/zadd/
- `ZRANGE ... BYSCORE REV` takes the HIGH bound first (`ZRANGE z 10 5 REV BYSCORE`), and `LIMIT` needs `BYSCORE` or `BYLEX`.
  https://redis.io/docs/latest/commands/zrange/
- Name keys `type:id` with one prefix per data kind, so two parts of an app never share a key of another type (WRONGTYPE).
  https://redis.io/docs/latest/develop/using-commands/keyspace/ and https://redis.io/docs/latest/develop/reference/errors/common/wrongtype/

### Keys and expiration

- A plain `SET` discards the TTL: write with `SET k v EX 60`, keep a TTL with `KEEPTTL`. Never `SET` then `EXPIRE`.
  https://redis.io/docs/latest/commands/set/ and https://redis.io/docs/latest/develop/using-commands/keyspace/
- Hash fields expire individually since 7.4. Default: `HSETEX` to write with a TTL, `HGETEX` to read and refresh.
  https://redis.io/docs/latest/develop/data-types/hashes/ and https://redis.io/docs/latest/commands/hsetex/
- Many keys (25% or more of those with a TTL) expiring in the same second can block the server, for example after `EXPIREAT` with one Unix time on many keys.
  https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/latency/
- Use extreme care with `KEYS` in production. Loop `SCAN` until the cursor is 0: pages can be empty or repeat keys, and `MATCH` filters after retrieval.
  https://redis.io/docs/latest/commands/keys/ and https://redis.io/docs/latest/commands/scan/
- `UNLINK` rather than `DEL` for large values. `FLUSHALL` clears every database.
  https://redis.io/docs/latest/commands/unlink/ and https://redis.io/docs/latest/commands/flushall/

### Atomicity

- Transactions have no rollback: a command failing inside EXEC does not stop the others. Only queue-time errors abort (EXECABORT).
  https://redis.io/docs/latest/develop/using-commands/transactions/ and https://redis.io/docs/latest/develop/reference/errors/common/execabort/
- WATCH is per connection and aborts on any change, expiry included. Retry on a null EXEC. Never WATCH on a shared multiplexed connection.
  https://redis.io/docs/latest/develop/using-commands/transactions/ and https://redis.io/docs/latest/develop/clients/nodejs/transpipe/
- On 8.4+, compare-and-set and lock release use `SET ... IFEQ` and `DELEX ... IFEQ` instead of WATCH loops or Lua. On 8.8+, `INCREX` does a window counter in one command.
  https://redis.io/docs/latest/develop/using-commands/transactions/ and https://redis.io/docs/latest/develop/whats-new/8-8/
- A pipeline cannot feed one reply into a later write: that needs a script or function. Send large pipelines in batches of about 10k commands.
  https://redis.io/docs/latest/develop/using-commands/pipelining/
- Redis 7 and later offer Functions (`FUNCTION LOAD`, `FCALL`), managed by the database and kept through persistence and replication. The EVAL script cache is lost on restart and failover (NOSCRIPT).
  https://redis.io/docs/latest/develop/programmability/ and https://redis.io/docs/latest/develop/reference/errors/common/noscript/
- Pass every key as a key argument and never embed values in script text. A script running past `busy-reply-threshold` (5 s by default) is not killed: other clients get BUSY, and after a write only `SHUTDOWN NOSAVE` stops it.
  https://redis.io/docs/latest/commands/eval/ and https://redis.io/docs/latest/develop/programmability/ and https://redis.io/docs/latest/develop/reference/errors/common/busy/
- Cluster does not load functions on other nodes for you: load them on all cluster nodes (replicas receive them from their primary).
  https://redis.io/docs/latest/develop/programmability/functions-intro/

### Messaging

- Pub/Sub and keyspace notifications are at-most-once: offline subscribers lose messages. Use Streams for persistence and at-least-once.
  https://redis.io/docs/latest/develop/pubsub/
- Streams: create groups with `XGROUP CREATE ... $ MKSTREAM` and catch BUSYGROUP, always `XACK`, recover dead consumers with `XAUTOCLAIM`, cap with `MAXLEN ~`.
  https://redis.io/docs/latest/commands/xgroup-create/ and https://redis.io/docs/latest/commands/xautoclaim/ and https://redis.io/docs/latest/commands/xadd/
- Retrying producers use `XADD ... IDMP` (8.6) to avoid duplicates. In the general case exactly-once processing cannot be obtained.
  https://redis.io/docs/latest/develop/data-types/streams/idempotency/ and https://redis.io/docs/latest/develop/data-types/streams/
- In Cluster use sharded Pub/Sub (`SSUBSCRIBE`/`SPUBLISH`). Keyspace notifications are off by default and per node.
  https://redis.io/docs/latest/develop/pubsub/ and https://redis.io/docs/latest/develop/pubsub/keyspace-notifications/

### Clients

- Open as few connections as possible: use a pool or a multiplexer, not a new connection per operation. Blocking pops (`BLPOP`) need a pool or a dedicated connection, never a multiplexer.
  https://redis.io/docs/latest/develop/clients/pools-and-muxing/
- Portable URL: `rediss://default:<percent-encoded password>@host:6379/0`, lowercase scheme, no query parameters.
  https://redis.io/docs/latest/develop/clients/connection-urls/
- Retry timeouts, temporary network errors and `LOADING` with backoff. Fail fast on `WRONGTYPE`, syntax and auth errors. Cache reads fall back to the database.
  https://redis.io/docs/latest/develop/clients/error-handling/
- node-redis: `await client.connect()`, an `error` listener on every client, and beware the offline queue replaying non-idempotent commands (`disableOfflineQueue`).
  https://redis.io/docs/latest/develop/clients/nodejs/error-handling/ and https://redis.io/docs/latest/develop/clients/nodejs/produsage/
- redis-py and ioredis (by default) ignore Sentinel failover announcements and keep writing to the old primary. ioredis needs `failoverDetector: true`.
  https://redis.io/docs/latest/develop/clients/redis-py/connect/ and https://redis.io/docs/latest/develop/clients/ioredis/connect/
- redis-rs async responses time out after 500 ms, blocking commands included: send BLPOP on its own connection with no response timeout.
  https://redis.io/docs/latest/develop/clients/rust/connect/

### Memory and operations

- `maxmemory` defaults to 0 (no limit) on 64-bit systems: set it, and with replication or persistence leave some RAM free for buffers. The default policy `noeviction` fails writes at the limit. `allkeys-lru` is a good default for a cache. `volatile-*` policies act like `noeviction` when no key has a TTL.
  https://redis.io/docs/latest/develop/reference/eviction/ and https://redis.io/docs/latest/develop/reference/errors/common/oom/
- Persistence for data you keep: RDB + AOF with `appendfsync everysec`. Switch to AOF on the live server with `CONFIG SET appendonly yes`, never by editing the file and restarting.
  https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/
- `CONFIG SET` is lost at restart unless followed by `CONFIG REWRITE`.
  https://redis.io/docs/latest/operate/oss_and_stack/management/config/
- Replication is asynchronous: acknowledged writes can be lost on failover, even with `WAIT`. A primary without persistence that auto-restarts wipes its replicas.
  https://redis.io/docs/latest/commands/wait/ and https://redis.io/docs/latest/operate/oss_and_stack/management/replication/
- Sentinel: at least 3 on independent machines. Cluster: multi-key commands, transactions and scripts need one slot (hash tags `{...}`), and one uncovered slot stops the cluster by default.
  https://redis.io/docs/latest/operate/oss_and_stack/management/sentinel/ and https://redis.io/docs/latest/operate/oss_and_stack/management/scaling/ and https://redis.io/docs/latest/operate/oss_and_stack/reference/cluster-spec/ and https://redis.io/docs/latest/develop/reference/errors/common/clusterdown/
- Linux hosts: `vm.overcommit_memory = 1` and transparent huge pages disabled.
  https://redis.io/docs/latest/operate/oss_and_stack/management/admin/

### Security

- The default user has no password and every right (`on nopass ~* &* +@all`). Create least-privilege ACL users: a worker that fetches jobs does not need `FLUSHALL`, `FLUSHDB` or `SWAPDB`.
  https://redis.io/docs/latest/operate/oss_and_stack/management/security/acl/
- Redis 8 puts module commands in standard ACL categories: `+@all -@write` no longer allows `JSON.SET`.
  https://redis.io/docs/latest/develop/whats-new/8-0/
- TLS adds `tls-port` next to `port`: set `port 0` to close plaintext.
  https://redis.io/docs/latest/operate/oss_and_stack/management/security/encryption/

### Search, JSON and specialized types

- Search: use `DIALECT 2`+ and bound result sets with `LIMIT` or `CURSOR`. Timeouts (500 ms) return partial results with no error unless `search-on-timeout FAIL`. JSON TAG fields have no default separator.
  https://redis.io/docs/latest/develop/ai/search-and-query/administration/configuration/ and https://redis.io/docs/latest/develop/ai/search-and-query/indexing/ and https://redis.io/docs/latest/develop/ai/search-and-query/best-practices/scalable-query-best-practices/
- `FT.ALTER` only adds fields: reindex behind an alias and switch with `FT.ALIASUPDATE`.
  https://redis.io/docs/latest/develop/ai/search-and-query/best-practices/index-mgmt-best-practices/
- JSONPath (`$`) acts on every match and returns arrays. Legacy paths act on the first match only.
  https://redis.io/docs/latest/develop/data-types/json/path/
- Time series duplicate policy defaults to BLOCK, and retention counts from the newest sample, not the clock.
  https://redis.io/docs/latest/develop/data-types/timeseries/configuration/
- `TDIGEST.ADD` does not create its key. Cuckoo filters allow deletions, which Bloom filters do not. Vector set filters silently exclude elements with missing or mistyped attributes.
  https://redis.io/docs/latest/develop/data-types/probabilistic/t-digest/ and https://redis.io/docs/latest/develop/data-types/probabilistic/bloom-filter/ and https://redis.io/docs/latest/develop/data-types/vector-sets/troubleshooting/

## Old patterns

| Old | Current | Source |
| --- | --- | --- |
| `SETNX`, `SETEX`, `PSETEX`, `GETSET` | `SET` with `NX`, `EX`/`PX`, `GET` | https://redis.io/docs/latest/commands/set/ |
| WATCH loop or Lua for compare-and-set and lock release | `SET ... IFEQ`, `DELEX` (8.4) | https://redis.io/docs/latest/develop/using-commands/transactions/ |
| `INCR` + `EXPIRE` window counter | `INCREX` (8.8) | https://redis.io/docs/latest/develop/whats-new/8-8/ |
| `HSET` then `HEXPIRE` | `HSETEX` (8.0) | https://redis.io/docs/latest/commands/hsetex/ |
| `ZRANGEBYSCORE`, `ZREVRANGE` and family | `ZRANGE` with `BYSCORE`, `BYLEX`, `REV` | https://redis.io/docs/latest/commands/zrange/ |
| `GEORADIUS`, `RPOPLPUSH` | `GEOSEARCH`, `LMOVE`/`BLMOVE` | https://redis.io/docs/latest/develop/data-types/geospatial/ and https://redis.io/docs/latest/commands/georadius/ and https://redis.io/docs/latest/commands/rpoplpush/ (both "Deprecated as of Redis v6.2.0") |
| EVAL/EVALSHA app logic | Redis Functions (7.0) | https://redis.io/docs/latest/develop/programmability/functions-intro/ |
| `XPENDING` + `XCLAIM` recovery | `XAUTOCLAIM`, `XREADGROUP ... CLAIM` (8.4) | https://redis.io/docs/latest/commands/xreadgroup/ |
| `requirepass`, `rename-command` | ACL users and rules | https://redis.io/docs/latest/operate/oss_and_stack/management/security/ |
| `CLUSTER SLOTS` | `CLUSTER SHARDS` | https://redis.io/docs/latest/operate/oss_and_stack/reference/cluster-spec/ |
| Redis Stack with `loadmodule` options | Redis Open Source 8 with `search-*`, `ts-*`, `bf-*` configs | https://redis.io/docs/latest/develop/ai/search-and-query/administration/configuration/ and https://redis.io/docs/latest/develop/data-types/probabilistic/configuration/ |
| `BM25` scorer | `BM25STD` (8.4) | https://redis.io/docs/latest/develop/ai/search-and-query/administration/overview/ |
| ioredis for new Node.js projects | node-redis | https://redis.io/docs/latest/develop/clients/nodejs/migration/ |
| `*-max-ziplist-*` configs | `*-max-listpack-*` (7.0) | https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/memory-optimization/ |

## Reference files

| File | Read it for |
| --- | --- |
| [references/data-types.md](references/data-types.md) | type choice, strings, bitmaps, lists and job queues, sets, sorted sets and leaderboards, arrays, geo, limits |
| [references/keys-and-expiration.md](references/keys-and-expiration.md) | key naming, `SET` options, key TTL, hash field expiration, `SCAN` vs `KEYS`, `UNLINK`, cache-aside and sessions |
| [references/transactions-pipelining-locks.md](references/transactions-pipelining-locks.md) | pipelining, MULTI/EXEC, WATCH, `IFEQ`/`DELEX`, distributed locks and Redlock, rate limiters, `WAIT` |
| [references/scripting-and-functions.md](references/scripting-and-functions.md) | Functions vs EVAL, key declaration, script cache, flags, BUSY, cluster loading, Lua type conversion |
| [references/streams-and-pubsub.md](references/streams-and-pubsub.md) | streams, trimming, consumer groups, claiming, idempotent producers, Pub/Sub, sharded Pub/Sub, keyspace notifications |
| [references/client-connections.md](references/client-connections.md) | client choice, `HELLO` and RESP3, connection URLs, pools and multiplexing, client-side caching, limits, retries, Sentinel discovery |
| [references/nodejs-clients.md](references/nodejs-clients.md) | node-redis lifecycle, offline queue, pipelines, WATCH pools, Sentinel, caching, ioredis migration |
| [references/python-and-rust-clients.md](references/python-and-rust-clients.md) | redis-py timeouts, retries, asyncio, pipelines, Sentinel, failover, redis-rs connections and timeouts |
| [references/errors.md](references/errors.md) | meaning and fix of MOVED, ASK, CROSSSLOT, OOM, MISCONF, READONLY, BUSY, NOSCRIPT, WRONGPASS, NOPERM and others |
| [references/memory-eviction-performance.md](references/memory-eviction-performance.md) | `maxmemory`, eviction policies, INFO memory, compact hashes and `HIMPORT`, slow commands, latency, benchmarks |
| [references/persistence-and-backups.md](references/persistence-and-backups.md) | RDB vs AOF, fsync, switching to AOF, backups and `BACKUP`, config files, shutdown, host setup |
| [references/replication-sentinel-cluster.md](references/replication-sentinel-cluster.md) | replicas, write-loss bounds, Sentinel deployment and failover, Cluster slots, hash tags, redirects, operations |
| [references/security-acl-tls.md](references/security-acl-tls.md) | default exposure, ACL users and traps, key and channel permissions, ACL files, TLS, security fixes |
| [references/upgrades-and-versions.md](references/upgrades-and-versions.md) | release model, Redis 8 packaging, breaking changes, standalone and cluster upgrades, additions by release |
| [references/json-and-time-series.md](references/json-and-time-series.md) | JSONPath semantics, JSON memory, RESP3 replies, time series retention, duplicates, compaction, cluster |
| [references/probabilistic-and-vector-sets.md](references/probabilistic-and-vector-sets.md) | Bloom, Cuckoo, Count-Min Sketch, Top-K, t-digest, HyperLogLog, vector set quantization, filters, sharding |
| [references/search-and-vector-index.md](references/search-and-vector-index.md) | `FT.CREATE` schema, hash vs JSON indexes, query syntax, timeouts, aliases, vector indexes and KNN, `FT.HYBRID` |

Sources and fetch details are listed in [SOURCES.md](SOURCES.md).
