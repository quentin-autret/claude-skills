# Workers: concurrency, sandboxing, pausing, manual fetching, results

## Contents

- Basics
- Concurrency and scaling
- Sandboxed processors
- Pausing
- Manual fetching
- Returning results
- Several job kinds in one worker

## Basics

- A `Worker` starts processing at construction. Pass `autorun: false` and call `worker.run()` to start later.
  https://docs.bullmq.io/guide/workers
- Always attach `worker.on('error', ...)`. Without it, an emitted error is unhandled and the worker may stop.
  https://docs.bullmq.io/guide/workers
- Type with generics: `new Worker<MyData, MyReturn>(...)`. A 5th generic (after `string` and `RedisQueueBackend`) types progress, which must be JSON-serializable. Sandboxed processors always get the default progress type.
  https://docs.bullmq.io/guide/workers

```typescript
const worker = new Worker(queueName, async (job: Job) => {
  await job.updateProgress(42);
  return 'some value';
});
worker.on('error', err => console.error(err));
```
https://docs.bullmq.io/guide/workers

## Concurrency and scaling

- `concurrency` defaults to 1, is local to one worker, and changes at runtime with `worker.concurrency = 5`. It helps only when the processor awaits IO.
  https://docs.bullmq.io/guide/workers/concurrency
- IO-bound jobs: 100 to 300 is standard. Tune against production load.
  https://docs.bullmq.io/guide/parallelism-and-concurrency
- CPU-bound jobs: a large `concurrency` only adds overhead and lowers throughput (a slight value may still help, since BullMQ itself does IO against Redis). Add workers instead, throughput scales roughly linearly with their count.
  https://docs.bullmq.io/guide/parallelism-and-concurrency
- The recommended setup is several workers on several processes or machines, each with a high `concurrency`.
  https://docs.bullmq.io/guide/workers/concurrency
- A cap of 1 job across the fleet needs global concurrency on the queue, not worker `concurrency`.
  https://docs.bullmq.io/guide/workers/concurrency

## Sandboxed processors

- Pass a file path instead of a function to run the processor in a child process, so CPU work does not block lock renewal. Keep their concurrency sane.
  https://docs.bullmq.io/guide/workers/sandboxed-processors
- `useWorkerThreads: true` (since v3.13.0) uses worker threads instead of spawned processes. They are meant to be lighter, but not as light as expected: each thread duplicates the Node runtime.
  https://docs.bullmq.io/guide/workers/sandboxed-processors
- The processor can also be given as a `pathToFileURL(...)` URL, which the docs recommend on Windows.
  https://docs.bullmq.io/guide/workers/sandboxed-processors

```typescript
// processor file
module.exports = async (job: SandboxedJob) => { /* ... */ };
// main
const worker = new Worker(queueName, processorFile, { useWorkerThreads: true });
```
https://docs.bullmq.io/guide/workers/sandboxed-processors

- Timeouts in a sandbox: killing the process can corrupt in-flight writes, a TTL hook never fires during an infinite synchronous loop, clear timers before returning because the process is reused, and the exit code must be below 256.
  https://docs.bullmq.io/patterns/timeout-for-sandboxed-processors

## Pausing

Pausing a queue lets busy workers finish the current job, then idle. `await worker.pause()` waits for active jobs, `worker.pause(true)` returns at once while they continue. `Queue.resume()` is async since v6: await it (the docs also await `worker.resume()`).

```typescript
await worker.pause();
await worker.resume();
```
https://docs.bullmq.io/guide/workers/pausing-queues, https://docs.bullmq.io/guide/migrations/migrate-from-v5-to-v6 and https://docs.bullmq.io/guide/relay/examples/controlling-workers

## Manual fetching

A worker created without a processor (`new Worker('q', null, opts)`) processes nothing by itself. Pass a unique token to `getNextJob(token)`. There is no automatic lock renewal: finish within `lockDuration` (default 30 seconds) or call `job.extendLock`. You will usually also want to start the stalled checker yourself. Omitting the third argument of `moveToCompleted` / `moveToFailed` returns the next job.

```typescript
const worker = new Worker('my-queue', null, { lockDuration: 60000 });
await worker.startStalledCheckTimer();
const job = (await worker.getNextJob(token)) as Job;
await job.extendLock(token, 30000);
await job.moveToCompleted('some return value', token, false);
```
https://docs.bullmq.io/patterns/manually-fetching-jobs

## Returning results

Persist results inside the processor, not in a `completed` handler: the handler can fail while the job still completes. To hand results to another service robustly, add them to a dedicated results queue.
https://docs.bullmq.io/guide/returning-job-data

## Several job kinds in one worker

Named processors (a Bull feature) do not exist. Branch on `job.name`.

```typescript
const worker = new Worker('q', async job => {
  switch (job.name) {
    case 'taskType1': /* ... */ break;
    case 'taskType2': /* ... */ break;
  }
});
```
https://docs.bullmq.io/patterns/named-processor
