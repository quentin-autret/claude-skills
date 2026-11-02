# node-redis and ioredis

Connection lifecycle, error events, the offline queue, pipelines, WATCH with pools, Sentinel, caching and the ioredis to node-redis migration, from the Redis Node.js client docs.

## Contents

- Which client
- Connecting and closing
- Errors, reconnection and timeouts
- Pipelines and transactions
- Pub/Sub and scanning
- Cluster and Sentinel
- Client-side caching
- Search, vectors and probabilistic types
- Migrating from ioredis

## Which client

- Default: node-redis (npm `redis`). The migration page calls ioredis deprecated in favor of node-redis, while the ioredis page says it is still maintained: no ioredis in new projects, no forced migration.
  https://redis.io/docs/latest/develop/clients/nodejs/migration/ and https://redis.io/docs/latest/develop/clients/ioredis/

## Connecting and closing

- node-redis does not connect on creation: `await client.connect()`. ioredis connects in its constructor.

```js
import { createClient } from 'redis';
const client = createClient({ url: 'redis://alice:foobared@host:6380' });
client.on('error', err => console.log('Redis Client Error', err));
await client.connect();
```
  https://redis.io/docs/latest/develop/clients/nodejs/connect/ and https://redis.io/docs/latest/develop/clients/nodejs/migration/
- Close with `client.close()` or `client.destroy()`. ioredis uses `disconnect()`.
  https://redis.io/docs/latest/develop/clients/nodejs/connect/ and https://redis.io/docs/latest/develop/clients/ioredis/
- `isOpen` means the socket is open, `isReady` means commands can be sent. The `end` event means the client was intentionally closed with `close()`.
  https://redis.io/docs/latest/develop/clients/nodejs/connect/
- TLS: `socket: { tls: true, key, cert, ca }` with `username` and `password`. ioredis takes a `tls` object.
  https://redis.io/docs/latest/develop/clients/nodejs/connect/ and https://redis.io/docs/latest/develop/clients/ioredis/connect/
- node-redis ignores all query parameters in `redis://` and `rediss://` URLs. ioredis turns any non-redis scheme into a host (`unix:///x` becomes host `unix`).
  https://redis.io/docs/latest/develop/clients/connection-urls/
- Hash reads return strings: `hGetAll` gives `"age": "29"`.
  https://redis.io/docs/latest/develop/clients/nodejs/connect/

## Errors, reconnection and timeouts

- Register `on('error')` on every client, cluster and sentinel. Without a listener the EventEmitter throws and can crash the process.
  https://redis.io/docs/latest/develop/clients/nodejs/error-handling/ and https://redis.io/docs/latest/develop/clients/nodejs/connect/
- Reconnection uses exponential backoff with jitter by default. Override through `socket.reconnectStrategy` (`false`, a delay in ms, or `(retries, cause) => ...`).
  https://redis.io/docs/latest/develop/clients/nodejs/connect/
- Offline queue trap: while disconnected, node-redis queues commands and replays them after reconnect, so a non-idempotent command cut mid-flight can run twice. Set `disableOfflineQueue: true`, or send those commands on a second connection with the queue off.
  https://redis.io/docs/latest/develop/clients/nodejs/produsage/
- `socket.connectTimeout` defaults to 5 s. Per-command timeouts use an `AbortController`.

```js
const ac = new AbortController(); setTimeout(() => ac.abort(), 1000);
await client.get(commandOptions({ signal: ac.signal }), key);
```
  https://redis.io/docs/latest/develop/clients/nodejs/produsage/
- Retry `ECONNREFUSED`, `ETIMEDOUT`, `ECONNRESET`, `EAI_AGAIN` and `BUSY`/`TRYAGAIN`/`LOADING` replies with bounded backoff. Fail fast on `WRONGTYPE`.
  https://redis.io/docs/latest/develop/clients/nodejs/error-handling/

## Pipelines and transactions

- Commands issued in the same tick are pipelined automatically, so `Promise.all([...])` is one round trip. `multi()....execAsPipeline()` pipelines without a transaction, `multi()....exec()` runs MULTI/EXEC.
  https://redis.io/docs/latest/develop/clients/nodejs/transpipe/
- On connection loss, `Promise.all` resumes from the interruption point while a `multi()....execAsPipeline()` pipeline discards the remaining commands.
  https://redis.io/docs/latest/develop/clients/nodejs/transpipe/
- WATCH in a server with concurrent requests: run it in `createClientPool()` + `pool.execute(...)`, since WATCH is tracked per connection. Read the watched keys on the client, queue writes on `multi()`, retry on `WatchError`.

```js
await pool.execute(async client => {
  await client.watch('key');
  return client.multi().get('key').exec();
});
```
  https://redis.io/docs/latest/develop/clients/nodejs/transpipe/

## Pub/Sub and scanning

- A subscribed connection cannot run other commands: `const sub = client.duplicate(); await sub.connect(); await sub.subscribe(ch, cb)`.
  https://redis.io/docs/latest/develop/clients/nodejs/migration/
- `scanIterator()` (also `hscanIterator`, `sscanIterator`, `zscanIterator`) yields one array per SCAN page. Pages can be empty or repeat keys.

```js
for await (const keys of client.scanIterator()) { const values = await client.mGet(keys); }
```
  https://redis.io/docs/latest/develop/clients/nodejs/migration/

## Cluster and Sentinel

- Cluster: `createCluster({ rootNodes: [{ url: 'redis://127.0.0.1:16379' }] })`. Azure Managed Redis has clustering on by default, so it needs `createCluster()`.
  https://redis.io/docs/latest/develop/clients/nodejs/connect/ and https://redis.io/docs/latest/develop/clients/nodejs/amr/
- `createSentinel()` takes no URL. `sentinelClientOptions` and `nodeClientOptions` carry separate credentials. `replicaPoolSize` enables replica reads, which can be stale.
  https://redis.io/docs/latest/develop/clients/nodejs/connect/
- ioredis Sentinel ignores failover announcements by default and keeps using the old primary until Sentinel reconfigures it as a replica, and writes it accepts meanwhile are lost: set `failoverDetector: true`. A wrong or missing `sentinelPassword` shows as "all Sentinels are unreachable".
  https://redis.io/docs/latest/develop/clients/ioredis/connect/
- ioredis: a `rediss://` URL in cluster startup nodes does not enable TLS. Set `tls` in `redisOptions`.
  https://redis.io/docs/latest/develop/clients/connection-urls/
- With Entra ID tokens, always use `multi()`/`exec()`: a token refresh can slip an AUTH between hand-written MULTI commands.
  https://redis.io/docs/latest/develop/clients/nodejs/amr/

## Client-side caching

- v5.1.0+, needs `RESP: 3`: `createClient({ RESP: 3, clientSideCache: { ttl: 0, maxEntries: 0, evictPolicy: 'LRU' } })`. Opt-in, opt-out and broadcast modes are not implemented. A pool shares one cache.
  https://redis.io/docs/latest/develop/clients/nodejs/connect/
- Smart client handoffs (v5.9.0+, v5.11.0+ for OSS Cluster API connections; Redis Cloud and Software only) also need `RESP: 3` set explicitly.
  https://redis.io/docs/latest/develop/clients/nodejs/connect/

## Search, vectors and probabilistic types

- Search methods send query dialect 2 by default since v5.0.0, overriding the server default.
  https://redis.io/docs/latest/develop/clients/nodejs/queryjson/
- `ft.dropIndex` throws if the index is missing. `{ DD: true }` also deletes the documents.
  https://redis.io/docs/latest/develop/clients/nodejs/queryjson/
- Hash vectors are a `Buffer` of Float32 data. JSON vectors are plain arrays, but the query parameter is still a `Buffer`.
  https://redis.io/docs/latest/develop/clients/nodejs/vecsearch/
- Probabilistic commands live under `client.bf`, `client.cf`, `client.cms`, `client.tDigest`, `client.topK`.
  https://redis.io/docs/latest/develop/clients/nodejs/prob/
- `hImportPrepare`/`hImportSet` (v6.2.0+, Redis Open Source 8.10) bulk-load compact hashes. Experimental, and `hImportSet` replaces an existing key.
  https://redis.io/docs/latest/develop/clients/nodejs/queryjson/
- OpenTelemetry: `OpenTelemetry.init({ metrics: { enabled: true } })` once per app. Only `connection-basic` and `resiliency` groups are on by default.
  https://redis.io/docs/latest/develop/clients/nodejs/observability/

## Migrating from ioredis

| ioredis | node-redis | Source |
| --- | --- | --- |
| connects in constructor | `await client.connect()` | https://redis.io/docs/latest/develop/clients/nodejs/migration/ |
| `set('key', 100, 'EX', 10)` | `set('bike:5', 'bike', { EX: 10 })` | https://redis.io/docs/latest/develop/clients/nodejs/migration/ |
| lowercase only (`hset`) | `HSET` or `hSet` | https://redis.io/docs/latest/develop/clients/nodejs/migration/ |
| `pipeline()` | `multi()....execAsPipeline()` or same-tick `Promise.all` | https://redis.io/docs/latest/develop/clients/nodejs/migration/ |
| `scanStream()` | `scanIterator()` | https://redis.io/docs/latest/develop/clients/nodejs/migration/ |
| `on('message')` | `subscribe(channel, cb)` on `duplicate()` | https://redis.io/docs/latest/develop/clients/nodejs/migration/ |
| `call()`, `config('SET', ...)` | `sendCommand([...])`, `configSet()` | https://redis.io/docs/latest/develop/clients/nodejs/migration/ |
| callbacks | Promises, callbacks only via `client.legacy()` | https://redis.io/docs/latest/develop/clients/nodejs/migration/ |
| `close` event | `end` event | https://redis.io/docs/latest/develop/clients/nodejs/connect/ |
| `setnx`, `hmset` | `set(k, v, { NX: true })`, `hSet` with several pairs | https://redis.io/docs/latest/develop/clients/nodejs/migration/ |
