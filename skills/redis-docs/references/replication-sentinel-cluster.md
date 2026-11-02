# Replication, Sentinel and Cluster

Replica behavior and data-loss windows, Sentinel deployment and failover, Cluster hash slots, redirects, multi-key rules and operations in self-managed Redis Open Source 8.10.

## Contents

- Replication
- Bounding write loss
- Sentinel deployment
- Sentinel failover behavior
- Cluster topology
- Hash slots and multi-key commands
- Cluster clients
- Cluster operations
- Old patterns

## Replication

- Replicas stay read-only (the default). Writable replicas exist for historical reasons, diverge from the primary, and since 7.0 every former use has a replacement (`SUNION` instead of `SUNIONSTORE`, `SORT_RO`, `EVAL_RO`).
  https://redis.io/docs/latest/operate/oss_and_stack/management/replication/
- Replicas do not expire keys: they wait for the primary's DEL but hide logically expired keys on reads. A replica can show fewer keys than its primary.
  https://redis.io/docs/latest/operate/oss_and_stack/management/replication/ and https://redis.io/docs/latest/develop/get-started/faq/
- An authenticated primary needs `masterauth` on replicas. Behind NAT or Docker, set `replica-announce-ip`/`replica-announce-port`.
  https://redis.io/docs/latest/operate/oss_and_stack/management/replication/
- Size the replication backlog in proportion to memory so reconnecting replicas resync partially instead of fully.
  https://redis.io/docs/latest/operate/oss_and_stack/management/admin/
- Restart a replica with `SHUTDOWN` (it saves the data needed for a partial resync). A replica restarted from AOF cannot partial-sync.
  https://redis.io/docs/latest/operate/oss_and_stack/management/replication/
- Loading a full sync blocks the replica's clients, possibly for seconds.
  https://redis.io/docs/latest/operate/oss_and_stack/management/replication/
- A primary without disk use but with replicas: enable `repl-diskless-sync`, otherwise syncs still write RDB files.
  https://redis.io/docs/latest/operate/oss_and_stack/management/admin/
- Read-only replicas still accept `DEBUG` and `CONFIG`: never expose them.
  https://redis.io/docs/latest/operate/oss_and_stack/management/replication/

## Bounding write loss

- Replication is asynchronous: Sentinel and Cluster can lose acknowledged writes on failover (last failover wins). `WAIT` narrows the window but does not remove it.
  https://redis.io/docs/latest/operate/oss_and_stack/management/sentinel/ and https://redis.io/docs/latest/operate/oss_and_stack/management/scaling/
- `min-replicas-to-write N` + `min-replicas-max-lag S` stop a primary from accepting writes when fewer than N replicas have a lag under S seconds. Writes fail with `NOREPLICAS` when replicas are missing or lagging.

```
min-replicas-to-write 1
min-replicas-max-lag 10
```
  https://redis.io/docs/latest/operate/oss_and_stack/management/replication/ and https://redis.io/docs/latest/develop/reference/errors/common/noreplicas/

## Sentinel deployment

- At least 3 Sentinels on machines or zones that fail independently, never 2. Each needs a writable config file and port 26379 open between Sentinels.
  https://redis.io/docs/latest/operate/oss_and_stack/management/sentinel/
- Clients must support Sentinel and ask it for the primary. Never hardcode the primary address.
  https://redis.io/docs/latest/operate/oss_and_stack/management/sentinel/

```
sentinel monitor mymaster 127.0.0.1 6379 2
sentinel down-after-milliseconds mymaster 60000
SENTINEL CKQUORUM mymaster
```
  https://redis.io/docs/latest/operate/oss_and_stack/management/sentinel/
- `SENTINEL MONITOR` takes an IP. Hostnames are off by default: `resolve-hostnames` makes Sentinel accept them (reliable, fast DNS required), `announce-hostnames` makes it announce hostnames instead of IPs, which some clients reject.
  https://redis.io/docs/latest/operate/oss_and_stack/management/sentinel/
- Docker port remapping breaks Sentinel discovery: map ports 1:1 or use host networking.
  https://redis.io/docs/latest/operate/oss_and_stack/management/sentinel/
- Runtime config changes do not propagate between Sentinels: apply them to all. Sentinels never forget peers: after stopping a Sentinel, run `SENTINEL RESET *` on each other Sentinel, at least 30 s apart. To drop a replica for good, run `SENTINEL RESET <master-name>` on every Sentinel.
  https://redis.io/docs/latest/operate/oss_and_stack/management/sentinel/
- `replica-priority` steers promotion (lower wins, 0 never). Set it on every instance, since roles change.
  https://redis.io/docs/latest/operate/oss_and_stack/management/sentinel/
- Sentinel's ACL user gets only the channel `&__sentinel__:hello` and the commands it needs. Hello messages are unauthenticated, so users able to publish there can redirect Sentinel.
  https://redis.io/docs/latest/operate/oss_and_stack/management/sentinel/

## Sentinel failover behavior

- Quorum only detects failure. Failover also needs a majority of Sentinels, so a minority partition never fails over.
  https://redis.io/docs/latest/operate/oss_and_stack/management/sentinel/
- A clock jump or busy host puts Sentinel in TILT mode: it stops acting until everything looks normal for 30 s.
  https://redis.io/docs/latest/operate/oss_and_stack/management/sentinel/
- A long Lua script on the primary makes Sentinel try `SCRIPT KILL` first, which works only for read-only scripts.
  https://redis.io/docs/latest/operate/oss_and_stack/management/sentinel/
- Test failover regularly (for example `DEBUG sleep 30` on a dev primary).
  https://redis.io/docs/latest/operate/oss_and_stack/management/sentinel/

## Cluster topology

- 16384 hash slots, slot = CRC16(key) mod 16384. Minimum 3 primaries, recommended 3 primaries + 3 replicas, `appendonly yes` on nodes.
  https://redis.io/docs/latest/operate/oss_and_stack/reference/cluster-spec/ and https://redis.io/docs/latest/operate/oss_and_stack/management/scaling/
- Open the data port and the bus port (data port + 10000) between all nodes. NAT and remapped ports are not supported: use host networking.
  https://redis.io/docs/latest/operate/oss_and_stack/management/scaling/
- Default `cluster-require-full-coverage yes`: one uncovered slot stops the whole cluster. A primary failing together with its replica stops it too.
  https://redis.io/docs/latest/develop/reference/errors/common/clusterdown/ and https://redis.io/docs/latest/operate/oss_and_stack/management/scaling/
- Only database 0 exists, `SELECT` is refused.
  https://redis.io/docs/latest/operate/oss_and_stack/reference/cluster-spec/

## Hash slots and multi-key commands

- Multi-key commands, transactions and scripts need all keys in one slot, or fail with `CROSSSLOT`. This covers `MGET`, `MSET`, `DEL`, `EXISTS`, `RENAME`, `SINTER`, `*STORE`, `LMOVE`, `WATCH`, `XREAD`, blocking pops, pipelines with keys.
  https://redis.io/docs/latest/develop/using-commands/multi-key-operations/
- Only the text between the first `{` and the first `}` after it is hashed, and an empty pair means the whole key is hashed: `foo{}{bar}` hashes the whole key, `foo{{bar}}zap` hashes `{bar`.

```
MSET {user:1000}.name Angela {user:1000}.surname White
```
  https://redis.io/docs/latest/operate/oss_and_stack/reference/cluster-spec/
- Exceptions to watch: `JSON.MGET` across slots returns partial results with no error. `KEYS`, `SCAN`, `DBSIZE`, `FLUSHALL` act on the current shard only.
  https://redis.io/docs/latest/develop/using-commands/multi-key-operations/
- During resharding, multi-key commands on split or missing keys return `TRYAGAIN`.
  https://redis.io/docs/latest/operate/oss_and_stack/reference/cluster-spec/

## Cluster clients

- A cluster client caches the slot map. On `MOVED` it can refetch the whole map with `CLUSTER SHARDS`. On `ASK` it sends `ASKING` plus the command to the target once, without updating the map.
  https://redis.io/docs/latest/operate/oss_and_stack/reference/cluster-spec/
- Replica reads: `READONLY` on the connection, accepting stale data.
  https://redis.io/docs/latest/operate/oss_and_stack/reference/cluster-spec/

## Cluster operations

- Use `redis-cli --cluster create/add-node/reshard/check/fix/del-node`. Since 8.10 reshard and rebalance use server-side atomic slot migration (`CLUSTER MIGRATION`, 8.4).

```
redis-cli --cluster create 127.0.0.1:7000 ... 127.0.0.1:7005 --cluster-replicas 1
redis-cli --cluster check 127.0.0.1:7000
```
  https://redis.io/docs/latest/operate/oss_and_stack/management/scaling/, https://redis.io/docs/latest/develop/whats-new/8-10/ and https://redis.io/docs/latest/develop/whats-new/8-4/
- `nodes.conf` is machine-written: never edit it.
  https://redis.io/docs/latest/operate/oss_and_stack/management/scaling/
- Remove a primary only once empty. For a dead node `del-node` fails: run `redis-cli --cluster call <host:port> cluster forget <node-id>`.
  https://redis.io/docs/latest/operate/oss_and_stack/management/scaling/
- Extra replicas on some primaries migrate to orphaned ones (`cluster-migration-barrier`).
  https://redis.io/docs/latest/operate/oss_and_stack/management/scaling/
- Moving to Cluster has no live migration: load each primary's AOF into the cluster then `--cluster fix`, or use `redis-cli --cluster import`.
  https://redis.io/docs/latest/operate/oss_and_stack/management/scaling/

## Old patterns

| Old | Current | Source |
| --- | --- | --- |
| `CLUSTER SLOTS` | `CLUSTER SHARDS` | https://redis.io/docs/latest/operate/oss_and_stack/reference/cluster-spec/ |
| `SYNC` | `PSYNC` (partial resync) | https://redis.io/docs/latest/operate/oss_and_stack/management/replication/ |
| Writable replica for temporary results | `SUNION`, `ZINTER`, `SORT_RO`, `EVAL_RO` (7.0) | https://redis.io/docs/latest/operate/oss_and_stack/management/replication/ |
| Sentinel password-only auth | `sentinel auth-user` / `auth-pass` with ACL users (6.0) | https://redis.io/docs/latest/operate/oss_and_stack/management/sentinel/ |
| "slave" option names | "replica" names, except fixed protocol strings | https://redis.io/docs/latest/operate/oss_and_stack/management/sentinel/ |
