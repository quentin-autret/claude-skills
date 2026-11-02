# Events, custom events, built-in metrics, Prometheus

## Contents

- QueueEvents
- Custom events
- Built-in metrics
- Prometheus

## QueueEvents

- Default: `QueueEvents` for anything that must see every worker (dashboards, websockets, SSE). `Worker` and `Queue` listeners see only events of that instance.
  https://docs.bullmq.io/guide/events
- Events carry `jobId`, not the job. Load it with `Job.fromId` when needed.
  https://docs.bullmq.io/quick-start
- `QueueEvents` reads a Redis stream, so events survive short disconnections, unlike pub/sub.
  https://docs.bullmq.io/guide/events
- The stream is auto-trimmed to about 10,000 events. Change it with `streams.events.maxLen`, or trim by hand with `queue.trimEvents(n)`, which leaves `n` events.
  https://docs.bullmq.io/guide/events
- Progress: `job.updateProgress(number | object)` in the worker, `progress` event on `QueueEvents`.
  https://docs.bullmq.io/guide/events
- Close it on shutdown to free its connection.
  https://docs.bullmq.io/guide/events

```typescript
const queueEvents = new QueueEvents('Paint', { connection });
queueEvents.on('completed', ({ jobId, returnvalue }) => {});
queueEvents.on('failed', ({ jobId, failedReason }) => {});
queueEvents.on('progress', ({ jobId, data }: { jobId: string; data: number | object }) => {});
await job.updateProgress({ completed: 3, total: 10 });
await queueEvents.close();
```
https://docs.bullmq.io/guide/events

## Custom events

`QueueEventsProducer.publishEvent` plus `QueueEvents` form a distributed event emitter. Only `eventName` is required. Some event names are reserved by the `QueueListener` interface: do not reuse built-in event names.

```typescript
const producer = new QueueEventsProducer(queueName, { connection });
const queueEvents = new QueueEvents(queueName, { connection });
interface CustomListener extends QueueEventsListener {
  example: (args: { custom: string }, id: string) => void;
}
queueEvents.on<CustomListener>('example', async ({ custom }) => {});
await producer.publishEvent<CustomEventPayload>({ eventName: 'example', custom: 'value' });
```
https://docs.bullmq.io/guide/events/create-custom-events

## Built-in metrics

- Enable on workers with `metrics: { maxDataPoints }`. Recommended: `MetricsTime.ONE_WEEK * 2`, about 120 KB of RAM per queue. Old points are disposed automatically.
  https://docs.bullmq.io/guide/metrics
- Use the same `metrics` setting on every worker of a queue, or the numbers are inconsistent.
  https://docs.bullmq.io/guide/metrics
- Counts are per 1-minute bucket (`data[]` has one entry per minute). `meta.count` is the total since the queue started, not the count of the interval. `meta.prevTS` and `meta.prevCount` are internal.
  https://docs.bullmq.io/guide/metrics

```typescript
const worker = new Worker('Paint', processor, {
  connection,
  metrics: { maxDataPoints: MetricsTime.ONE_WEEK * 2 },
});
const metrics = await queue.getMetrics('completed', 0, MetricsTime.ONE_WEEK * 2);
```
https://docs.bullmq.io/guide/metrics

## Prometheus

Serve `await queue.exportPrometheusMetrics(globalVariables?)` on a `/metrics` endpoint. `globalVariables` adds labels to every metric. The gauge is `bullmq_job_count`, one series per state.

```typescript
const text = await queue.exportPrometheusMetrics({ env: 'Production', server: '1' });
// bullmq_job_count{queue="my-queue", state="waiting"} 5
```
https://docs.bullmq.io/guide/metrics/prometheus
