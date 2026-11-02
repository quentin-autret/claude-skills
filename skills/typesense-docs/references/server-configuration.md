# Server configuration

Server flags, their sources and priority, defaults that surprise, logging, TLS, resource limits and install notes for a self-hosted Typesense 30.2.

## Contents

- Configuration sources
- Network, CORS and TLS
- Logging
- Resource limits and write rejection
- Search limits
- Snapshots, compaction and recovery flags
- Analytics flags
- Install notes

## Configuration sources

- Required: `--api-key` (the bootstrap admin key) and `--data-dir`. `--config` points to an INI file.
- Priority: command line first, then the config file, then environment variables.
- Environment variables: `TYPESENSE_` prefix, upper case, underscores: `--data-dir` becomes `TYPESENSE_DATA_DIR`.
- The INI file has a `[server]` section and keys without `--`. DEB and RPM packages install it at `/etc/typesense/typesense-server.ini`, with an auto-generated admin key.

```ini
[server]
api-key = Rhsdhas2asasdasj2
data-dir = /var/lib/typesense
log-dir = /var/log/typesense
api-port = 9090
```
```bash
TYPESENSE_DATA_DIR=/var/lib/typesense TYPESENSE_API_KEY=AS3das2awQ2 ./typesense-server
```
https://typesense.org/docs/30.2/api/server-configuration.html and https://typesense.org/docs/guide/install-typesense.html

- Typesense Cloud manages these values.
  https://typesense.org/docs/30.2/api/server-configuration.html

## Network, CORS and TLS

- `--api-address` defaults to `0.0.0.0`, `--api-port` to `8108`. `--peering-port` defaults to `8107`.
- `--peering-address` must be a private internal IP: Raft traffic between nodes is unencrypted. If omitted, Typesense picks the first internal IP. `--peering-subnet` narrows the choice.
- Browser access needs `--enable-cors`. `--cors-domains` takes a comma list without trailing slashes: `https://example.com,https://example2.com`.
- HTTPS needs both `--ssl-certificate` and `--ssl-certificate-key`. Certificates reload from disk every 8 hours (`--ssl-refresh-interval-seconds`).
- Production: `--api-port 443` with HTTPS, firewall everything else inbound. The built-in HTTP server can face public traffic without Nginx in front.

https://typesense.org/docs/30.2/api/server-configuration.html, https://typesense.org/docs/guide/running-in-production.html and https://typesense.org/docs/guide/high-availability.html

## Logging

| Flag | Default | Effect |
| --- | --- | --- |
| `--log-dir` | stdout and stderr | writes `typesense.log` |
| `--enable-access-logging` | `false` | `typesense-access.log` with requests and IPs |
| `--enable-search-logging` | `false` | logs each search payload at request start |
| `--log-slow-requests-time-ms` | `-1` (off) | any slow request, changeable at runtime with `POST /config` |
| `--log-slow-searches-time-ms` | `30000` (on) | slow `/documents/search` and `/multi_search`, `-1` disables |

https://typesense.org/docs/30.2/api/server-configuration.html and https://typesense.org/docs/30.2/api/cluster-operations.html

## Resource limits and write rejection

- `--memory-used-max-percentage` and `--disk-used-max-percentage` default to `100`, so writes are never rejected before the host runs out. Lower them to reject writes early.
- `--healthy-read-lag` (default 1000) and `--healthy-write-lag` (default 500): a node lagging beyond them rejects reads or writes with 503. Tune last, after CPU, timeouts and bulk writes.
- `--thread-pool-size` defaults to `NUM_CORES * 8`. `--num-collections-parallel-load` to `NUM_CORES * 4`, `--num-documents-parallel-load` to 1000.
- `--cache-num-entries` (default 1000) sizes the search response cache. `--embedding-cache-num-entries` (default 100) caches remote embeddings.
- RocksDB flags (`--db-write-buffer-size`, `--max-indexing-concurrency` default 4) are for specific write-performance cases only.

https://typesense.org/docs/30.2/api/server-configuration.html and https://typesense.org/docs/guide/syncing-data-into-typesense.html

## Search limits

`--filter-by-max-ops` (default 100), `--max-per-page` (default 250), `--max-group-limit` (default 99).
https://typesense.org/docs/30.2/api/server-configuration.html

## Snapshots, compaction and recovery flags

- `--snapshot-interval-seconds` defaults to 3600. More frequent snapshots speed cold start, but too low a value on a large dataset slows follower recovery.
- `--db-compaction-interval` defaults to 7 days. Consider 24 hours when collections are dropped and recreated often.
- `--skip-writes` starts without replaying the Raft log, to step over recent writes that crashed the server. Temporary use only.
- `--reset-peers-on-error` force-resets peers on an irrecoverable clustering error. It can lose data, last resort only.
- `--nodes` points to the cluster nodes file.

https://typesense.org/docs/30.2/api/server-configuration.html

## Analytics flags

- `--enable-search-analytics` (default `false`) and `--analytics-dir` are both required for analytics.
- `--analytics-flush-interval` defaults to 3600s, minimum 60.
- `--analytics-minute-rate-limit` defaults to 5 events per minute.
- The official CI example for analytics tests sets `--analytics-flush-interval=60 --analytics-minute-rate-limit=100`.

https://typesense.org/docs/30.2/api/server-configuration.html, https://typesense.org/docs/30.2/api/analytics-query-suggestions.html and https://typesense.org/docs/guide/github-actions.html

## Install notes

- Docker: pin the image tag, mount a volume on `/data`, pass `--data-dir /data`.

```yaml
services:
  typesense:
    image: typesense/typesense:30.2
    restart: on-failure
    ports:
      - "8108:8108"
    volumes:
      - ./typesense-data:/data
    command: '--data-dir /data --api-key=xyz --enable-cors'
```
https://typesense.org/docs/guide/install-typesense.html

- The key `xyz` and `--enable-cors` appear in the docs samples, so set your own key. Homebrew also defaults to key `xyz` and port 8108.
- Linux needs Ubuntu 20 or later. The native macOS binary needs Ventura (13.x), otherwise use Docker.
- ARM hosts with 16K memory pages need the `lg-page16` build, for example `typesense/typesense:30.2-arm64-lg-page16`.
- GPU (Nvidia only) needs CUDA 11.8, cuDNN 8 and `typesense-gpu-deps`. It helps generating embeddings for vector search and voice queries, and Typesense uses available Nvidia GPUs automatically.
- Kubernetes support comes from community operators only.
- Health: `curl http://localhost:8108/health` returns `{"ok":true}` when the server is ready.

https://typesense.org/docs/guide/install-typesense.html and https://typesense.org/docs/guide/system-requirements.html
