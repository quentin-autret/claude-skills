# JSON and time series

JSONPath semantics, memory and RESP3 reply shapes of Redis JSON, then retention, duplicates, compaction and out-of-order costs of Redis time series, in Redis Open Source 8.10.

## Contents

- When to use JSON
- JSONPath
- JSON memory
- JSON replies and RESP3
- Time series basics
- Retention and duplicates
- Compaction and aggregation
- Time series in a cluster
- Old patterns

## When to use JSON

- JSON reads nested values by path and updates them atomically (increment, append) without fetching the whole object, unlike a serialized String. A Lua script with cjson decodes the whole value every time.
  https://redis.io/docs/latest/develop/data-types/json/use_cases/ and https://redis.io/docs/latest/develop/data-types/json/performance/
- Path access keeps constant rates and latencies whatever the document size, because the value is stored deserialized. Target sub-paths on big documents.
  https://redis.io/docs/latest/develop/data-types/json/performance/
- Indexing and querying JSON is done by Redis Search.
  https://redis.io/docs/latest/develop/data-types/json/indexing_json/
- Max nesting depth is 128.
  https://redis.io/docs/latest/develop/data-types/json/

## JSONPath

- Default: JSONPath (starts with `$`), which acts on every match. Anything else is legacy path, which acts only on the first match.
  https://redis.io/docs/latest/develop/data-types/json/path/
- `$..price` updates all matches in one command: `JSON.NUMINCRBY bikes:inventory $..price -100`.
  https://redis.io/docs/latest/develop/data-types/json/path/
- Keys with special characters need brackets: `$["a~b"]`. In redis-cli, wrap paths holding double quotes in single quotes.

```
JSON.GET bikes:inventory '$..mountain_bikes[?(@.price < 3000 && @.specs.weight < 10)]'
```
  https://redis.io/docs/latest/develop/data-types/json/path/
- Filters that cannot evaluate (non-numeric operand, division by zero, type mismatch) yield nothing, silently: the element is skipped or the projection returns `[]`.
  https://redis.io/docs/latest/develop/data-types/json/path/
- Arithmetic needs spaces: `@.a + 1` adds, `@.a+1` is a field named `a+1`. `$.arr.length()` calls a function, `$.arr.length` reads a field. `!` binds tighter than `&&` and `||`.
  https://redis.io/docs/latest/develop/data-types/json/path/
- With a multi-node operand (`@.*`) a comparison holds if any node matches. `==` treats `1` and `1.0` as equal.
  https://redis.io/docs/latest/develop/data-types/json/path/
- Path cost is O(depth x keys per object): very wide objects slow navigation, array indexes are O(1).
  https://redis.io/docs/latest/develop/data-types/json/path/
- 8.10 adds projections, `in`/`nin`, `!`, arithmetic, and functions such as `length()`, `count()`, `keys()`, `min()`, `avg()`. It also reserves the words `in`, `nin`, `size`, `sizeof`, `empty`, `subsetof`, `anyof`, `noneof` and no longer accepts `~` in a dot-notation field name: use brackets, `$["a~b"]`. A field named like a reserved word is read with `$["size"]`.
  https://redis.io/docs/latest/develop/data-types/json/path/

## JSON memory

- In memory JSON is a tree and is usually larger than the text (381 bytes became 1069). Every value takes at least 8 bytes. Arrays start with room for 4 and double.
  https://redis.io/docs/latest/develop/data-types/json/ram/
- Deleting values from arrays or objects does not free the container's allocation.
  https://redis.io/docs/latest/develop/data-types/json/ram/
- A string that appears several times, in one document or across documents on the same node, is stored once. `JSON.DEBUG MEMORY` ignores this and counts every copy, so it overestimates datasets with repeated strings.
  https://redis.io/docs/latest/develop/data-types/json/ram/
- Vectors for search: `JSON.SET ... FPHA BF16|FP16|FP32|FP64` (8.8) forces the storage type of float arrays.
  https://redis.io/docs/latest/develop/data-types/json/

## JSON replies and RESP3

- `JSON.GET key $` returns an array of matches even for one match (`"[\"Hyperion\"]"`). Under RESP3, where `$` is the default path, `JSON.GET k` on `[1,2,3]` returns `[[1,2,3]]` instead of `[1,2,3]`.
  https://redis.io/docs/latest/develop/data-types/json/
- The optional path defaults to `.` under RESP2 and `$` under RESP3, which changes reply shapes: with `$`, `ARR*`, `OBJLEN`, `STR*`, `TOGGLE` return arrays of numbers or nulls.
  https://redis.io/docs/latest/develop/data-types/json/resp3/
- `JSON.MGET` gives no CROSSSLOT error in a cluster, but when the keys are not in the same slot it returns partial results, only for the slots on the current shard.
  https://redis.io/docs/latest/develop/using-commands/multi-key-operations/

## Time series basics

- `TS.ADD` auto-creates a missing key with defaults, `TS.MADD` does not. Create series explicitly with `TS.CREATE ... RETENTION ... LABELS ... DUPLICATE_POLICY` when defaults do not fit.
  https://redis.io/docs/latest/develop/data-types/timeseries/
- Batch writes with `TS.MADD`. Use `*` to stamp with the server clock.
  https://redis.io/docs/latest/develop/data-types/timeseries/
- Labels enable multi-series queries: `TS.MRANGE - + FILTER country=(us,uk) GROUPBY country REDUCE max`. With clustering enabled, `TS.MGET`/`TS.MRANGE` cannot be part of a transaction.
  https://redis.io/docs/latest/develop/data-types/timeseries/ and https://redis.io/docs/latest/develop/using-commands/multi-key-operations/
- Keep ingestion in order. With compressed chunks, 1% out-of-order samples cost about 10% of ingest rate and 5-25% cost 35-75% (measured on RedisTimeSeries 1.4). Reducing the chunk size lowers the impact (it can cut it in half at 99% out-of-order).
  https://redis.io/docs/latest/develop/data-types/timeseries/out-of-order_performance_considerations/
- NaN samples (8.6) mark missing measurements. All aggregators skip them except `countNaN` and `countAll`, and `TS.INCRBY`/`TS.DECRBY` return an error when the current value or the operand is NaN.
  https://redis.io/docs/latest/develop/data-types/timeseries/
- 8.10 adds `TS.NRANGE`/`TS.NREVRANGE` (multi-key range pivoted by timestamp), `TS.READ` (optionally blocking) and `TS.QUERYLABELS`.
  https://redis.io/docs/latest/develop/data-types/timeseries/

## Retention and duplicates

- Always set `RETENTION` (0 means forever). Retention counts back from the series' HIGHEST timestamp, not the wall clock: old samples go only when newer ones arrive.
  https://redis.io/docs/latest/develop/data-types/timeseries/ and https://redis.io/docs/latest/develop/data-types/timeseries/configuration/
- The default duplicate policy is `BLOCK`: a second sample at an existing timestamp errors. Precedence: `ON_DUPLICATE_POLICY` on TS.ADD, then the key's `DUPLICATE_POLICY`, then `ts-duplicate-policy`.
  https://redis.io/docs/latest/develop/data-types/timeseries/configuration/
- `ts-ignore-max-time-diff`/`ts-ignore-max-val-diff` drop noise-level repeats only when the policy is `LAST`, the sample is in order and the series is not a compaction.
  https://redis.io/docs/latest/develop/data-types/timeseries/configuration/
- Changing `ts-chunk-size-bytes` (default 4096) does not touch existing chunks.
  https://redis.io/docs/latest/develop/data-types/timeseries/configuration/

## Compaction and aggregation

- `TS.CREATERULE` needs the destination series to exist and only processes samples added after it is created.

```
TS.CREATE hyg:compacted
TS.CREATERULE hyg:1 hyg:compacted AGGREGATION min 3
```
  https://redis.io/docs/latest/develop/data-types/timeseries/
- The latest bucket is not written until it closes, and older buckets are never final: late or deleted samples still update them.
  https://redis.io/docs/latest/develop/data-types/timeseries/
- `ts-compaction-policy` applies only to keys auto-created by `TS.ADD`/`INCRBY`/`DECRBY`, not to `TS.CREATE`.
  https://redis.io/docs/latest/develop/data-types/timeseries/configuration/
- Buckets align on the epoch by default. Use `ALIGN start` for buckets that begin at the range start.
  https://redis.io/docs/latest/develop/data-types/timeseries/
- `TS.RANGE` and `TS.DEL` bounds are inclusive.
  https://redis.io/docs/latest/develop/data-types/timeseries/

## Time series in a cluster

- With `ts-compaction-policy` set, every series key needs a hash tag so compactions land in the source's slot. Otherwise compaction can fail silently.
  https://redis.io/docs/latest/develop/data-types/timeseries/configuration/
- `CONFIG SET` and `CONFIG REWRITE` must run on each node. `ts-num-threads` (default 3) is load-time only.
  https://redis.io/docs/latest/develop/data-types/timeseries/configuration/

## Old patterns

| Old | Current | Source |
| --- | --- | --- |
| Legacy path (`.`, first match only) | JSONPath `$` | https://redis.io/docs/latest/develop/data-types/json/path/ |
| Serialized JSON string in a String key | Redis JSON | https://redis.io/docs/latest/develop/data-types/json/use_cases/ |
| `CHUNK_SIZE_BYTES`, `RETENTION_POLICY`, `DUPLICATE_POLICY` module args | `ts-chunk-size-bytes`, `ts-retention-policy`, `ts-duplicate-policy` (8.0) | https://redis.io/docs/latest/develop/data-types/timeseries/configuration/ |
| `OSS_GLOBAL_PASSWORD` | ignored since 8.0 (internal shared secret) | https://redis.io/docs/latest/develop/data-types/timeseries/configuration/ |
