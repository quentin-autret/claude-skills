# Memory, eviction, latency and benchmarking

maxmemory and eviction policies, compact encodings and compact hashes, INFO memory, slow commands, latency sources and benchmarking in Redis Open Source 8.10.

## Contents

- maxmemory and eviction
- Measuring memory
- Compact encodings and compact hashes (8.10)
- Slow commands
- Latency sources
- Diagnosing latency
- Benchmarking
- Old patterns

## maxmemory and eviction

- Always set `maxmemory` below physical RAM (8 or 9 GB if 10 GB are free), leaving room for replication and AOF buffers, which are not counted. `mem_not_counted_for_evict` in `INFO memory` estimates that headroom.
  https://redis.io/docs/latest/operate/oss_and_stack/management/admin/ and https://redis.io/docs/latest/develop/reference/eviction/
- The default policy is `noeviction`: at the limit writes fail with OOM and reads keep working.
  https://redis.io/docs/latest/develop/reference/errors/common/oom/
- Cache default: `maxmemory-policy allkeys-lru`. Keys then need no TTL. Escape hatches: `allkeys-random` for uniform access, `volatile-ttl` when short TTLs mark the best victims, `allkeys-lrm`/`volatile-lrm` (8.6) to evict data not modified recently whatever the reads.
  https://redis.io/docs/latest/develop/reference/eviction/
- `volatile-*` policies act like `noeviction` when no key has a TTL. `volatile-*` policies are mainly for one instance holding both cache and persistent keys, but the doc advises two separate instances if possible.
  https://redis.io/docs/latest/develop/reference/eviction/
- One big command (a large stored intersection) can overshoot `maxmemory` by a lot.
  https://redis.io/docs/latest/develop/reference/eviction/
- `maxmemory-samples` defaults to 5. 10 approximates true LRU for more CPU.
  https://redis.io/docs/latest/develop/reference/eviction/
- Cache health: hit ratio `keyspace_hits / (hits + misses)`, then `evicted_keys` (policy or size wrong) and `expired_keys` (TTL too low). An `EXISTS` on a missing key counts as a miss.
  https://redis.io/docs/latest/develop/reference/eviction/
- Replicas ignore `maxmemory` by default (unless promoted), so they can use more memory than `maxmemory` says. Monitor them. `replica-ignore-maxmemory no` changes this.
  https://redis.io/docs/latest/operate/oss_and_stack/management/replication/

## Measuring memory

- `MEMORY USAGE key SAMPLES 0` measures every nested value. The default samples 5 and estimates large aggregates.
  https://redis.io/docs/latest/commands/memory-usage/
- `used_memory_rss` should be slightly above `used_memory`. RSS far above may mean fragmentation (check `allocator_frag_ratio`, not `mem_fragmentation_ratio`). `used_memory` above RSS means swapping.
  https://redis.io/docs/latest/commands/info/
- Redis does not always return freed memory to the OS: RSS can stay near the peak after deletes, which makes the fragmentation ratio unreliable. Provision for peak, not average.
  https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/memory-optimization/
- RDB saves and AOF rewrites can double memory on write-heavy loads (copy-on-write). Size RAM accordingly.
  https://redis.io/docs/latest/operate/oss_and_stack/management/admin/
- `MEMORY DOCTOR` and `LATENCY DOCTOR` give guided diagnosis.
  https://redis.io/docs/latest/operate/oss_and_stack/management/admin/

## Compact encodings and compact hashes (8.10)

- Small hashes, lists, sets and sorted sets use listpack (`OBJECT ENCODING`) and convert automatically to the normal encoding past `*-max-listpack-*` limits. Raise limits only after benchmarking the conversion.
  https://redis.io/docs/latest/commands/object-encoding/ and https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/memory-optimization/
- Many small hashes beat one key per field.
  https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/memory-optimization/
- Compact hashes (8.10) store shared field names once for hashes with the same shape. Load them with `HIMPORT PREPARE` once per connection, then pipelined `HIMPORT SET` with values only:

```
HIMPORT PREPARE user name email age
HIMPORT SET user:1 user Alice alice@example.com 34
```
  https://redis.io/docs/latest/develop/data-types/hashes/
- `HIMPORT SET` replaces an existing key. Field sets are per connection. Field order in `HGETALL` can change. Most clients mark their support as experimental (redis-py needs v8.1.0 or later), and Redis Cloud and Redis Software do not support it.
  https://redis.io/docs/latest/develop/data-types/hashes/ and https://redis.io/docs/latest/develop/clients/redis-py/queryjson/
- Poor fit: unique or changing field names per key, very large hashes (moved as one blob in replication and slot migration). Once compact, a hash stays compact.
  https://redis.io/docs/latest/develop/data-types/hashes/
- Apps that cannot change code can turn on automatic conversion with `hash-min-template-entries`/`hash-max-template-entries` (both default to 0, which is off: min 0 disables conversion, max 0 means no upper bound). It applies lazily on the next write. A hash using field expiration is never converted. Metrics: `used_memory_hash_templates`, `hash_templates`, `hash_template_keys`.
  https://redis.io/docs/latest/develop/data-types/hashes/ and https://redis.io/docs/latest/commands/info/

## Slow commands

- Redis runs commands on one main thread: any slow command (`KEYS`, `SORT`, `LREM`, `SUNION`, big intersections) blocks every client. High CPU with low traffic usually means slow commands.
  https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/latency/
- Check the documented complexity before using a command on big values. Move unavoidable O(N) analysis to a replica.
  https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/latency/
- Deleting, expiring or evicting a very large object blocks: split it into smaller keys, or use `UNLINK`.
  https://redis.io/docs/latest/commands/latency-doctor/ and https://redis.io/docs/latest/commands/unlink/
- `HOTKEYS` (8.6) finds keys with the highest CPU and network use.
  https://redis.io/docs/latest/develop/whats-new/8-6/ and https://redis.io/docs/latest/develop/clients/redis-py/produsage/

## Latency sources

- `fork()` for RDB or AOF rewrite stalls the main thread, more with bigger memory. Check `latest_fork_usec`. Some Xen VMs (old EC2 instance types) fork 1 to 2 orders of magnitude slower, so use HVM instances.
  https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/latency/
- Transparent huge pages cause latency and memory spikes after fork: `echo never > /sys/kernel/mm/transparent_hugepage/enabled`, then restart Redis. Set `vm.overcommit_memory = 1` so background saves can fork.
  https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/latency/ and https://redis.io/docs/latest/operate/oss_and_stack/management/admin/
- Swapping causes erratic latency. Check `vmstat 1` and the `Swap:` lines of `/proc/<pid>/smaps`.
  https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/latency/
- AOF: `appendfsync everysec` is the compromise. `no-appendfsync-on-rewrite yes` relieves the disk during rewrites. Keep other I/O-heavy processes off the host.
  https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/latency/
- Do not pin Redis to one core: background save children must not share the event loop's core.
  https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/latency/
- Keep connections long-lived. On the same host, Unix sockets cut latency (about 30 us versus 200 us over 1 Gbit/s TCP in the doc's figures).
  https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/latency/

## Diagnosing latency

- Start with the Slow Log, which measures execution only (no network I/O). Entry IDs never reset until restart, so use them to avoid reprocessing. Arguments are truncated (32 by default), and 8.10 adds the original argument count as a 7th field.

```
CONFIG SET slowlog-log-slower-than 1000     # microseconds
SLOWLOG GET 10
```
  https://redis.io/docs/latest/commands/slowlog-get/
- The latency monitor is off by default: `CONFIG SET latency-monitor-threshold 100` (ms), then `LATENCY LATEST`, `LATENCY DOCTOR`.
  https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/latency-monitor/
- Baseline the host with `redis-cli --intrinsic-latency 100`, run on the server itself.
  https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/latency/
- `CLIENT LIST` is O(N) and flagged dangerous. The software watchdog is experimental and a last resort.
  https://redis.io/docs/latest/commands/client-list/ and https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/latency/

## Benchmarking

- A synchronous loop measures your network and client, not Redis. `redis-benchmark` defaults to 50 clients and no pipelining: set `-P` to the app's real pipeline depth and `-r` to a realistic key space.

```
redis-benchmark -n 1000000 -t set,get -P 16 -q
```
  https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/benchmarks/
- Avoid `MONITOR` during a benchmark: it impacts measured performance significantly. Throughput falls with connection count (30,000 connections give about half the throughput of 100).
  https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/benchmarks/

## Old patterns

| Old | Current | Source |
| --- | --- | --- |
| `hash-max-ziplist-*`, `zset-max-ziplist-*` | `*-max-listpack-*` (7.0) | https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/memory-optimization/ |
| `ziplist` encoding (`linkedlist` and `zipmap` are no longer used) | `listpack` (7.0, sets 7.2) | https://redis.io/docs/latest/commands/object-encoding/ |
| `used_memory_lua` | `used_memory_vm_eval` (7.0) | https://redis.io/docs/latest/commands/info/ |
