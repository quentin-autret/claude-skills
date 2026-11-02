# Probabilistic types and vector sets

Sizing and traps of Bloom, Cuckoo, Count-Min Sketch, Top-K, t-digest and HyperLogLog, then quantization, filtering, performance and sharding of vector sets, in Redis Open Source 8.10.

## Contents

- Picking a probabilistic type
- Bloom filter
- Cuckoo filter
- Count-Min Sketch and Top-K
- t-digest and HyperLogLog
- Vector sets: storage and memory
- Vector sets: queries and filters
- Vector sets: performance and scale
- Old patterns

## Picking a probabilistic type

| Need | Type | Source |
| --- | --- | --- |
| membership, frequent adds | Bloom | https://redis.io/docs/latest/develop/data-types/probabilistic/bloom-filter/ |
| membership with deletes, faster checks | Cuckoo | https://redis.io/docs/latest/develop/data-types/probabilistic/cuckoo-filter/ |
| frequency of high-count items | Count-Min Sketch | https://redis.io/docs/latest/develop/data-types/probabilistic/count-min-sketch/ |
| heavy hitters (top K) | Top-K | https://redis.io/docs/latest/develop/data-types/probabilistic/top-k/ |
| percentiles, CDF, trimmed mean | t-digest | https://redis.io/docs/latest/develop/data-types/probabilistic/t-digest/ |
| unique counts | HyperLogLog | https://redis.io/docs/latest/develop/data-types/probabilistic/hyperloglogs/ |

- A "no" from Bloom or Cuckoo is certain, a "yes" can be a false positive.
  https://redis.io/docs/latest/develop/data-types/probabilistic/bloom-filter/
- Size every filter for its capacity and error rate. Auto-creation through `ADD` or `INSERT` uses defaults (`bf-error-rate` 0.01, `bf-initial-size` 100, `cf-initial-size` 1024) and suits only many small filters.
  https://redis.io/docs/latest/develop/data-types/probabilistic/configuration/
- `bf-error-rate` above 0.25 is silently treated as 0.25.
  https://redis.io/docs/latest/develop/data-types/probabilistic/configuration/

## Bloom filter

```
BF.RESERVE bikes:models 0.001 1000000
BF.MADD bikes:models "A" "B" "C"
BF.MEXISTS bikes:models "A" "B" "C"
```
https://redis.io/docs/latest/develop/data-types/probabilistic/bloom-filter/

- Adding never fails when full: the error rate climbs, or a sub-filter is stacked. Each sub-filter slows existence checks, not adds.
  https://redis.io/docs/latest/develop/data-types/probabilistic/bloom-filter/
- `EXPANSION` defaults to 2 (fewer sub-filters when growth is unknown), 1 saves memory when the size is known. `NONSCALING` drops one hash function but lets the error rate grow at capacity.
  https://redis.io/docs/latest/develop/data-types/probabilistic/bloom-filter/
- Cost: about 9.6 bits per item at 1% error, 14.4 at 0.1%, against about 40 bytes per IP in a Set.
  https://redis.io/docs/latest/develop/data-types/probabilistic/bloom-filter/
- No deletes.
  https://redis.io/docs/latest/develop/data-types/probabilistic/bloom-filter/

## Cuckoo filter

- Create with `CF.RESERVE`, since auto-creation through `CF.ADD` may be mis-sized.
  https://redis.io/docs/latest/develop/data-types/probabilistic/cuckoo-filter/
- Adding an item twice stores two entries: use `CF.ADDNX` when duplicates are possible.
  https://redis.io/docs/latest/develop/data-types/probabilistic/cuckoo-filter/
- It reports full before reaching capacity. Capacity and expansion round up to powers of 2. Bucket size trades fill rate against error (size 1: 55% fill, 0.78% error).
  https://redis.io/docs/latest/develop/data-types/probabilistic/cuckoo-filter/
- Each expansion adds error and costs speed. Growth stops at `cf-max-expansions` (32).
  https://redis.io/docs/latest/develop/data-types/probabilistic/cuckoo-filter/ and https://redis.io/docs/latest/develop/data-types/probabilistic/configuration/
- Coupon pattern: "no" means invalid, "yes" means check the database, then `CF.DEL` once used.
  https://redis.io/docs/latest/develop/data-types/probabilistic/cuckoo-filter/

## Count-Min Sketch and Top-K

- CMS only over-counts. Discard results below `error x total_count`, with `count` read from `CMS.INFO`: they are noise.

```
CMS.INITBYPROB bikes:profit 0.001 0.002
CMS.INCRBY bikes:profit "Rocky Mountain Racer" 200
```
  https://redis.io/docs/latest/develop/data-types/probabilistic/count-min-sketch/
- Top-K sizing: `width = k*log(k)`, `depth = max(5, log(k))`, decay 0.9. `TOPK.ADD` returns the item pushed out of the top K.
  https://redis.io/docs/latest/develop/data-types/probabilistic/top-k/
- Top-K approximates heavy hitters only and cannot give exact ranks: leaderboards use sorted sets.
  https://redis.io/docs/latest/develop/use-cases/leaderboard/

## t-digest and HyperLogLog

- `TDIGEST.ADD` does NOT create the key: `TDIGEST.CREATE` first. `COMPRESSION` defaults to 100 (higher is more accurate).
  https://redis.io/docs/latest/develop/data-types/probabilistic/t-digest/
- `TDIGEST.MERGE` into an existing key merges unless `OVERRIDE`. `TDIGEST.MIN`/`MAX` return `nan` when empty. Merge per-server sketches for fleet-wide percentiles.
  https://redis.io/docs/latest/develop/data-types/probabilistic/t-digest/
- HyperLogLog: at most 12 KB and 0.81% standard error. It is a string (copy with `GET`/`SET`), and keeps only a state, not the elements.
  https://redis.io/docs/latest/develop/data-types/probabilistic/hyperloglogs/

## Vector sets: storage and memory

```
VADD points VALUES 2 1.0 1.0 pt:A
VSIM points ELE pt:A WITHSCORES COUNT 4
```
https://redis.io/docs/latest/develop/data-types/vector-sets/

- Default quantization `Q8` (4x smaller than FP32). `NOQUANT` for precision, `BIN` (32x smaller) for extreme compactness.
  https://redis.io/docs/latest/develop/data-types/vector-sets/memory/
- `VEMB` returns quantized approximations, not the input. `BIN` turned 1.26 and 1.96 into 1 and 1.
  https://redis.io/docs/latest/develop/data-types/vector-sets/performance/
- Memory drivers: dimensions, high `M`, large or nested attribute JSON, `NOQUANT`. `REDUCE` projects vectors to fewer dimensions at `VADD` time, but the projection matrix is not replicated, so `VEMB` differs on replicas.
  https://redis.io/docs/latest/develop/data-types/vector-sets/memory/ and https://redis.io/docs/latest/develop/data-types/vector-sets/troubleshooting/
- FP32 blobs are little-endian. On other platforms use `VALUES`.
  https://redis.io/docs/latest/develop/data-types/vector-sets/

## Vector sets: queries and filters

- `FILTER` returns only elements that match both the similarity query and the expression. Only top-level attribute fields work (`.genre`, not `.movie.genre`).

```
VSIM vset VALUES 3 0 0 0 COUNT 10 FILTER '.year > 2000' FILTER-EF 500
```
  https://redis.io/docs/latest/develop/data-types/vector-sets/filtered-search/
- A missing field, invalid JSON or a type mismatch excludes the element silently. Check with `VGETATTR`.
  https://redis.io/docs/latest/develop/data-types/vector-sets/troubleshooting/
- `FILTER-EF` defaults to `COUNT * 100`: raise it for rare matches.
  https://redis.io/docs/latest/develop/data-types/vector-sets/filtered-search/
- Low recall: raise `EF`, reconsider `BIN`, and compare with `VSIM ... TRUTH` (exact, slow).
  https://redis.io/docs/latest/develop/data-types/vector-sets/troubleshooting/

## Vector sets: performance and scale

- `VADD` is single-threaded and slower than queries (a few thousand inserts per second). `CAS` offloads the candidate search to a background thread.
  https://redis.io/docs/latest/develop/data-types/vector-sets/performance/
- `DEL` on a large vector set can spike latency.
  https://redis.io/docs/latest/develop/data-types/vector-sets/troubleshooting/
- Shard manually across keys (key `vset:N` with N = crc32(item) mod shard count), query all shards in parallel and merge by score. Writes scale linearly, `VSIM` does not.
  https://redis.io/docs/latest/develop/data-types/vector-sets/scalability/
- Save and load keep the HNSW graph, with no rebuild.
  https://redis.io/docs/latest/develop/data-types/vector-sets/performance/

## Old patterns

| Old | Current | Source |
| --- | --- | --- |
| `loadmodule ./redisbloom.so` with options | `redis.conf` and `CONFIG SET` (8.0) | https://redis.io/docs/latest/develop/data-types/probabilistic/configuration/ |
| `ERROR_RATE`, `INITIAL_SIZE`, `CF_MAX_EXPANSIONS` | `bf-error-rate`, `bf-initial-size`, `cf-max-expansions` | https://redis.io/docs/latest/develop/data-types/probabilistic/configuration/ |
| Huge Set for approximate membership | Bloom or Cuckoo filter | https://redis.io/docs/latest/develop/data-types/sets/ |
