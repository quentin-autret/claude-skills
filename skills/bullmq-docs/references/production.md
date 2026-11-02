# Going to production and troubleshooting

## Contents

- Redis settings
- Shutdown and error listeners
- Retention and data
- Locks and stalled jobs
- Troubleshooting messages
- Monitoring

## Redis settings

- Set `maxmemory-policy` to `noeviction`. The docs call it the only setting that guarantees correct queue behavior. A Redis configured as a cache evicts keys and breaks BullMQ.
  https://docs.bullmq.io/guide/going-to-production
- Enable persistence yourself, many hosts do not. The docs recommend AOF (Append Only File), and one second per write is usually enough. Benchmark the impact.
  https://docs.bullmq.io/guide/going-to-production

## Shutdown and error listeners

- Attach `error` listeners on every `Worker` and `Queue`. Without one on a worker, Node.js raises an unhandled exception when an error is emitted.
  https://docs.bullmq.io/guide/going-to-production and https://docs.bullmq.io/guide/workers
- Close workers on `SIGINT` (Ctrl+C) and `SIGTERM` (what Kubernetes and PM2 send): `await worker.close()`, then other closings, then `process.exit(0)`. This minimizes stalled jobs but does not prevent them when a job outlasts the grace period.

```typescript
const gracefulShutdown = async (signal: string) => {
  await worker.close();
  process.exit(0);
};
process.on('SIGINT', () => gracefulShutdown('SIGINT'));
process.on('SIGTERM', () => gracefulShutdown('SIGTERM'));
worker.on('error', err => { /* log */ });
```
https://docs.bullmq.io/guide/going-to-production

- In TypeScript `worker.close()` waits for active jobs with no built-in timeout, so a hung job blocks it. If the process ends early, other workers recover the active jobs as stalled.
  https://docs.bullmq.io/guide/workers/graceful-shutdown
- Killed workers leave stalled jobs that other workers pick up after about 30 seconds by default.
  https://docs.bullmq.io/guide/going-to-production

## Retention and data

- Completed and failed jobs are kept forever by default. Set `removeOnComplete` and `removeOnFail`: a handful of completed, many or all failed for debugging and manual retry.
  https://docs.bullmq.io/guide/going-to-production
- Job data is stored in clear text. Keep sensitive data out of it, or encrypt that part before `add`.
  https://docs.bullmq.io/guide/going-to-production
- Delivery is at-least-once: a crash or a lost lock can run a job again. Processors must be idempotent.
  https://docs.bullmq.io/

## Locks and stalled jobs

- A job lock lasts 30 seconds by default (`lockDuration`) and the worker renews it after half that time.
  https://docs.bullmq.io/patterns/manually-fetching-jobs
- There is no "stalled" state, only a `stalled` event when a job moves from active back to waiting. `maxStalledCount` defaults to 1: past it the job fails with "job stalled more than allowable limit".
  https://docs.bullmq.io/guide/jobs/stalled
- Blocking the event loop longer than the stalled check (30 seconds by default) with CPU work starves lock renewal. Sandboxed processors (separate Node.js processes) reduce the chance of stalls. The docs say `stalledInterval` normally needs no change.
  https://docs.bullmq.io/guide/jobs/stalled and https://docs.bullmq.io/guide/workers/stalled-jobs

## Troubleshooting messages

- "Missing lock for job X. moveToFinished": the job lost its lock. Causes: worker CPU starvation, lost Redis connection, job or queue removed by force, or `maxmemory-policy` other than `noeviction`.
  https://docs.bullmq.io/guide/troubleshooting
- "Lua redis() command arguments must be strings or integers": an undefined, empty or non-string value reached a queue name or job parameter, typically from an environment variable. Validate at startup, enable `strictNullChecks` and type env vars as `string | undefined`.

```typescript
const queueName = process.env.QUEUE_NAME;
if (!queueName) throw new Error('QUEUE_NAME is not defined or is empty.');
```
https://docs.bullmq.io/guide/troubleshooting

## Monitoring

Watch queue and worker events, built-in metrics, the Prometheus export or OpenTelemetry.
https://docs.bullmq.io/guide/going-to-production
