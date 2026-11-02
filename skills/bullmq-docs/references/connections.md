# Connections, Redis clients and hosting

## Contents

- Defaults and connection count
- Producer versus worker settings
- Client adapters
- Reaching the raw client
- Redis Cluster and hash tags
- Hosted Redis and compatible datastores

## Defaults and connection count

- With no `connection` option, BullMQ creates an ioredis client to `localhost:6379`.
  https://docs.bullmq.io/guide/connections
- Supported datastores: Redis 6.2.0 or newer (also Valkey and Dragonfly) and PostgreSQL 13 or newer (14+ recommended). Only vendors BullMQ tests regularly are officially supported.
  https://docs.bullmq.io/, https://docs.bullmq.io/guide/postgresql and https://docs.bullmq.io/guide/redis-tm-compatibility/
- Every class takes at least one connection. `Worker` and `QueueEvents` duplicate their connection internally for blocking commands, so a client you pass must support `duplicate()`. Reusing an ioredis instance in a `Worker` still creates a duplicate.
  https://docs.bullmq.io/guide/connections
- Redis connections are cheap. Default to separate connections and reuse one only when the provider imposes a hard limit.
  https://docs.bullmq.io/guide/connections
- Never use the ioredis `keyPrefix` option. It is incompatible with BullMQ. Use the BullMQ `prefix` queue option.
  https://docs.bullmq.io/guide/connections

## Producer versus worker settings

Use two different connection profiles: a `Queue` should fail quickly when Redis is down, a `Worker` should wait indefinitely without raising.
https://docs.bullmq.io/guide/going-to-production

- `maxRetriesPerRequest` (ioredis) is the number of retries of a failed command before it throws. A producer `Queue` keeps the default (20) or a low value such as 1.
  https://docs.bullmq.io/guide/connections
- An ioredis instance you build yourself and pass to a `Worker` must have `maxRetriesPerRequest: null`, or BullMQ throws. When BullMQ builds the worker connection from options it sets `null` itself, and overriding it prints a warning.
  https://docs.bullmq.io/guide/connections and https://docs.bullmq.io/guide/going-to-production

```typescript
const connection = new IORedis({ maxRetriesPerRequest: null });
const worker = new Worker('myFirstWorker', async job => {}, { connection });
```
https://docs.bullmq.io/guide/connections

- By default `queue.add` while Redis is down does not fail: it waits for reconnection (ioredis offline queue). For a `Queue` called from an HTTP handler, pass `enableOfflineQueue: false` so `add` throws, then answer 503. Keep the offline queue on workers.
  https://docs.bullmq.io/patterns/failing-fast-when-redis-is-down

```typescript
const queue = new Queue('transcoding', { connection: { enableOfflineQueue: false } });
app.post('/jobs', async (req, res) => {
  try {
    const job = await queue.add('myjob', req.body);
    res.status(201).json(job.id);
  } catch (err) {
    res.status(503).send(err);
  }
});
```
https://docs.bullmq.io/patterns/failing-fast-when-redis-is-down

- Redis must be reachable while the `Queue` is being instantiated, fail-fast mode included.
  https://docs.bullmq.io/patterns/failing-fast-when-redis-is-down
- ioredis connections BullMQ builds reconnect forever with `Math.max(Math.min(Math.exp(times), 20000), 1000)` ms (1 s to 20 s). Override it through the ioredis `retryStrategy` option.
  https://docs.bullmq.io/guide/going-to-production

## Client adapters

- A raw `IORedis` instance as `connection` still works for backward compatibility. It is wrapped in a transparent proxy implementing `IRedisClient`, and its `duplicate()` returns another wrapped proxy, not the raw client.
  https://docs.bullmq.io/guide/connections
- The preferred path is an adapter: `createIORedisClient`, `createNodeRedisClient`, `createBunRedisClient`, `createValkeyGlideClient`. BullMQ does not create node-redis or Bun clients directly: create the raw client in your application and wrap it.
  https://docs.bullmq.io/guide/connections
- node-redis needs `redis` 5 or newer.
  https://docs.bullmq.io/guide/connections

```typescript
import { Queue, createNodeRedisClient } from 'bullmq';
import { createClient } from 'redis';
const connection = createNodeRedisClient(createClient({ url: 'redis://localhost:6379' }));
const queue = new Queue('myqueue', { connection });
```
https://docs.bullmq.io/guide/connections

- To make every BullMQ class build a non-ioredis client from plain options, set a global factory:

```typescript
RedisConnection.clientFactory = opts => {
  const raw = createClient({
    socket: { host: opts.host, port: opts.port },
    username: opts.username, password: opts.password, database: opts.db,
  });
  return createNodeRedisClient(raw);
};
```
https://docs.bullmq.io/guide/connections

- With a non-ioredis adapter, configure reconnect and timeouts in that client yourself (for example node-redis `socket.reconnectStrategy`), so workers keep retrying and producers fail fast. Test Redis down at startup, a transient disconnect while processing, and graceful shutdown with active jobs before production.
  https://docs.bullmq.io/guide/going-to-production
- Close a shared connection only after every `Queue` and `Worker` using it is closed. With Bun, close through the wrapper (`connection.disconnect()` or `await connection.quit()`), never `close()` on the raw Bun `RedisClient`: in-flight commands reject with `ConnectionClosedError` and the wrapper tries to reconnect.
  https://docs.bullmq.io/guide/connections
- A custom backend factory needs an options object with a `connection`. Implementing `IQueueBackend` to production grade (atomicity, locking, timing, events) is substantial work.
  https://docs.bullmq.io/guide/connections

## Reaching the raw client

High-level classes have no `client` getter. Use the backend (Redis only). `getBackend().blockingClient` gives a worker's blocking connection. These escape hatches are outside the datastore-agnostic contract, so prefer the `Queue`, `Worker` and `FlowProducer` API.

```typescript
const client: RedisClient = await queue.getBackend().client;
```
https://docs.bullmq.io/guide/connections

## Redis Cluster and hash tags

All keys of a queue must land in one slot. Put a hash tag in the prefix or in the queue name. Give different queues different prefixes so they spread over the nodes.

```typescript
const queue = new Queue('cluster', { prefix: '{myprefix}' });
// or
const queue2 = new Queue('{cluster}');
```
https://docs.bullmq.io/patterns/redis-cluster

## Hosted Redis and compatible datastores

- AWS ElastiCache: use standard cache nodes, not serverless (its eviction policy is incompatible). Default parameter groups cannot be edited, so create a custom one with `maxmemory-policy` = `noeviction` and attach it. Open TCP 6379 to the BullMQ services.
  https://docs.bullmq.io/guide/redis-tm-hosting/aws-elasticache
- AWS MemoryDB: cluster mode only (use hash tags), reachable only inside the VPC. Close in order: `await worker.close()`, then `await connection.quit()`.

```typescript
const connection = new Cluster([{ host: 'clustercfg.xxx.amazonaws.com', port: 6379 }], { tls: {} });
const worker = new Worker('myqueue', async (job: Job) => {}, { connection });
await worker.close();
await connection.quit();
```
https://docs.bullmq.io/guide/redis-tm-hosting/aws-memorydb

- Dragonfly: name queues with braces (`{myqueue}`) so each queue gets a thread. Splitting one queue into `{myqueue-1}`, `{myqueue-2}` uses more cores, but priorities and rate limiting may not hold across the split. Dragonfly's own guide lists the required flags.
  https://docs.bullmq.io/guide/redis-tm-compatibility/dragonfly
