# Choosing and using core data types

Type choice, complexity traps and current commands for strings, lists, sets, sorted sets, arrays, bitmaps and geo in Redis Open Source 8.10. Hashes and their field expiration live in the keys and expiration reference.

## Contents

- Choosing a type
- Strings, counters and bitmaps
- Lists and queues
- Sets
- Sorted sets
- Arrays (8.8+)
- Geo
- Limits and sizing
- Old patterns

## Choosing a type

- Documents: String, Hash or JSON. Cost rises from String to Hash to JSON. Hash fits flat objects with few fields, JSON fits nested objects and has the stronger Redis Search support.
  https://redis.io/docs/latest/develop/data-types/compare-data-types/
- Structured data stored as a serialized String: consider a Hash or JSON instead.
  https://redis.io/docs/latest/develop/data-types/strings/
- Extra data about Set or Sorted set members goes in an auxiliary Hash or JSON object keyed by the member name. A leaderboard keeps the sorted set small (member = entity ID) and the profile in a Hash.
  https://redis.io/docs/latest/develop/data-types/compare-data-types/ and https://redis.io/docs/latest/develop/use-cases/leaderboard/
- A Set or Sorted set used as a query index: prefer JSON (or Hash) plus Redis Search.
  https://redis.io/docs/latest/develop/data-types/sets/, https://redis.io/docs/latest/develop/data-types/sorted-sets/ and https://redis.io/docs/latest/develop/data-types/compare-data-types/
- A huge Set used only for membership, where exactness is not needed: a Bloom or Cuckoo filter.
  https://redis.io/docs/latest/develop/data-types/sets/
- Elements read in timestamp order or by several consumers (consumer groups, at-least-once delivery): Streams rather than Lists. Fast access to the middle of a large collection: Sorted set rather than List.
  https://redis.io/docs/latest/develop/data-types/compare-data-types/ and https://redis.io/docs/latest/develop/data-types/lists/
- The Geo type is for simple cases. Redis Search geo supports more formats and queries.
  https://redis.io/docs/latest/develop/data-types/geospatial/

## Strings, counters and bitmaps

- `INCR`/`INCRBY` are atomic and start a missing key at 0. They reject decimals: use `INCRBYFLOAT`. Integers are signed 64-bit.
  https://redis.io/docs/latest/develop/data-types/strings/ and https://redis.io/docs/latest/develop/reference/errors/common/err-not-an-integer/
- `SET` overwrites any existing value, of any type, and drops its TTL (unless `KEEPTTL` is given).
  https://redis.io/docs/latest/commands/set/
- Max string size is 512 MB. `GETRANGE`, `SETRANGE` and `SUBSTR` can be O(N) on large strings.
  https://redis.io/docs/latest/develop/data-types/strings/
- Bitmaps suit boolean state indexed by integer IDs (100 million users in about 12 MB). `SETBIT` on a far offset grows the string, up to 512 MB for 2^32 bits. `BITCOUNT`, `BITPOS` and `BITOP` are O(N).
  https://redis.io/docs/latest/develop/data-types/strings/bitmaps/ and https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/memory-optimization/
- Split a huge bitmap across keys: key = bit / M, bit inside the key = bit mod M.
  https://redis.io/docs/latest/develop/data-types/strings/bitmaps/
- `BITFIELD` packs several small counters in one string. `BITOP` also offers `DIFF`, `DIFF1`, `ANDOR`, `ONE` (8.2).

```
BITFIELD bike:1:stats INCRBY u32 #0 -50 INCRBY u32 #1 1
```
https://redis.io/docs/latest/develop/data-types/strings/bitfields/ and https://redis.io/docs/latest/develop/whats-new/8-2/

## Lists and queues

- Head and tail operations are O(1). `LINDEX`, `LINSERT`, `LSET` are O(N).
  https://redis.io/docs/latest/develop/data-types/lists/
- Capped "latest N" list: push, then `LTRIM key -N -1` (or `LTRIM 0 N-1` with LPUSH).
  https://redis.io/docs/latest/develop/data-types/lists/
- Consumers block with `BRPOP`/`BLPOP` instead of polling. Timeout 0 waits forever, the reply is `[key, element]` or NULL on timeout.
  https://redis.io/docs/latest/develop/data-types/lists/
- Reliable job queue: pending IDs in a List, claim with `BLMOVE pending processing ...` so a crashed worker's job can be reclaimed, then a reclaimer moves timed-out jobs back (at-least-once). Job metadata in a Hash with `EXPIRE` so records do not leak. Delayed or priority jobs: a sorted set scored by run-at time.
  https://redis.io/docs/latest/develop/use-cases/job-queue/
- `LMPOP`/`BLMPOP` pop several elements from the first non-empty of several lists. 8.10 adds `LMOVEM`/`BLMOVEM` to move several elements at once.
  https://redis.io/docs/latest/commands/lmpop/ and https://redis.io/docs/latest/develop/whats-new/8-10/
- Removing the last element deletes the key. Streams are the only type that stays at length zero.
  https://redis.io/docs/latest/develop/data-types/lists/

## Sets

- `SMEMBERS` is O(N) in one reply and is dangerous at hundreds of thousands of members: iterate with `SSCAN`.
  https://redis.io/docs/latest/develop/data-types/sets/
- `SMISMEMBER` tests several members in one round trip. `SINTERCARD` (7.0), `SUNIONCARD` and `SDIFFCARD` (8.10) count without building the result.
  https://redis.io/docs/latest/develop/data-types/sets/ and https://redis.io/docs/latest/develop/whats-new/8-10/
- `SDIFF` is not commutative and set order is never guaranteed.
  https://redis.io/docs/latest/develop/data-types/sets/

## Sorted sets

- Scores are 64-bit doubles: integers are exact only within plus or minus 2^53. Beyond that, use a lexicographic index (zero-padded or big-endian member).
  https://redis.io/docs/latest/commands/zadd/ and https://redis.io/docs/latest/develop/clients/patterns/indexes/
- Equal scores sort as binary arrays of bytes: non-plain-ASCII text such as UTF-8 may not sort like a dictionary.
  https://redis.io/docs/latest/commands/zrange/
- `ZRANGE` with `REV`, `BYSCORE`, `BYLEX` and `LIMIT` replaces the older range commands. Traps:
  - With `BYSCORE` or `BYLEX` plus `REV`, start is the HIGH bound: `ZRANGE zset 10 5 REV BYSCORE`. Reversed bounds return an empty list.
  - `LIMIT` needs `BYSCORE` or `BYLEX`. A large offset walks the skipped elements first (up to O(N)).
  - `BYLEX` is only defined when all members share one score.
  - `(` makes a bound exclusive. Lex bounds need `(` or `[`, `-` and `+` are infinities.

  https://redis.io/docs/latest/commands/zrange/
- `ZADD` returns only the count of NEW members. Add `CH` to count updates too. `NX`/`XX`/`GT`/`LT` restrict writes, and options go before the score: `ZADD k NX 1 a` works, `ZADD k 1 a NX` is a syntax error.
  https://redis.io/docs/latest/commands/zadd/ and https://redis.io/docs/latest/develop/reference/errors/common/err-syntax-error/
- Leaderboards: `ZINCRBY` updates in place, `ZREVRANK` gives the rank, `ZRANGE ... REV LIMIT` gives top-N. One key per time window with `EXPIRE`, aggregated with `ZUNIONSTORE`. Top-K cannot give exact ranks.
  https://redis.io/docs/latest/develop/use-cases/leaderboard/
- Case-insensitive autocomplete: same score, members like `foo:Foo`, query with `BYLEX`, display the part after the colon.
  https://redis.io/docs/latest/commands/zrange/

## Arrays (8.8+)

- Arrays suit sparse or high-index access, ring buffers and timestamped logs. Lists stay better for push and pop at both ends, Streams for logs with consumer groups.
  https://redis.io/docs/latest/develop/data-types/arrays/
- `ARRING` keeps a fixed-size ring, `ARLASTITEMS` returns the N newest in order, `AROP` aggregates a range in one pass.

```
ARRING readings 3 "v0"
ARLASTITEMS readings 3
```
  https://redis.io/docs/latest/develop/data-types/arrays/
- `ARLEN` is highest index + 1, `ARCOUNT` counts non-empty slots: they can differ hugely. `ARGETRANGE` returns nil for empty slots and caps at 1,000,000 elements per call, so iterate with `ARSCAN`. Range commands are O(N).
  https://redis.io/docs/latest/develop/data-types/arrays/

## Geo

- `GEOADD` takes longitude then latitude. Query with `GEOSEARCH`. `SCAN ... TYPE` reports geo keys as `zset`.

```
GEOSEARCH bikes:rentable FROMLONLAT -122.2612767 37.7936847 BYRADIUS 5 km WITHDIST
```
  https://redis.io/docs/latest/develop/data-types/geospatial/ and https://redis.io/docs/latest/commands/scan/

## Limits and sizing

- Up to 2^32 - 1 elements per List, Set or Hash, and 2^32 keys per instance.
  https://redis.io/docs/latest/develop/data-types/lists/ and https://redis.io/docs/latest/develop/get-started/faq/
- FAQ figures (64-bit): empty instance about 3 MB, 1M small string keys about 85 MB, 1M hashes of 5 fields about 160 MB. Measure with `MEMORY USAGE` and `INFO memory`.
  https://redis.io/docs/latest/develop/get-started/faq/

## Old patterns

| Old | Current | Source |
| --- | --- | --- |
| `ZREVRANGE`, `ZRANGEBYSCORE`, `ZRANGEBYLEX` and reverse forms | `ZRANGE` with `REV`, `BYSCORE`, `BYLEX`, `LIMIT` (6.2) | https://redis.io/docs/latest/commands/zrange/ |
| `GEORADIUS`, `GEORADIUSBYMEMBER` | `GEOSEARCH`, `GEOSEARCHSTORE` (6.2) | https://redis.io/docs/latest/commands/georadius/ and https://redis.io/docs/latest/develop/data-types/geospatial/ |
| `RPOPLPUSH`, `BRPOPLPUSH` | `LMOVE`, `BLMOVE` (6.2) | https://redis.io/docs/latest/commands/rpoplpush/ and https://redis.io/docs/latest/develop/data-types/lists/ |
| `HMSET` | `HSET` with several field-value pairs | https://redis.io/docs/latest/develop/clients/nodejs/migration/ |
| WATCH loop to pop the lowest member | `ZPOPMIN`/`ZPOPMAX` (5.0) | https://redis.io/docs/latest/develop/using-commands/transactions/ |
