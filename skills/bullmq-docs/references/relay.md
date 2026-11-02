# Relay: cross-process messages

## Contents

- When to use it
- Instance and lifecycle
- Topics and patterns
- Messages and delivery
- Retained messages
- Nodes, leases and recovery
- Backends
- Defaults and errors

The Relay is experimental: its API may change in minor releases.
https://docs.bullmq.io/guide/relay/

## When to use it

- A queue is for work: exactly one worker, retries, persistence. The Relay tells other processes that something happened or what the current state is (cancel, progress, cache invalidation, config, notifications).
  https://docs.bullmq.io/guide/relay/ and https://docs.bullmq.io/guide/relay/relay-vs-kafka
- No replay, no history, no log. A subscription sees only messages published after it, except the retained message of an exact topic.
  https://docs.bullmq.io/guide/relay/relay-vs-kafka
- Over Redis pub/sub it adds an inbox that holds messages during brief disconnections, and it works with every Redis client BullMQ supports (ioredis, node-redis, Bun's Redis client, Valkey GLIDE).
  https://docs.bullmq.io/guide/relay/ and https://docs.bullmq.io/guide/relay/backends

## Instance and lifecycle

- One `Relay` per process, shared by all code. Each node uses 2 connections.
  https://docs.bullmq.io/guide/relay/nodes-and-failures
- Listen to `'error'`: listener exceptions and background failures are emitted there.
- `waitUntilReady()` is optional, `subscribe` and `publish` wait for it.
- Close on shutdown after the workers, so other nodes do not wait for the lease. `close()` drops messages still in the inbox. A closed relay throws `Error('Relay is closed')`.

```typescript
import { Relay } from 'bullmq';
const relay = new Relay({ connection: { host: 'localhost', port: 6379 } });
relay.on('error', err => logger.error(err, 'relay error'));

const subscription = await relay.subscribe('orders.*.status', message => {
  console.log(message.topic, message.data);
});
const result = await relay.publish('orders.42.status', { status: 'shipped' });
// { mid: '1873', nodes: 3, endpoints: 5 }
await subscription.unsubscribe();

process.on('SIGTERM', async () => {
  await worker.close();
  await relay.close();
});
```
https://docs.bullmq.io/guide/relay/getting-started and https://docs.bullmq.io/guide/relay/reference

## Topics and patterns

- Dot-separated, most general segment first, each identifier in its own segment. Segment characters: `A-Z a-z 0-9 _ - : %`. Max 512 bytes, max 16 segments, no empty segment.
  https://docs.bullmq.io/guide/relay/topics-and-patterns
- `*` matches exactly one segment, `>` one or more trailing segments (last position only). Hierarchy exists only in names: `orders.42.status` is not delivered to `orders.42`, and `orders.>` does not match `orders`.
  https://docs.bullmq.io/guide/relay/topics-and-patterns
- Publish to a concrete topic. Wildcards are for subscriptions only.
  https://docs.bullmq.io/guide/relay/topics-and-patterns
- One topic per resource (`jobs.42.progress`) instead of the id in the payload. Topics cost nothing until subscribed.
  https://docs.bullmq.io/guide/relay/topics-and-patterns
- Subscribe narrowly: every matching message lands in the node's inbox even if the handler ignores it. Exact lookups are direct, wildcard patterns are tested one by one, so keep them few.
  https://docs.bullmq.io/guide/relay/topics-and-patterns
- Percent-encode data-derived segments. An unencoded `.` adds levels, and a subscription built from user input may see more than intended. Validate and authorize ids before building a topic.
  https://docs.bullmq.io/guide/relay/topics-and-patterns and https://docs.bullmq.io/guide/relay/examples/live-job-progress

```typescript
function topicSegment(value: string | number): string {
  return encodeURIComponent(String(value)).replace(
    /[.!'()*~]/g,
    char => `%${char.charCodeAt(0).toString(16).toUpperCase()}`,
  );
}
```
https://docs.bullmq.io/guide/relay/topics-and-patterns

- Isolate environments or applications with `namespace`, not topic prefixes.
  https://docs.bullmq.io/guide/relay/topics-and-patterns

## Messages and delivery

- Small messages: what changed plus an id. Above `maxMessageSize` (512 KiB) publishing throws `RelayError` `DataTooLarge`.
  https://docs.bullmq.io/guide/relay/messages and https://docs.bullmq.io/guide/relay/reference
- Payloads are JSON: `undefined` becomes `null`, `Date` an ISO string, class instances plain objects.
  https://docs.bullmq.io/guide/relay/messages
- Listeners are not awaited. Slow async listeners pile up. Hand real work to a queue from the listener.

```typescript
await relay.subscribe('reports.*.requested', async message => {
  await reports.add('generate', message.data);
});
```
https://docs.bullmq.io/guide/relay/messages

- Each matching subscription gets its own copy, the publisher included. Only nodes subscribed at publish time receive it.
  https://docs.bullmq.io/guide/relay/messages
- No ordering between concurrent publishes, even from one process. Await each `publish` when order matters, and compare `mid` (increasing within a namespace) to drop stale updates.
  https://docs.bullmq.io/guide/relay/messages
- `inboxMaxLength` (10000) drops the oldest unread messages. The `'lag'` event (threshold `inboxMaxLength / 2`) reports the backlog after trimming, so it does not count losses.
  https://docs.bullmq.io/guide/relay/messages

## Retained messages

- `{ retain: ms }` keeps the last value of a topic. A subscriber to the exact topic gets it first, before `subscribe()` resolves. Wildcard subscriptions never get retained values.
  https://docs.bullmq.io/guide/relay/retained-messages
- Publishing without `retain` leaves the retained value unchanged. `retain` omitted or `0` means none. Remove with `deleteRetained(topic)`.
  https://docs.bullmq.io/guide/relay/retained-messages and https://docs.bullmq.io/guide/relay/reference
- Keep the database as the source of truth, the retained message is a pushed cache.
  https://docs.bullmq.io/guide/relay/retained-messages

```typescript
await relay.publish('jobs.42.progress', 70, { retain: 10 * 60 * 1000 });
await relay.subscribe('jobs.42.progress', message => {
  console.log(message.data, message.retained);
});
```
https://docs.bullmq.io/guide/relay/retained-messages

## Nodes, leases and recovery

- `leaseDuration` 30000 ms, heartbeat every third of it, sweep every lease.
  https://docs.bullmq.io/guide/relay/nodes-and-failures
- A node silent longer than the lease is swept and loses messages published meanwhile. On return it re-registers, restores subscriptions and emits `'recovered'` before new messages. Retained messages are not redelivered: refresh state on `'recovered'`.
  https://docs.bullmq.io/guide/relay/nodes-and-failures and https://docs.bullmq.io/guide/relay/retained-messages

## Backends

- Redis is the default and handles heavy traffic better. It needs Redis 6.2+. Keys live under `{prefix}::relay:{<namespace>}:` and expire by themselves. A namespace is one hash slot, so spread unrelated traffic over several namespaces in a cluster. A hash tag in the prefix overrides that.
  https://docs.bullmq.io/guide/relay/backends
- An existing Redis client passed as `connection` is duplicated for blocking reads and left open on close.
  https://docs.bullmq.io/guide/relay/backends
- PostgreSQL needs migration `0004_relay` (or `migrate: true`). A passed `pg.Pool` needs `max >= 2` because the listener holds one slot. Every listening node receives every `NOTIFY` of the namespace, so large busy fleets belong on Redis.
  https://docs.bullmq.io/guide/relay/backends and https://docs.bullmq.io/guide/relay/getting-started

```typescript
import { Relay, createPostgresRelayBackend } from 'bullmq';
const relay = new Relay({
  connection: { connectionString: process.env.DATABASE_URL, migrate: true },
  backend: createPostgresRelayBackend,
});
```
https://docs.bullmq.io/guide/relay/getting-started

## Defaults and errors

`namespace` `'default'`, `prefix` `'bull'` (Redis), `blockTimeout` 5000 ms. Codes: `InvalidTopic` (-1), `InvalidPattern` (-2), `NodeNotRegistered` (-3, recovers on next heartbeat), `DataTooLarge` (-4), `IncompatibleLayout` (-6, nodes of an incompatible version share a namespace). Inspect with `getCounts()`, `getNodes(start, end)`, `getSubscriptions({ pattern })`.
https://docs.bullmq.io/guide/relay/reference
