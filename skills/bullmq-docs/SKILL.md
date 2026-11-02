---
name: bullmq-docs
description: Pitfalls, surprising defaults and recommended patterns of BullMQ for Node.js and TypeScript, distilled from the official docs. Use when writing, reviewing or debugging code that adds or processes jobs with BullMQ (Queue, Worker, Job, a Redis queue or the PostgreSQL backend), including connections and ioredis settings, maxRetriesPerRequest, retries and backoff, UnrecoverableError, stalled jobs and locks, concurrency, rate limiting, sandboxed processors, graceful shutdown, removeOnComplete retention, job ids and deduplication, delayed and prioritized jobs, job schedulers (upsertJobScheduler) replacing repeatable jobs, flows with FlowProducer, QueueEvents and custom events, metrics and Prometheus, OpenTelemetry with bullmq-otel, NestJS with @nestjs/bullmq, the experimental Relay for cross-process messages, Redis Cluster and hosted Redis, and upgrades from Bull or from BullMQ v5 to v6.
metadata:
  version: "6.x (docs of master, 6.5.3)"
  source: https://docs.bullmq.io/
  fetched: "2026-11-02"
---

# BullMQ from the official docs

## Overview

This skill holds what the BullMQ docs say that is easy to get wrong: connection settings that differ between producers and workers, defaults that keep every finished job forever or never retry, lock and stall behavior, scheduler timing, flow failure rules and the APIs removed in v6. Generic queueing advice is left out. Every claim carries the docs URL it comes from, so it can be checked.

## Version covered

BullMQ 6.x for Node.js and TypeScript, from the docs of the master branch (6.5.3). Python, Elixir, PHP, .NET and Rust SDKs, Bull (the predecessor) and BullMQ Pro features are excluded. Code written for v5 or earlier must go through "Old patterns" below and `references/migrations.md`.

## How to navigate

1. Read the quick start and the rules below. They cover most code.
2. Open the one reference file whose domain matches the task, from the table at the end.
3. Before reviewing old code or upgrading, read "Old patterns".

## Quick start

`ioredis` is an optional peer dependency in v6: install it next to `bullmq` when using Redis.
https://docs.bullmq.io/changelog

```typescript
import { Queue, Worker, UnrecoverableError } from 'bullmq';

// Producer: fail fast when Redis is down.
const queue = new Queue('emails', {
  connection: { host: 'localhost', port: 6379, enableOfflineQueue: false },
  defaultJobOptions: {
    attempts: 5,
    backoff: { type: 'exponential', delay: 1000 },
    removeOnComplete: 1000,
    removeOnFail: 5000,
  },
});
queue.on('error', err => console.error(err));

await queue.add('welcome', { userId: 'u-42' }); // plain JSON only

// Consumer: a Worker sets maxRetriesPerRequest: null by default on ioredis connections.
const worker = new Worker(
  'emails',
  async (job, token, signal) => {
    if (!job.data.userId) throw new UnrecoverableError('missing userId');
    return sendEmail(job.data, { signal });
  },
  { connection: { host: 'localhost', port: 6379 }, concurrency: 50 },
);
worker.on('error', err => console.error(err));

const shutdown = async () => {
  await worker.close();
  await queue.close();
  process.exit(0);
};
process.on('SIGINT', shutdown);
process.on('SIGTERM', shutdown);
```
https://docs.bullmq.io/guide/going-to-production (section maxRetriesPerRequest) , https://docs.bullmq.io/patterns/failing-fast-when-redis-is-down , https://docs.bullmq.io/guide/retrying-failing-jobs , https://docs.bullmq.io/guide/queues/auto-removal-of-jobs

When you create the ioredis instance yourself for a worker, it must have `maxRetriesPerRequest: null`. The connections page says BullMQ throws otherwise, the going-to-production page says it outputs a warning.

```typescript
const connection = new IORedis({ maxRetriesPerRequest: null });
const worker = new Worker('emails', processor, { connection });
```
https://docs.bullmq.io/guide/connections

## Most important rules

### Redis and connections

- Set Redis `maxmemory-policy` to `noeviction`. Any eviction breaks queues and shows up as "Missing lock for job" errors.
  https://docs.bullmq.io/guide/going-to-production and https://docs.bullmq.io/guide/troubleshooting
- Enable persistence yourself (AOF is recommended, 1 second per write is usually enough), many hosts ship without it.
  https://docs.bullmq.io/guide/going-to-production
- Never use the ioredis `keyPrefix` option. Use the BullMQ `prefix` option.
  https://docs.bullmq.io/guide/connections
- By default `queue.add` hangs while Redis is down. `enableOfflineQueue: false` on the producer makes it throw, keep the offline queue on workers.
  https://docs.bullmq.io/patterns/failing-fast-when-redis-is-down and https://docs.bullmq.io/guide/going-to-production
- `Worker` and `QueueEvents` duplicate their connection for blocking commands. A passed client must support `duplicate()`.
  https://docs.bullmq.io/guide/connections
- On Redis Cluster, put a hash tag in the prefix or queue name: `prefix: '{myprefix}'`.
  https://docs.bullmq.io/patterns/redis-cluster

### Defaults that surprise

- No retries: `attempts` must be greater than 1. Without `backoff`, retries are immediate.
  https://docs.bullmq.io/guide/retrying-failing-jobs
- Finished jobs are kept forever. Set `removeOnComplete` and `removeOnFail`. Removal runs lazily when another job finishes.
  https://docs.bullmq.io/guide/queues/auto-removal-of-jobs
- `concurrency` is 1 per worker. Raise it for IO-bound jobs (100 to 300 is standard), add workers for CPU-bound jobs.
  https://docs.bullmq.io/guide/parallelism-and-concurrency
- A job without priority runs before every prioritized job. Lower number means higher priority.
  https://docs.bullmq.io/guide/jobs/prioritized
- `delay` is a minimum, not an exact time.
  https://docs.bullmq.io/guide/jobs/delayed
- BullMQ has no job timeout. Build one with `AbortController` in the processor.
  https://docs.bullmq.io/patterns/timeout-jobs
- `limiter: { max, duration }` on a worker is global to the queue, not per worker.
  https://docs.bullmq.io/guide/rate-limiting

### Processing

- Delivery is at-least-once: make processors idempotent.
  https://docs.bullmq.io/
- Attach an `error` listener to every worker (and queue), or Node.js raises an unhandled exception when an error is emitted.
  https://docs.bullmq.io/guide/workers
- Throw `Error` objects. `UnrecoverableError` fails without retry.
  https://docs.bullmq.io/guide/retrying-failing-jobs and https://docs.bullmq.io/patterns/stop-retrying-jobs
- Locks last 30 s and are renewed by the event loop. CPU work that blocks it longer stalls the job: it goes back to waiting, and it fails permanently once it stalls more than `maxStalledCount` (default 1). Move CPU work to a sandboxed processor.
  https://docs.bullmq.io/guide/jobs/stalled , https://docs.bullmq.io/guide/troubleshooting and https://docs.bullmq.io/guide/workers/sandboxed-processors
- Job data is JSON: class instances arrive as plain objects, `Map`, `Set`, `BigInt` and `undefined` are lost. Data is stored in clear text.
  https://docs.bullmq.io/guide/jobs/job-data and https://docs.bullmq.io/guide/going-to-production
- Persist results inside the processor, not in a `completed` handler.
  https://docs.bullmq.io/guide/returning-job-data
- `await worker.close()` on `SIGTERM`. In TypeScript it waits for active jobs with no timeout.
  https://docs.bullmq.io/guide/workers/graceful-shutdown

### Ids, deduplication, scheduling, flows, events

- A custom `jobId` must not contain `:` and must not be digits only: use `"job-123"`.
  https://docs.bullmq.io/guide/jobs/job-ids
- On Redis, do not assume a failed `queue.addBulk` added no jobs. `flowProducer.addBulk` is all or none, across queues too.
  https://docs.bullmq.io/guide/queues/adding-bulks and https://docs.bullmq.io/patterns/adding-bulks
- Recurring work uses `queue.upsertJobScheduler(id, { every | pattern, tz }, { name, data, opts })`. The next job is produced only when the previous one starts.
  https://docs.bullmq.io/guide/job-schedulers and https://docs.bullmq.io/patterns/deduplication
- Cron patterns accept an optional leading seconds field: `0 15 3 * * *` is 03:15 daily.
  https://docs.bullmq.io/guide/job-schedulers/repeat-strategies
- Never put `deduplication` on flow nodes or expect it in scheduler templates.
  https://docs.bullmq.io/guide/flows and https://docs.bullmq.io/patterns/deduplication
- A flow parent waits in `waiting-children` until every child succeeds. A failed child blocks it unless the child sets a failure policy.
  https://docs.bullmq.io/guide/flows
- Listen with `QueueEvents` to see every worker. Event payloads are small objects such as `{ jobId }` or `{ jobId, data }`, and the stream keeps about 10,000 events by default (`streams.events.maxLen`).
  https://docs.bullmq.io/guide/events and https://docs.bullmq.io/quick-start
- To reach jobs running in another process (cancel, progress, settings), use the Relay, which is experimental.
  https://docs.bullmq.io/guide/workers/cancelling-jobs and https://docs.bullmq.io/guide/relay/

## Old patterns

| Old | Current | Source |
| --- | --- | --- |
| `queue.add(name, data, { repeat })`, class `Repeat` | `queue.upsertJobScheduler(id, repeatOpts, { name, data, opts })` | https://docs.bullmq.io/guide/migrations/migrate-from-v5-to-v6 |
| `getRepeatableJobs()`, `removeRepeatable()`, `removeRepeatableByKey()` | `getJobSchedulers()`, `removeJobScheduler(key)` | https://docs.bullmq.io/guide/migrations/migrate-from-v5-to-v6 |
| `repeat.utc: true` | `tz: 'UTC'` | https://docs.bullmq.io/guide/migrations/migrate-from-v5-to-v6 |
| `immediately: true` | nothing: deprecated since 5.19.0, a new scheduler always runs its first repetition at once | https://docs.bullmq.io/guide/job-schedulers/repeat-options |
| `debounce` option, `debounced` event | `deduplication`, `deduplicated` event | https://docs.bullmq.io/guide/migrations/migrate-from-v5-to-v6 |
| `job.discard()` | `throw new UnrecoverableError(...)` | https://docs.bullmq.io/changelog |
| `queue.client`, `worker.blockingClient`, `queue.redisVersion` | `await queue.getBackend().client` (Redis only) | https://docs.bullmq.io/changelog |
| raw `new IORedis()` as `connection` | still accepted, adapters `createIORedisClient`, `createNodeRedisClient`, `createBunRedisClient`, `createValkeyGlideClient` preferred | https://docs.bullmq.io/guide/connections |
| `queue.resume()` without `await` | `await queue.resume()` (async in v6) | https://docs.bullmq.io/guide/migrations/migrate-from-v5-to-v6 |
| `paused` in job counts | jobs of a paused queue count as waiting | https://docs.bullmq.io/changelog |
| `QueueScheduler` | nothing: deprecated from BullMQ 2.0, do not use it | https://docs.bullmq.io/guide/queuescheduler |
| `limiter.groupKey` | removed in 3.0 | https://docs.bullmq.io/guide/rate-limiting |
| named processors (Bull) | one processor with `switch (job.name)` | https://docs.bullmq.io/patterns/named-processor |
| one queue per step, each enqueuing the next | `FlowProducer` | https://docs.bullmq.io/patterns/flows |
| `new BullMQOtel('app', '1.0.0')` (traces only, still supported) | `new BullMQOtel({ tracerName, meterName, version, enableMetrics })` to get metrics | https://docs.bullmq.io/guide/telemetry/metrics |
| reusing Bull queues | new BullMQ queues, Bull drained in parallel | https://docs.bullmq.io/guide/migrations/bull-to-bullmq |

## Reference files

| File | Read it for |
| --- | --- |
| [references/connections.md](references/connections.md) | default connection, `maxRetriesPerRequest`, fail-fast producers, reconnect strategy, client adapters, raw client, Redis Cluster, ElastiCache, MemoryDB, Dragonfly |
| [references/production.md](references/production.md) | Redis settings, shutdown, error listeners, retention, locks and stalled jobs, troubleshooting messages |
| [references/postgresql.md](references/postgresql.md) | PostgreSQL backend, migrations, typing trap, schema and pool, throughput |
| [references/jobs.md](references/jobs.md) | job data serialization, job ids, delay, priority, LIFO, retention options, `defaultJobOptions` |
| [references/deduplication.md](references/deduplication.md) | Simple, Throttle, Debounce and keep-last-if-active modes, dedup with schedulers, throttling by `jobId` |
| [references/queue-operations.md](references/queue-operations.md) | `addBulk`, batches, global concurrency and rate limit, getters, drain, clean, obliterate, manual retry |
| [references/workers.md](references/workers.md) | `autorun`, concurrency, sandboxed processors, pausing, manual fetching, results, several job kinds |
| [references/retries-and-errors.md](references/retries-and-errors.md) | attempts, backoff, `UnrecoverableError`, rate limiting, timeouts, cancellation, moving jobs, step jobs |
| [references/job-schedulers.md](references/job-schedulers.md) | `upsertJobScheduler`, timing, repeat options, cron, custom strategies, managing schedulers |
| [references/flows.md](references/flows.md) | `FlowProducer`, parent lifecycle, child failure policies, dependencies, removal rules |
| [references/events-and-metrics.md](references/events-and-metrics.md) | `QueueEvents`, custom events, built-in metrics, Prometheus export |
| [references/telemetry.md](references/telemetry.md) | `bullmq-otel`, traces, span names, metric names, SDK ordering |
| [references/nestjs.md](references/nestjs.md) | `@nestjs/bullmq` module, `WorkerHost` consumers, injection, queue events listeners |
| [references/migrations.md](references/migrations.md) | upgrade strategy, Bull to BullMQ, v5 to v6 steps, v6 behavior changes, removed APIs |
| [references/relay.md](references/relay.md) | Relay concepts, topics, delivery, retained messages, leases, backends, errors |
| [references/relay-patterns.md](references/relay-patterns.md) | cancelling jobs on other workers, live progress, notifications, cache invalidation, live config |

Sources and fetch details are listed in [SOURCES.md](SOURCES.md).
