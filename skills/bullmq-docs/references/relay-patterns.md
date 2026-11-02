# Relay patterns from the docs examples

## Contents

- Cancelling jobs on other workers
- Live job progress
- Realtime notifications
- Cache invalidation
- Live configuration and worker control

## Cancelling jobs on other workers

The processor must observe the `AbortSignal` (3rd argument) and throw `UnrecoverableError` so the cancelled job is not retried. Each worker process subscribes and calls `worker.cancelJob`.

```typescript
const worker = new Worker('exports', async (job, token, signal) => {
  if (signal?.aborted) throw new UnrecoverableError(`cancelled: ${signal.reason}`);
  await exportPage(page, { signal });
}, { connection });

await relay.subscribe('jobs.exports.cancel', message => {
  const { jobId, reason } = message.data as { jobId: string; reason: string };
  worker.cancelJob(jobId, reason);
});
```
https://docs.bullmq.io/guide/relay/examples/cancelling-jobs-on-other-workers

Limits: only running jobs are cancelled, cancellation is cooperative, the job may finish first, there is no confirmation, and a worker that left misses the request. `endpoints > 0` means delivered to at least one inbox, not cancelled. Use one subscription per queue when wildcard traffic for other queues matters.
https://docs.bullmq.io/guide/relay/examples/cancelling-jobs-on-other-workers

## Live job progress

Publish only when the percentage changes, or a few times per second at most, with `retain` so late subscribers get the current value. Keep calling `job.updateProgress()` when the progress must also be readable on the job. Validate and authorize the job id (for example `/^\d+$/` plus an access check) before subscribing for a client.
https://docs.bullmq.io/guide/relay/examples/live-job-progress

## Realtime notifications

Subscribe once per user per server while at least one socket is open, unsubscribe when the last one closes. Store notifications in the database too, the Relay is live only. Fall back to a push queue when nobody listened.

```typescript
const { endpoints } = await relay.publish(`users.${userId}.notifications`, notification);
if (endpoints === 0) await pushQueue.add('push', { userId, notification });
```
https://docs.bullmq.io/guide/relay/examples/realtime-notifications

## Cache invalidation

- Publish after the transaction commits, never before.
- Use a generation counter so a read in flight during an invalidation is not cached.
- Clear the cache on `'recovered'`.
- Keep a cache TTL so a missed invalidation cannot stay stale forever.
- For data that changes very often, an expiry on the cache entries is simpler than invalidating every change.

```typescript
relay.on('recovered', () => { generation++; localCache.clear(); });
```
https://docs.bullmq.io/guide/relay/examples/cache-invalidation

## Live configuration and worker control

- One topic per independent setting, retained. Keep the database as the source of truth and the retained message as a pushed copy. Retained messages are not delivered again after a recovery, so reload from the database on `'recovered'`. To keep the retained value from expiring, republish periodically, for example from a job scheduler.
  https://docs.bullmq.io/guide/relay/examples/live-configuration
- Create the worker with `autorun: false`, apply the retained settings, then start. A retained message expires, so keep defaults in the worker's code (or the database) and republish to make a setting last. `worker.pause()` waits for active jobs, `worker.pause(true)` stops fetching new jobs without waiting.

```typescript
const worker = new Worker('emails', processEmail, { connection, concurrency: 10, autorun: false });
await relay.subscribe('workers.emails.settings', message => {
  /* apply concurrency, pause or resume */
});
await relay.publish('workers.emails.settings', { concurrency: 2, paused: false }, { retain: 7 * 24 * 60 * 60 * 1000 });
```
https://docs.bullmq.io/guide/relay/examples/controlling-workers
