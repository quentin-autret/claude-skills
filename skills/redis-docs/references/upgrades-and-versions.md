# Upgrading to Redis 8 and choosing a version

Upgrade procedures, the Redis 8 packaging change, breaking changes that bite on upgrade, and the release support model of Redis Open Source.

## Contents

- Release model
- What Redis 8 changed
- Breaking changes to check
- Standalone upgrade
- Replicated, Sentinel and Cluster upgrades
- Notable additions by release

## Release model

- Use the latest available version. Standard releases (8.0, 8.4, 8.6, 8.8, 8.10) get fixes until 6 months after the next minor. Extended releases (6.2, 7.2, 7.4, 8.2) get 5 years.
  https://redis.io/docs/latest/operate/oss_and_stack/install/version-mgmt/
- A feature from a later feature set needs the matching Redis version and module components.
  https://redis.io/docs/latest/develop/whats-new/redis-feature-sets/

## What Redis 8 changed

- Redis Stack and the community edition merged into "Redis Open Source": JSON, time series, probabilistic types, Search and vector sets ship in the server, with no separate module install.
  https://redis.io/docs/latest/develop/whats-new/8-0/
- Module settings moved to `redis.conf` and `CONFIG SET` with prefixed names (`search-*`, `ts-*`, `bf-*`, `cf-*`). Old names are still accepted but deprecated, and `CONFIG` needs the new ones.
  https://redis.io/docs/latest/develop/ai/search-and-query/administration/configuration/ and https://redis.io/docs/latest/develop/data-types/timeseries/configuration/ and https://redis.io/docs/latest/develop/data-types/probabilistic/configuration/
- 8.10 ships one `redis.conf`. 8.0 to 8.8 also shipped `redis-full.conf`.
  https://redis.io/docs/latest/operate/oss_and_stack/management/config/
- Supported upgrade path: from Redis Open Source 7.x (with or without modules) and Redis Stack 7.2 or 7.4.
  https://redis.io/docs/latest/operate/oss_and_stack/install/upgrade/standalone/

## Breaking changes to check

- ACL: module commands now belong to standard categories (`@read`, `@write`). `+@all -@write` no longer allows `JSON.SET`. Include the new categories explicitly.
  https://redis.io/docs/latest/develop/whats-new/8-0/
- Search 8.0: `LIMIT` offset must be 0 when the limit is 0, stricter `FT.CURSOR READ` and `FT.ALIASADD` parsing, parentheses needed for exponentiation in `APPLY`, invalid input errors instead of empty results, reducer defaults revisited, default scorer `BM25` instead of TF-IDF.
  https://redis.io/docs/latest/develop/whats-new/8-0/
- Search 8.4: `BM25` renamed `BM25STD`, which is the default scorer for text and tag.
  https://redis.io/docs/latest/develop/whats-new/8-4/ and https://redis.io/docs/latest/develop/ai/search-and-query/administration/overview/
- JSONPath 8.10: `~` is no longer valid in a dot-notation field name, and `in`, `nin`, `size`, `sizeof`, `empty`, `subsetof`, `anyof`, `noneof` are reserved. Use `$["a~b"]` or `$["size"]`.
  https://redis.io/docs/latest/develop/data-types/json/path/
- INFO 8.0: Search fields renamed (`search_bytes_collected` became `search_gc_bytes_collected`). Parse INFO tolerantly.
  https://redis.io/docs/latest/commands/info/
- From 7.2 (relevant when coming from a 7.x older than 7.2): RDB files use format 11, which is incompatible with older versions, Lua `print()` is no longer supported, `PFCOUNT` and `PUBLISH` are blocked in read-only scripts.
  https://redis.io/docs/latest/develop/whats-new/7-2/ and https://redis.io/docs/latest/develop/whats-new/redis-feature-sets/
- Vector sets were beta in 8.0, with APIs that could change.
  https://redis.io/docs/latest/develop/whats-new/8-0/

## Standalone upgrade

Rehearse in a controlled environment first (Docker is suggested).
https://redis.io/docs/latest/operate/oss_and_stack/install/upgrade/standalone/

```
redis-cli SAVE
redis-cli CONFIG GET dir        # copy this directory
# stop, install Redis 8, start
redis-cli INFO server | grep redis_version
```
Restore the copy if data is missing.
https://redis.io/docs/latest/operate/oss_and_stack/install/upgrade/standalone/

## Replicated, Sentinel and Cluster upgrades

- Sentinel or Cluster: upgrade one replica after the other, perform a manual failover to promote one of the upgraded replicas to primary, then promote the last replica.
  https://redis.io/docs/latest/operate/oss_and_stack/management/admin/
- Cluster: one node at a time, replicas first. For a primary, run `CLUSTER FAILOVER` on its replica, wait for the primary to become a replica, then upgrade it. Check with `cluster info` and `redis-cli --cluster check`.
  https://redis.io/docs/latest/operate/oss_and_stack/management/scaling/ and https://redis.io/docs/latest/operate/oss_and_stack/install/upgrade/cluster/
- Redis Search is not fully operational until every primary runs Redis 8: queries can error or report "Waiting for all nodes to connect".
  https://redis.io/docs/latest/operate/oss_and_stack/install/upgrade/cluster/
- Restart replicas with `SHUTDOWN` so they can partial-resync. Do not configure an instance as a writable replica as an intermediary step when upgrading a set of instances.
  https://redis.io/docs/latest/operate/oss_and_stack/management/replication/
- Zero-downtime move to a new instance: start it as a replica (on a different port if on the same server), wait for the initial sync, compare key counts with `INFO`, allow writes on the replica with `CONFIG SET slave-read-only no`, repoint clients (`CLIENT PAUSE` on the old primary helps), confirm the old primary receives no queries, then `REPLICAOF NO ONE` and shut the old primary down.
  https://redis.io/docs/latest/operate/oss_and_stack/management/admin/

## Notable additions by release

| Release | Additions that change the recommended way | Source |
| --- | --- | --- |
| 8.0 | `HGETEX`, `HSETEX`, `HGETDEL`, vector sets (beta), module config via `CONFIG SET` | https://redis.io/docs/latest/develop/whats-new/8-0/ |
| 8.2 | `XDELEX`, `XACKDEL`, `ACKED`/`DELREF`/`KEEPREF`, SVS-VAMANA vector index, `CLUSTER SLOT-STATS` | https://redis.io/docs/latest/develop/whats-new/8-2/ and https://redis.io/docs/latest/develop/ai/search-and-query/vectors/ |
| 8.4 | `SET ... IFEQ`, `DELEX`, `DIGEST`, `MSETEX`, `XREADGROUP ... CLAIM`, `FT.HYBRID`, atomic slot migration | https://redis.io/docs/latest/develop/whats-new/8-4/ and https://redis.io/docs/latest/commands/set/ |
| 8.6 | Stream idempotency (`IDMP`), `allkeys-lrm`/`volatile-lrm`, `HOTKEYS`, time series NaN | https://redis.io/docs/latest/develop/whats-new/8-6/ |
| 8.8 | `INCREX`, Array type, `XNACK`, hash subkey notifications, `JSON.SET ... FPHA` | https://redis.io/docs/latest/develop/whats-new/8-8/ |
| 8.10 | Compact hashes and `HIMPORT`, `BACKUP`, `LMOVEM`, `SUNIONCARD`/`SDIFFCARD`, `XREADGROUP MAXCOUNT/MAXSIZE`, JSONPath extensions, `TS.NRANGE`/`TS.READ` | https://redis.io/docs/latest/develop/whats-new/8-10/ |
