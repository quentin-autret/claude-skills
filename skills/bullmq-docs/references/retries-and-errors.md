# Retries, backoff, rate limiting, timeouts, cancellation, step jobs

## Contents

- Automatic retries
- Stopping retries
- Rate limiting
- Timeouts
- Cancellation
- Moving a job from inside the processor
- Multi-step processors

## Automatic retries

- Retries are off by default. Pass `attempts` greater than 1.
  https://docs.bullmq.io/guide/retrying-failing-jobs
- Without `backoff`, a failed job retries immediately.
  https://docs.bullmq.io/guide/retrying-failing-jobs
- Built-in types: `fixed` and `exponential` (`2 ^ (attempts - 1) * delay` ms). `jitter` between 0 and 1 spreads retries, 0 by default.
  https://docs.bullmq.io/guide/retrying-failing-jobs
- Throw `Error` objects, never literals. The ESLint rule `no-throw-literal` enforces it.
  https://docs.bullmq.io/guide/retrying-failing-jobs
- Retried jobs keep their priority.
  https://docs.bullmq.io/guide/retrying-failing-jobs

```typescript
await queue.add('test-retry', { foo: 'bar' }, {
  attempts: 8,
  backoff: { type: 'exponential', delay: 3000, jitter: 0.5 },
});
```
https://docs.bullmq.io/guide/retrying-failing-jobs

Custom backoff lives in the worker `settings.backoffStrategy` and is selected with `backoff: { type: 'custom' }`. Return 0 to move the job to the end of waiting (or back to prioritized), -1 to fail it without retry. With several custom types the strategy receives `(attemptsMade, type, err, job)`.

```typescript
new Worker('foo', processor, {
  settings: { backoffStrategy: (attemptsMade: number) => attemptsMade * 1000 },
});
```
https://docs.bullmq.io/guide/retrying-failing-jobs

## Stopping retries

`throw new UnrecoverableError('reason')` fails the job at once, overriding `attempts`. It replaces `job.discard()`.
https://docs.bullmq.io/patterns/stop-retrying-jobs and https://docs.bullmq.io/changelog

`attemptsMade` does not grow for `RateLimitError`, `DelayedError`, `WaitingChildrenError` and `WaitingError`. `attemptsStarted` grows every time a job becomes active. `maxStartedAttempts` caps how many times a job may start.
https://docs.bullmq.io/patterns/stop-retrying-jobs and https://docs.bullmq.io/patterns/process-step-jobs

## Rate limiting

- `limiter: { max, duration }` is global across all workers of the queue: 10 workers with `max: 10, duration: 1000` process 10 jobs per second in total. Limited jobs stay waiting.
  https://docs.bullmq.io/guide/rate-limiting
- Manual limit (for example after a 429): `await worker.rateLimit(ms)` then `throw Worker.RateLimitError()`. The worker still needs `limiter` options because `limiter.max` enables the check.
  https://docs.bullmq.io/guide/rate-limiting
- `RateLimitError` retries ignore `attempts`. To cap them, compare `job.attemptsStarted >= job.opts.attempts` and throw `UnrecoverableError`.
  https://docs.bullmq.io/patterns/stop-retrying-jobs
- `queue.removeRateLimitKey()` ends the delay and resets the counter.
  https://docs.bullmq.io/guide/rate-limiting

```typescript
const worker = new Worker('myQueue', async () => {
  const [isRateLimited, duration] = await doExternalCall();
  if (isRateLimited) {
    await worker.rateLimit(duration);
    throw Worker.RateLimitError();
  }
}, { connection, limiter: { max: 1, duration: 500 } });
```
https://docs.bullmq.io/guide/rate-limiting

## Timeouts

BullMQ has no job timeout. Build one with `AbortController`, and throw `UnrecoverableError` to skip retries.

```typescript
const controller = new AbortController();
const timer = setTimeout(() => controller.abort(), job.data.timeout);
try {
  await doSomethingAbortable(controller.signal);
} catch (err) {
  if (err.name == 'AbortError') throw new UnrecoverableError('Timeout');
  throw err;
} finally {
  clearTimeout(timer);
}
```
https://docs.bullmq.io/patterns/timeout-jobs

## Cancellation

- The processor's 3rd parameter is `signal?: AbortSignal`. Listen to `abort` rather than polling `signal.aborted`, clean up there, and pass the signal to `fetch`.
  https://docs.bullmq.io/guide/workers/cancelling-jobs
- `worker.cancelJob(id, reason?)` reaches only jobs of that worker instance and returns true only if the job was active. Cancelling from another process needs the Relay.
  https://docs.bullmq.io/guide/workers/cancelling-jobs
- Rejecting with a plain `Error` fails the job and retries it if attempts remain. `UnrecoverableError` does not retry.
  https://docs.bullmq.io/guide/workers/cancelling-jobs
- A worker that lost its lock cannot fail the job. Cancel on `lockRenewalFailed` so the processor stops, and the stalled checker moves the job back to waiting.
  https://docs.bullmq.io/guide/workers/cancelling-jobs

```typescript
const worker = new Worker('fetchQueue', async (job, token, signal) => {
  const response = await fetch(job.data.url, { signal });
  return response.json();
});
worker.on('lockRenewalFailed', jobIds => jobIds.forEach(id => worker.cancelJob(id)));
```
https://docs.bullmq.io/guide/workers/cancelling-jobs

## Moving a job from inside the processor

Use the lock token, then throw the matching error so the worker does not complete or fail the job afterwards.

```typescript
// back to wait
await job.moveToWait(token);
throw new WaitingError();

// delay
await job.moveToDelayed(Date.now() + 200, token);
throw new DelayedError();

// wait for children added during processing
if (await job.moveToWaitingChildren(token)) {
  throw new WaitingChildrenError();
}
```
https://docs.bullmq.io/patterns/manual-retrying and https://docs.bullmq.io/patterns/process-step-jobs

## Multi-step processors

Store the next step in the data with `job.updateData` so a retry resumes there. Keep jobs idempotent, atomic and simple, and split complex work with flows.
https://docs.bullmq.io/patterns/process-step-jobs and https://docs.bullmq.io/patterns/idempotent-jobs
