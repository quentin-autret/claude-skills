# Adapters (Socket.IO 4.8)

## Contents

- Choosing an adapter
- Feature matrix
- Redis adapter
- Sharded Redis adapter
- Redis Streams adapter
- Postgres adapter
- MongoDB adapter
- Cluster adapter
- Cloud adapters
- Emitters
- Security of the broker

## Choosing an adapter

- The adapter is the server-side component that broadcasts events to all or a subset of clients. Replace the in-memory default as soon as there are 2 servers. https://socket.io/docs/v4/adapter/
- Official adapters: Redis, Redis Streams, MongoDB, Postgres, Cluster, Google Cloud Pub/Sub, AWS SQS, Azure Service Bus. https://socket.io/docs/v4/adapter/
- Each namespace has its own adapter instance, read with `io.of("/").adapter`. `io.adapter()` does not work for reading it. https://socket.io/docs/v4/adapter/#api
- Sticky sessions stay required with every adapter while long-polling is enabled. https://socket.io/docs/v4/redis-adapter/#do-i-still-need-to-enable-sticky-sessions-when-using-the-redis-adapter

## Feature matrix

| Adapter | `fetchSockets` etc. | `serverSideEmit` | Broadcast acks | Recovery |
| ------- | ------------------- | ---------------- | -------------- | -------- |
| Redis | 6.1.0+ | 7.0.0+ | 7.2.0+ | No |
| Redis Streams | Yes | Yes | Yes | Yes |
| MongoDB | Yes | Yes | 0.2.0+ | 0.3.0+ |
| Postgres | Yes | Yes | 0.3.0+ | No |
| Cluster | Yes | Yes | 0.2.0+ | No |
| AWS SQS, Azure, GCP | Yes | Yes | Yes | No |

Sources: https://socket.io/docs/v4/redis-adapter/#supported-features, https://socket.io/docs/v4/redis-streams-adapter/#supported-features, https://socket.io/docs/v4/mongo-adapter/#supported-features, https://socket.io/docs/v4/postgres-adapter/#supported-features

## Redis adapter

- `@socket.io/redis-adapter` uses Redis Pub/Sub and stores no keys. If Redis is down, packets reach only the clients connected to the current server. https://socket.io/docs/v4/redis-adapter/#how-it-works
- Provide 2 clients: the subscriber is a `duplicate()` of the publisher. With the `redis` package, connect both before use. https://socket.io/docs/v4/redis-adapter/#with-the-redis-package

```js
import { createClient } from "redis";
import { Server } from "socket.io";
import { createAdapter } from "@socket.io/redis-adapter";

const pubClient = createClient({ url: "redis://localhost:6379" });
const subClient = pubClient.duplicate();
await Promise.all([pubClient.connect(), subClient.connect()]);

const io = new Server({ adapter: createAdapter(pubClient, subClient) });
io.listen(3000);
```

https://socket.io/docs/v4/redis-adapter/#with-the-redis-package

- The `redis` package has known issues restoring subscriptions after a reconnection. The docs suggest `ioredis`. https://socket.io/docs/v4/redis-adapter/#with-the-redis-package
- Options: `key` (channel prefix, `socket.io`), `requestsTimeout` (`5_000` ms for `fetchSockets` and `serverSideEmit`), `publishOnSpecificResponseChannel` (`false`), `parser`. https://socket.io/docs/v4/redis-adapter/#default-adapter
- Set `publishOnSpecificResponseChannel: true`: responses then go only to the requesting server. The `false` default exists for backward compatibility. https://socket.io/docs/v4/redis-adapter/#default-adapter
- Adapter 7.x and later need Socket.IO 4.3.1 or later. https://socket.io/docs/v4/redis-adapter/#compatibility-table

## Sharded Redis adapter

- For new projects, default to `createShardedAdapter`, based on sharded Pub/Sub. Minimums: Redis 7.0, `redis` 4.6.0 or `ioredis` 5.9.0. With an `ioredis` Cluster, set `shardedSubscribers: true`. https://socket.io/docs/v4/redis-adapter/#with-redis-sharded-pubsub

```js
import { createShardedAdapter } from "@socket.io/redis-adapter";
const io = new Server({ adapter: createShardedAdapter(pubClient, subClient) });
```

https://socket.io/docs/v4/redis-adapter/#with-redis-sharded-pubsub

- Options: `channelPrefix` (`socket.io`) and `subscriptionMode`. `dynamic` (default) adds 1 channel per public room, good for rooms with few clients. `static` uses 2 channels per namespace, good with dynamic namespaces. `dynamic-private` adds private rooms too, good with many 1:1 emits. https://socket.io/docs/v4/redis-adapter/#sharded-adapter

## Redis Streams adapter

- `@socket.io/redis-streams-adapter` reads a Redis stream, so it resumes after a short Redis outage without losing packets. It works with Valkey and supports connection state recovery. https://socket.io/docs/v4/redis-streams-adapter/#how-it-works
- It takes 1 client. https://socket.io/docs/v4/redis-streams-adapter/#with-the-redis-package

```js
import { createClient } from "redis";
import { createAdapter } from "@socket.io/redis-streams-adapter";

const redisClient = createClient({ url: "redis://localhost:6379" });
await redisClient.connect();
const io = new Server({
  adapter: createAdapter(redisClient),
  connectionStateRecovery: { maxDisconnectionDuration: 2 * 60 * 1000 },
});
```

https://socket.io/docs/v4/redis-streams-adapter/#with-the-redis-package

- Options: `streamName` (`socket.io`), `streamCount` (`1`), `channelPrefix`, `useShardedPubSub` (`false`), `maxLen` (`10_000`), `readCount` (`100`), `blockTimeInMs` (`5_000`), `sessionKeyPrefix` (`sio:session:`), `onlyPlaintext` (`false`, set `true` when payloads never hold binary). https://socket.io/docs/v4/redis-streams-adapter/#options

## Postgres adapter

- `@socket.io/postgres-adapter` uses `LISTEN`/`NOTIFY`. Binary packets and packets over 8000 bytes go to an attachment table you create, and NOTIFY carries the row ID. https://socket.io/docs/v4/postgres-adapter/#how-it-works

```js
await pool.query(`
  CREATE TABLE IF NOT EXISTS socket_io_attachments (
    id bigserial UNIQUE,
    created_at timestamptz DEFAULT NOW(),
    payload bytea
  );
`);
pool.on("error", (err) => console.error("Postgres error", err));
io.adapter(createAdapter(pool));
```

https://socket.io/docs/v4/postgres-adapter/#standalone

- Options: `channelPrefix`, `tableName` (`socket_io_attachments`), `payloadThreshold` (`8_000`), `cleanupInterval` (`30_000`), `heartbeatInterval` (`5_000`), `heartbeatTimeout` (`10_000`). Needs `pg`, and `@types/pg` in TypeScript. https://socket.io/docs/v4/postgres-adapter/#options

## MongoDB adapter

- `@socket.io/mongo-adapter` relies on Change Streams, so it needs a replica set or a sharded cluster. https://socket.io/docs/v4/mongo-adapter/#how-it-works
- Clean the collection with a capped collection or a TTL index. The TTL variant needs `addCreatedAtField: true`. https://socket.io/docs/v4/mongo-adapter/#usage-with-a-ttl-index

```js
await mongoCollection.createIndex({ createdAt: 1 }, { expireAfterSeconds: 3600, background: true });
io.adapter(createAdapter(mongoCollection, { addCreatedAtField: true }));
```

https://socket.io/docs/v4/mongo-adapter/#usage-with-a-ttl-index

- When the cluster is unreachable, the driver buffers packets by default. With `bufferMaxEntries: 0`, packets reach only local clients. https://socket.io/docs/v4/mongo-adapter/#what-happens-when-the-mongodb-cluster-is-down

## Cluster adapter

- `@socket.io/cluster-adapter` forwards packets between workers of a Node.js cluster over the IPC channel. `setupPrimary()` runs in the primary, `createAdapter()` and `setupWorker(io)` in workers (`setupWorker` comes from `@socket.io/sticky`, which also provides `setupMaster` for sticky sessions). https://socket.io/docs/v4/cluster-adapter/#with-nodejs-cluster

## Cloud adapters

- AWS SQS (`@socket.io/aws-sqs-adapter`): an SNS topic plus an SQS queue. Wait for the queue, and close on shutdown to delete it. https://socket.io/docs/v4/aws-sqs-adapter/#usage

```js
const io = new Server({ adapter: createAdapter(new SNS(), new SQS()) });
await io.of("/").adapter.init();
process.once("SIGTERM", async () => { await io.close(); });
```

https://socket.io/docs/v4/aws-sqs-adapter/#usage

- Azure Service Bus (`@socket.io/azure-service-bus-adapter`): needs `ServiceBusClient` and `ServiceBusAdministrationClient`, creates 1 subscription per server. Await `io.of("/").adapter.init()`. https://socket.io/docs/v4/azure-service-bus-adapter/#usage
- Google Cloud Pub/Sub (`@socket.io/gcp-pubsub-adapter`): pass a `topic`. A new subscription is created (`subscriptionPrefix`, default `socket.io`). Await `io.of("/").adapter.init()`. https://socket.io/docs/v4/gcp-pubsub-adapter/#usage

## Emitters

- An emitter sends to clients from a process that runs no Socket.IO server. Its API mirrors broadcasting. https://socket.io/docs/v4/adapter/#emitter-cheatsheet

```js
const emitter = new Emitter(redisClient); // @socket.io/redis-emitter
emitter.to("room1").except("room2").emit("hello");
emitter.of("/admin").in("room1").socketsJoin("room2");
emitter.in("room1").disconnectSockets(true);
emitter.serverSideEmit("hello", "world");
```

https://socket.io/docs/v4/adapter/#emitter-cheatsheet

## Security of the broker

- The Redis adapters trust Redis: messages are not signed, encrypted or authenticated. Anyone who can publish can inject packets, anyone who can subscribe reads payloads. https://socket.io/docs/v4/redis-adapter/#security-considerations
- Keep Redis on a private network with ACLs, auth, TLS and dedicated credentials. Do not share it with unrelated apps without channel isolation, and keep long-lived secrets out of payloads. https://socket.io/docs/v4/redis-adapter/#security-considerations
