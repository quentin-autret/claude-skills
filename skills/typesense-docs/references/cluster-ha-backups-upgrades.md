# Clusters, high availability, backups, upgrades and sizing

Raft clusters, the nodes file, recovery playbooks, cluster operations, monitoring, backups, rolling upgrades, sizing and CI testing for Typesense 30.2.

## Contents

- How a cluster works
- Nodes file
- Verifying a cluster
- Recovery
- Cluster operations API
- Monitoring
- Backups
- Upgrades
- Sizing
- Testing in CI

## How a cluster works

- Raft consensus. Every node holds the whole dataset. A node serves reads itself and forwards writes to the leader.
- 3 nodes tolerate 1 failure, 5 nodes tolerate 2 with slightly slower writes. N nodes tolerate `(N-1)/2` failures. The minimum is 3 nodes.
- A successful write is on a majority of nodes and searchable at once. No polling.
- Typesense Cloud HA runs 3 nodes by default and 3 to 7 nodes selectable when launching or modifying a cluster, behind a load balancer. HA cannot be turned off once enabled.

https://typesense.org/docs/guide/high-availability.html, https://typesense.org/docs/guide/updating-typesense.html and https://typesense.org/docs/guide/migrating-from-algolia.html

## Nodes file

- One line, `<peering_address>:<peering_port>:<api_port>` per node, comma separated, identical on every node. Values match each node's own flags. Every node shares the same bootstrap `--api-key`.
- Changes are picked up without restart, within 30s.
- Docker networking must let containers reach each other by IP.

```shell
echo '192.168.12.1:8107:8108,192.168.12.2:8107:8108,192.168.12.3:8107:8108' | sudo tee /etc/typesense/nodes
typesense-server --data-dir /var/lib/typesense --api-key=xyz \
  --api-address 0.0.0.0 --api-port 8108 \
  --peering-address 192.168.12.1 --peering-port 8107 \
  --nodes=/etc/typesense/nodes
```
https://typesense.org/docs/guide/high-availability.html

## Verifying a cluster

`GET /debug` on each node (needs a key): exactly one `state: 1` (leader), the others `state: 4` (follower). Two leaders mean the cluster formed wrongly: check `typesense.log` and the HTTP and peering ports.
https://typesense.org/docs/guide/high-availability.html and https://typesense.org/docs/30.2/api/cluster-operations.html

## Recovery

- A failed node returning with the same IP rejoins and catches up alone, answering 503 meanwhile.
- After a long outage, replay is slow because the leader pauses snapshots. Playbook:
  1. Stop the unhealthy follower.
  2. Remove its IP from the nodes file on healthy nodes.
  3. Trigger `/operations/snapshot` on the leader WITHOUT a path, which compacts the Raft log.
  4. Clear the follower's data dir.
  5. Add its IP back on every node.
  6. Start it. It loads the leader's snapshot.
- Lost quorum (more than `(N-1)/2` down): reads AND writes stop to avoid split brain. Recover by editing one node's file to list only itself, waiting for `/health` ok, then adding nodes one by one to every file (clearing a lagging node's data dir first).
- Full disaster only: start one node from a snapshot, then add empty nodes one at a time.

https://typesense.org/docs/guide/high-availability.html

## Cluster operations API

| Call | Effect |
| --- | --- |
| `POST /operations/snapshot` with `snapshot_path` | writes a snapshot to a directory ON THE SERVER, without path an internal Raft compaction only |
| `POST /operations/db/compact` | RocksDB compaction, non-blocking, run off-peak |
| `POST /operations/vote` | the called follower starts a Raft vote and becomes the new leader once it succeeds |
| `POST /operations/cache/clear` | clears the `use_cache` response cache |
| `POST /config` | toggles `log-slow-requests-time-ms` at runtime (default `-1` = off) |
| `GET /operations/schema_changes` | progress of a running alter (documented on the collections page) |

```js
client.operations.perform('snapshot', { snapshot_path: '/tmp/typesense-data-snapshot' })
```
https://typesense.org/docs/30.2/api/cluster-operations.html and https://typesense.org/docs/30.2/api/collections.html

## Monitoring

- `GET /health` returns `{"ok": true}`, plus `resource_error` (`OUT_OF_DISK` or `OUT_OF_MEMORY`) when resources run out.
- `GET /metrics.json`: CPU, disk, RAM, network. `typesense_memory_allocated_bytes` measures real use, and `fragmentation_ratio` near 0 is good.
- `GET /stats.json`: requests per second and latency per endpoint over the last 10 seconds only.
- Keep RAM under 85% and average CPU under 90%. Configure a little swap, and add RAM if it is ever used.

https://typesense.org/docs/30.2/api/cluster-operations.html and https://typesense.org/docs/guide/running-in-production.html

## Backups

- Self-hosted only. Archiving the live data dir is unsafe.
- Back up: snapshot to a directory, then `tar -czvf backup.tar.gz -C /tmp/typesense-data-snapshot .`.
- Restore: stop Typesense, empty the data dir, extract the archive into it, start. The in-memory index rebuilds from disk, so startup time grows with the dataset.

https://typesense.org/docs/guide/backups.html

## Upgrades

- Read the release notes of every version in between. Changed defaults alter results. Replay a fixed set of test queries on staging.
- Self-hosted rolling upgrade: first trigger a snapshot on the leader as a backup.
- Install the new binary and restart. Docker: stop the container, `docker run` the new tag with the same arguments.
- Rolling HA upgrade: followers first, one at a time, each waiting for `/health` 200. The leader stays on the old version until last. Stop it so another node is elected, upgrade it, and it rejoins as follower.
- `kill <pid>` shuts down gracefully.
- Typesense Cloud: a single-node non-HA upgrade means about 5 to 60 minutes of downtime. HA clusters upgrade one node at a time with no downtime, once clients list all hostnames. Supported changes: server version (upgrade, or downgrade when a version supports it), RAM, CPU, enabling or disabling High Performance disk and GPU acceleration, enabling HA. Anything else needs a new cluster and a reindex.

https://typesense.org/docs/guide/updating-typesense.html

## Sizing

- Keyword RAM: 2X to 3X the size of indexed field values. Unindexed fields cost disk only. Idle process: about 20 MB.
- Vectors: `7 bytes * dimensions * records`. Built-in embedding models add 2 GB to 6 GB.
- CPU: at least 2 vCPU. Burst vCPU (Cloud) does not suit a high baseline of traffic or high indexing volume.
- Disk: at least the raw dataset size, SSD recommended.

https://typesense.org/docs/guide/system-requirements.html

## Testing in CI

- Run a real server: Testcontainers (Java module `org.testcontainers:typesense`) or the Docker image in the CI job.
- Poll `/health` until `"ok":true` with a timeout, never a fixed sleep.
- Quote matrix versions as strings (`'28.0'`), or YAML reads `26.0` as the number 26.

```yaml
- name: Wait for Typesense to be healthy
  shell: bash
  run: |
    timeout=30
    counter=0
    until curl -s http://localhost:8108/health | grep -q '"ok":true'; do
      if [ $counter -eq $timeout ]; then exit 1; fi
      sleep 1
      counter=$((counter + 1))
    done
```
https://typesense.org/docs/guide/github-actions.html and https://typesense.org/docs/guide/testcontainers.html
