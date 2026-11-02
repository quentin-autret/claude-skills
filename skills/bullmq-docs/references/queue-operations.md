# Queue operations: bulk adds, global limits, reads, removal, manual retry

## Contents

- Bulk adds and batches
- Global concurrency and rate limit
- Reading jobs
- Removing jobs
- Manual retry

## Bulk adds and batches

- Default: `queue.addBulk` instead of `await queue.add` in a loop. It cuts round trips and every item stays its own job (own retries, events, completion).
  https://docs.bullmq.io/guide/queues/batches
- On Redis, `addBulk` is not all-or-nothing: TypeScript pipelines the additions. After a failure, check the queue before retrying because some jobs may exist.
  https://docs.bullmq.io/guide/queues/adding-bulks
- For atomic adds, or adds across several queues, use `flowProducer.addBulk`: all or none.
  https://docs.bullmq.io/patterns/adding-bulks
- When items must share one retry, timeout and outcome, put them in one job payload instead.
  https://docs.bullmq.io/guide/queues/batches
- Worker `concurrency` still hands one job per processor call. Native batch processing is a Pro feature.
  https://docs.bullmq.io/guide/queues/batches
- For a live UI over batched work, listen with `QueueEvents`, which sees every worker.
  https://docs.bullmq.io/guide/queues/batches

```typescript
const jobs = await queue.addBulk([
  { name: 'send', data: { invoiceId: 'A-100' } },
  { name: 'send', data: { invoiceId: 'A-101' } },
]);

const flow = new FlowProducer({ connection });
await flow.addBulk([
  { name: 'job-1', queueName: 'queueName-1', data: {} },
  { name: 'job-2', queueName: 'queueName-2', data: {} },
]);
```
https://docs.bullmq.io/guide/queues/batches and https://docs.bullmq.io/patterns/adding-bulks

## Global concurrency and rate limit

- Cap parallel jobs across every worker with the queue, not the worker. Worker `concurrency` never exceeds the global value.
  https://docs.bullmq.io/guide/queues/global-concurrency
- A worker rate limit does not override the global one. `queue.getRateLimitTtl()` returns the current TTL.
  https://docs.bullmq.io/guide/queues/global-rate-limit

```typescript
await queue.setGlobalConcurrency(4);          // also getGlobalConcurrency, removeGlobalConcurrency
await queue.setGlobalRateLimit(1, 1000);      // 1 job per second
const { max, duration } = await queue.getGlobalRateLimit();
```
https://docs.bullmq.io/guide/queues/global-concurrency and https://docs.bullmq.io/guide/queues/global-rate-limit

## Reading jobs

- States: `completed`, `failed`, `delayed`, `active`, `wait`, `waiting-children`, `prioritized`, `paused`, `repeat`.
  https://docs.bullmq.io/guide/jobs/getters
- `getJobs(types, start, end, asc)` uses inclusive indices: `getJobs(['completed'], 0, 99, true)` returns 100 jobs.
  https://docs.bullmq.io/guide/jobs/getters
- The v6 changelog says a paused queue's jobs are represented as waiting: the `paused` state is removed from `JobType` and from the default `getJobCounts()` result. The getters and removing-jobs pages still list or use `paused`, so test it on the installed version before relying on either.
  https://docs.bullmq.io/changelog

```typescript
const counts = await queue.getJobCounts('wait', 'completed', 'failed');
```
https://docs.bullmq.io/guide/jobs/getters

## Removing jobs

- `queue.drain()` removes waiting jobs only, `drain(true)` delayed ones too. Active, waiting-children, completed and failed jobs stay. Parents of the drained queue with pending children stay in `waiting-children`, those without pending children are removed. Parents in other queues stay in `waiting-children` or move to `wait`.
  https://docs.bullmq.io/guide/queues/removing-jobs
- `queue.clean(grace, limit, type)` removes jobs of one state older than `grace` ms and returns their ids.
  https://docs.bullmq.io/guide/queues/removing-jobs
- `queue.obliterate()` wipes the queue. Active jobs require `{ force: true }`. Parents in other queues stay in `waiting-children` or move to `wait`.
  https://docs.bullmq.io/guide/queues/removing-jobs
- `job.remove()` throws on a locked (active) job. If any child is locked, deletion stops. A removed completed child's value stays in the parent's processed set.
  https://docs.bullmq.io/guide/jobs/removing-job

```typescript
await queue.drain(true);
await queue.clean(60000, 1000, 'paused');
await queue.obliterate({ force: true });
```
https://docs.bullmq.io/guide/queues/removing-jobs

## Manual retry

`job.retry()` works only on `completed` or `failed` jobs (default `'failed'`). Without `resetAttemptsMade: true`, a job that exhausted its attempts fails again at once. Error codes: `-1` job missing, `-3` job not in the expected state. For a flow child it restores the parent dependency.

```typescript
const job = await Job.fromId(queue, 'job-id');
await job.retry('failed', { resetAttemptsMade: true, resetAttemptsStarted: true });
```
https://docs.bullmq.io/guide/jobs/retrying-job
