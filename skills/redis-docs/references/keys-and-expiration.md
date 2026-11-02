# Keys, TTL, hash field expiration and keyspace iteration

Key naming, SET options, key and hash-field expiration, SCAN versus KEYS, and deletion in Redis Open Source 8.10.

## Contents

- Key naming
- SET options
- Key expiration
- Hash field expiration
- Iterating the keyspace
- Deleting and flushing
- Caching and session patterns
- Old patterns

## Key naming

- Use `object-type:id` (`user:1000`), colon separated, with dots or dashes inside multi-word parts (`comment:4321:reply-to`). Redis has no namespaces.
  https://redis.io/docs/latest/develop/using-commands/keyspace/
- One prefix per kind of data (`user:` hashes, `queue:` lists) avoids WRONGTYPE when two parts of an app share a name.
  https://redis.io/docs/latest/develop/reference/errors/common/wrongtype/
- Avoid very long keys (memory and slow comparisons): hash a large value (SHA1) before embedding it. Avoid cryptic short keys too.
  https://redis.io/docs/latest/develop/using-commands/keyspace/
- In a cluster, hash tags `{...}` co-locate keys on one slot, but too many keys mapped to one hash slot eventually harm database performance.
  https://redis.io/docs/latest/develop/using-commands/keyspace/

## SET options

`SET key value [NX|XX|IFEQ ifeq-value|IFNE ifne-value|IFDEQ ifdeq-digest|IFDNE ifdne-digest] [GET] [EX|PX|EXAT|PXAT|KEEPTTL]`
https://redis.io/docs/latest/commands/set/

- A plain `SET` discards the existing TTL. Add `KEEPTTL` to keep it.
  https://redis.io/docs/latest/commands/set/
- Conditions are mutually exclusive, so are the expiration options. The 8.10 release notes list as fixed: "`SET` does not enforce mutually exclusive `NX`/`XX` and `IF*` options".
  https://redis.io/docs/latest/commands/set/ and https://redis.io/docs/latest/develop/whats-new/8-10/
- `IFEQ`/`IFNE`/`IFDEQ`/`IFDNE` (8.4): `IFEQ`/`IFNE` compare the current value, `IFDEQ`/`IFDNE` compare its digest from `DIGEST`. `IFNE`/`IFDNE` create a missing key, `IFEQ`/`IFDEQ` do not.
  https://redis.io/docs/latest/commands/set/
- A nil reply is ambiguous: missing key with `XX`/`IFEQ`, or existing key with `NX` or a failed condition. With `GET` the reply is the old value (nil if the key did not exist) whether or not the write happened, and `GET` errors if the old value is not a string.
  https://redis.io/docs/latest/commands/set/
- `MSETEX` (8.4) sets several keys with an expiration atomically.
  https://redis.io/docs/latest/develop/whats-new/8-4/

## Key expiration

- Keys can be created with a TTL by the writing command itself: `SET k v EX 60`.
  https://redis.io/docs/latest/develop/using-commands/keyspace/
- `GETEX key EX n` reads and refreshes the TTL (sliding expiry), `GETEX key PERSIST` removes it. GETEX is a write command.
  https://redis.io/docs/latest/commands/getex/
- `EXPIRE key s NX|XX|GT|LT` (7.0) updates conditionally. A key without TTL is treated as an infinite TTL for `GT` and `LT`.
  https://redis.io/docs/latest/commands/expire/
- Only commands that delete or overwrite the key clear its TTL (`DEL`, `SET`, `GETSET`, `*STORE`). `INCR`, `LPUSH`, `HSET` keep it. `RENAME` carries the TTL to the new name.
  https://redis.io/docs/latest/commands/expire/
- A non-positive `EXPIRE` or a past `EXPIREAT` deletes the key, and the keyspace event is `del`, not `expired`.
  https://redis.io/docs/latest/commands/expire/
- Expirations are absolute Unix times: time runs while Redis is down, and an RDB moved to a host with clock skew can expire keys at load.
  https://redis.io/docs/latest/commands/expire/
- Expiry is lazy (on access) plus periodic sampling, so expired keys can linger. Replicas connected to a primary do not expire keys independently, they wait for the DEL from the primary.
  https://redis.io/docs/latest/commands/expire/
- Many keys sharing one expiry second can block the server (the expire cycle repeats while over 25% of sampled keys are expired). The page's example cause is `EXPIREAT` used extensively with the same Unix time.
  https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/latency/
- A TTL costs memory per key. `allkeys-lru` is more memory efficient than a TTL-based policy since it needs no expire value.
  https://redis.io/docs/latest/develop/reference/eviction/

## Hash field expiration

Per-field TTL exists since 7.4: `HEXPIRE`, `HPEXPIRE`, `HEXPIREAT`, `HPEXPIREAT`, read with `HTTL`/`HPTTL`/`HEXPIRETIME`, clear with `HPERSIST`. 8.0 adds `HSETEX`, `HGETEX`, `HGETDEL`.
https://redis.io/docs/latest/develop/data-types/hashes/

```
HSETEX mykey FNX EX 60 FIELDS 2 field1 "Hello" field2 "World"
HGETEX mykey EX 120 FIELDS 1 field1
HEXPIRE mykey 10 NX FIELDS 1 field1
```
https://redis.io/docs/latest/commands/hsetex/ and https://redis.io/docs/latest/commands/hgetex/ and https://redis.io/docs/latest/commands/hexpire/

- Default: `HSETEX` to write and expire atomically, `HGETEX` to read and refresh.
  https://redis.io/docs/latest/commands/hsetex/
- `HSETEX` on existing fields discards their TTL unless `KEEPTTL`. `FNX` writes only if no field exists, `FXX` only if all exist. It returns 1 if all fields were set, 0 if none.
  https://redis.io/docs/latest/commands/hsetex/
- `HEXPIRE` replies per field: -2 missing, 0 condition not met, 1 TTL set, 2 field deleted (0 seconds or a past time deletes it).
  https://redis.io/docs/latest/commands/hexpire/
- Only `HDEL` and `HSET` (delete or overwrite) clear a field TTL. A hash that uses field expiration is not auto-converted to a compact hash.
  https://redis.io/docs/latest/commands/hexpire/ and https://redis.io/docs/latest/develop/data-types/hashes/
- Listed use cases: last-hour event tracking (`HLEN` counts live events), hourly fraud counters, per-session fields, active sessions with inactivity TTL.
  https://redis.io/docs/latest/develop/data-types/hashes/

## Iterating the keyspace

- `KEYS` is O(N) and blocks the server. It is meant for debugging and special operations. Application code uses `SCAN`, and `SSCAN`/`HSCAN`/`ZSCAN` instead of `SMEMBERS` on large collections.
  https://redis.io/docs/latest/commands/keys/ and https://redis.io/docs/latest/commands/scan/
- The iteration ends only when the returned cursor is 0. A page can be empty, and an element can come back twice, so processing must be idempotent.
  https://redis.io/docs/latest/commands/scan/
- `COUNT` is a hint (default 10). `MATCH` filters after retrieval, so sparse matches return mostly empty pages and need a higher COUNT (for example 1000). `TYPE` works on the whole-database `SCAN` only.

```
SCAN 0 MATCH *11* COUNT 1000
HSCAN myhash 0 NOVALUES
```
  https://redis.io/docs/latest/commands/scan/
- `SSCAN`/`HSCAN`/`ZSCAN` return a small packed collection in one call whatever the COUNT. `TYPE` cannot tell geo from zset or HyperLogLog from string.
  https://redis.io/docs/latest/commands/scan/
- In an OSS cluster, `SCAN`, `KEYS` and `DBSIZE` cover only the current shard. A pattern starting with a hash tag and no wildcard inside it (`{abc}*`) scans one slot (8.0).
  https://redis.io/docs/latest/develop/using-commands/multi-key-operations/ and https://redis.io/docs/latest/operate/oss_and_stack/reference/cluster-spec/

## Deleting and flushing

- `DEL` frees memory synchronously. Default for large values: `UNLINK`, which reclaims memory in another thread.
  https://redis.io/docs/latest/commands/unlink/
- `RENAME` onto an existing key does an implicit DEL, slow for a big value. In a cluster both names must share a slot.
  https://redis.io/docs/latest/commands/rename/
- `FLUSHALL` clears every database, not only the selected one, rewrites an empty RDB if `save` is on, and keeps functions. It is synchronous unless `ASYNC` or `lazyfree-lazy-user-flush yes`. In an OSS cluster it hits the current shard only.
  https://redis.io/docs/latest/commands/flushall/ and https://redis.io/docs/latest/develop/using-commands/multi-key-operations/

## Caching and session patterns

- Cache-aside: key `cache:{entity}:{id}` with a TTL. On a miss read the primary and write back. On an update write the primary, then `DEL` the cache key.
  https://redis.io/docs/latest/develop/use-cases/cache-aside/
- A hot key expiring under load causes a stampede. Guard it with an atomic mutex or a probabilistic early refresh in a script.
  https://redis.io/docs/latest/develop/use-cases/cache-aside/
- Prefetched reference data gets a long safety-net TTL, so a stopped sync job cannot serve stale data forever.
  https://redis.io/docs/latest/develop/use-cases/prefetch-cache/
- Sessions: one Hash per session with `EXPIRE` reset on each request, plus a Set of the user's sessions for logout-all. Enable persistence, since lost sessions have a business cost.
  https://redis.io/docs/latest/develop/use-cases/session-store/

## Old patterns

| Old | Current | Source |
| --- | --- | --- |
| `SETNX`, `SETEX`, `PSETEX`, `GETSET` | `SET` with `NX`, `EX`/`PX`, `GET` (old commands may be removed) | https://redis.io/docs/latest/commands/set/ |
| `SET` then `EXPIRE` | `SET ... EX`, `MSETEX` (8.4) for several keys | https://redis.io/docs/latest/develop/using-commands/keyspace/ and https://redis.io/docs/latest/develop/whats-new/8-4/ |
| `HSET` then `HEXPIRE` | `HSETEX` (8.0) | https://redis.io/docs/latest/commands/hsetex/ |
| `KEYS pattern` in code | `SCAN` cursor loop | https://redis.io/docs/latest/commands/keys/ |
| `DEL` on large values | `UNLINK` | https://redis.io/docs/latest/commands/unlink/ |
