# Job options: ids, data, delay, priority, retention

## Contents

- Job data
- Job ids
- Delay
- Priority and LIFO
- Retention of finished jobs
- Shared options

## Job data

Data goes through `JSON.stringify` and `JSON.parse`. `undefined`, functions, `BigInt`, `Map`, `Set` and circular references are lost. A class instance arrives as a plain object, so `job.data.someMethod()` throws `TypeError: ... is not a function`. The `DataType` generic only describes the JSON shape and TypeScript does not warn. Pass plain objects and rebuild instances in the processor, or implement `toJSON()`.

```typescript
await queue.add('send', { token: 't', timestamp: Date.now() });
new Worker(queueName, async job => {
  const msg = new Message(job.data.token, job.data.timestamp);
});
```
https://docs.bullmq.io/guide/jobs/job-data

## Job ids

- Default ids are an increasing counter. Uniqueness and the counter are scoped per queue.
  https://docs.bullmq.io/guide/jobs/job-ids
- A second `add` with an existing `jobId` is ignored and a `duplicated` event fires. Use it to avoid duplicates.
  https://docs.bullmq.io/guide/jobs/job-ids and https://docs.bullmq.io/guide/queues/auto-removal-of-jobs
- A custom `jobId` must not contain `:` (it is split in 2). Use `-` or `_`.
  https://docs.bullmq.io/guide/jobs/job-ids
- A custom `jobId` must not be digits only: it collides with generated ids and throws `Error: Custom Id cannot be integers`. Prefix numeric business ids: `"job-123"`.
  https://docs.bullmq.io/guide/jobs/job-ids
- A removed job (auto-removal or manual) no longer counts: the same `jobId` can be added again.
  https://docs.bullmq.io/guide/queues/auto-removal-of-jobs

```typescript
await queue.add('wall', { color: 'pink' }, { jobId: customJobId });
```
https://docs.bullmq.io/guide/jobs/job-ids

## Delay

`delay` (ms) is a minimum: the job waits at least that long, and the actual start depends on worker load. `job.changeDelay(ms)` works only on jobs in the `delayed` state and reschedules relative to now.

```typescript
await queue.add('house', { color: 'white' }, { delay: 5000 });
await job.changeDelay(4000);
```
https://docs.bullmq.io/guide/jobs/delayed

## Priority and LIFO

- `priority` goes from `1` to `2097151`. A lower number is a higher priority. Same priority is FIFO.
  https://docs.bullmq.io/guide/jobs/prioritized
- Surprising default: a job with no priority (or `priority: 0`) is processed before every prioritized job.
  https://docs.bullmq.io/guide/jobs/prioritized
- Adding a prioritized job costs `O(log(n))`. Skip priorities when ordering is not needed.
  https://docs.bullmq.io/guide/jobs/prioritized
- `prioritized` is its own state: read it with `queue.getJobs(['prioritized'])` or `queue.getPrioritized()`. `getCountsPerPriority([1, 0])` counts prioritized (above 0) and waiting (0) jobs.
  https://docs.bullmq.io/guide/jobs/prioritized
- With several workers or `concurrency > 1`, jobs start in order but may finish out of order.
  https://docs.bullmq.io/guide/jobs/fifo

```typescript
await queue.add('wall', { color: 'brown' }, { priority: 5 });
await queue.add('wall', { color: 'pink' }, { lifo: true });
await job.changePriority({ priority: 1 });
await job.changePriority({ lifo: true });
```
https://docs.bullmq.io/guide/jobs/prioritized and https://docs.bullmq.io/guide/jobs/lifo

## Retention of finished jobs

- By default finished jobs are kept in the `completed` and `failed` sets, so cap them in production. The docs suggest a handful of completed jobs and many more failed ones, for example `removeOnComplete: 1000, removeOnFail: 5000`.
  https://docs.bullmq.io/guide/queues/auto-removal-of-jobs
- `true` removes every finished job regardless of its name.
  https://docs.bullmq.io/guide/queues/auto-removal-of-jobs
- The object form keeps jobs by `age` (seconds) and caps them with `count`, which protects memory during bursts. On the worker options (`removeOnComplete`/`removeOnFail` of `Worker`), `KeepJobs` also takes `limit`, which caps how many jobs one cleanup iteration removes.
  https://docs.bullmq.io/guide/queues/auto-removal-of-jobs and https://docs.bullmq.io/guide/workers/auto-removal-of-jobs
- Removal is lazy: it runs only when a new job completes or fails.
  https://docs.bullmq.io/guide/queues/auto-removal-of-jobs

```typescript
await queue.add('test', { foo: 'bar' }, {
  removeOnComplete: { age: 3600, count: 1000 },
  removeOnFail: { age: 24 * 3600 },
});
```
https://docs.bullmq.io/guide/queues/auto-removal-of-jobs

## Shared options

Set common options once with `defaultJobOptions` on the `Queue`.

```typescript
const queue = new Queue('Paint', {
  defaultJobOptions: { removeOnComplete: true, removeOnFail: 1000 },
});
```
https://docs.bullmq.io/guide/jobs/fifo

Instantiating a `Queue` only upserts a small meta key, so an existing queue is picked up as is. `queue.getMeta()` returns `{ concurrency, max, duration, maxLenEvents, paused, version }`.
https://docs.bullmq.io/guide/queues and https://docs.bullmq.io/guide/queues/meta
