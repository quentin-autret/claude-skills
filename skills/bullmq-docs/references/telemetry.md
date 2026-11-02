# OpenTelemetry (bullmq-otel)

## Contents

- Setup
- Traces
- Metrics
- Local example

## Setup

- Telemetry comes from the separate package `bullmq-otel`. Pass a `telemetry` instance when creating each `Queue`, `Worker` and `FlowProducer`.
  https://docs.bullmq.io/guide/telemetry/getting-started and https://docs.bullmq.io/guide/telemetry/traces
- Register the OpenTelemetry SDK (tracer and meter providers) before creating BullMQ instances. The docs say to set up the meter provider before creating BullMQ instances with telemetry enabled. Load the instrumentation file first with `--import`.
  https://docs.bullmq.io/guide/telemetry/metrics and https://docs.bullmq.io/guide/telemetry/running-a-simple-example
- Use at least 6.3.11 with telemetry: the 6.3.11 changelog entry says "detach stalled checker from telemetry context to prevent trace leak".
  https://docs.bullmq.io/changelog

```typescript
const telemetry = new BullMQOtel({
  tracerName: 'my-app',
  meterName: 'my-app',
  version: '1.0.0',
  enableMetrics: true,
});
const queue = new Queue('myQueue', { connection, telemetry });
const worker = new Worker('myQueue', async job => 'some value', { connection, telemetry });
```
https://docs.bullmq.io/guide/telemetry/metrics

The positional constructor `new BullMQOtel('my-app', '1.0.0')` still works. Prefer the options object.
https://docs.bullmq.io/guide/telemetry/metrics

## Traces

- Traces need no flag. Context propagates through the job by default, so consumer spans link to the producer span.
  https://docs.bullmq.io/guide/telemetry/traces
- Per job: `telemetry.omitContext: true` starts a fresh trace at processing, `telemetry.metadata` supplies custom context.

```typescript
await queue.add('job', data, { telemetry: { omitContext: true } });
```
https://docs.bullmq.io/guide/telemetry/traces

- Span names: `{queue}.add` (PRODUCER), `{queue}.{jobName}` (CONSUMER), `{queue}.complete`, `{queue}.fail`, `{queue}.delay`, `{queue}.retry`.
  https://docs.bullmq.io/guide/telemetry/traces
- Spans can carry `bullmq.job.options` and `bullmq.job.result`. Mind this when results hold sensitive data.
  https://docs.bullmq.io/guide/telemetry/traces
- `bullmq.job.finished.timestamp` is deprecated, use `bullmq.job.attempt_finished_timestamp`. The v5 to v6 migration guide says worker spans no longer set the removed `JobFinishedTimestamp` attribute.
  https://docs.bullmq.io/guide/telemetry/traces and https://docs.bullmq.io/guide/migrations/migrate-from-v5-to-v6

## Metrics

- `enableMetrics` defaults to `false`. `tracerName` and `meterName` default to `'bullmq'`.
  https://docs.bullmq.io/guide/telemetry/metrics
- Counters `bullmq.jobs.completed|failed|delayed|retried|waiting|waiting_children`, histogram `bullmq.job.duration` (milliseconds), gauge `bullmq.queue.jobs` with attribute `bullmq.queue.jobs.state`. Job counters carry `bullmq.queue.name`, `bullmq.job.name`, `bullmq.job.state`.
  https://docs.bullmq.io/guide/telemetry/metrics
- `bullmq.jobs.failed` counts jobs failed after all retries. `bullmq.jobs.delayed` includes retry delays.
  https://docs.bullmq.io/guide/telemetry/metrics
- The gauge is recorded only when you call `queue.recordJobCountsMetric()`.
  https://docs.bullmq.io/guide/telemetry/metrics
- Custom unit or description: pre-create the instrument on `telemetry.meter` (`createCounter`, `createHistogram`, `createGauge`) before passing `telemetry` to any class. BullMQ reuses it by name, for example a histogram with `unit: 's'`.
  https://docs.bullmq.io/guide/telemetry/metrics
- A custom telemetry adapter must implement `createGauge`.
  https://docs.bullmq.io/guide/migrations/migrate-from-v5-to-v6

## Local example

Give producers and consumers their own `serviceName` and export with OTLP. Use `PeriodicExportingMetricReader` for metrics.

```typescript
const sdk = new NodeSDK({
  serviceName: 'producer',
  traceExporter: new OTLPTraceExporter({ url: 'http://127.0.0.1:4318/v1/traces' }),
  metricReader: new PeriodicExportingMetricReader({
    exporter: new OTLPMetricExporter({ url: 'http://127.0.0.1:4318/v1/metrics' }),
  }),
});
sdk.start();
// node --import producer.inst.otlp.js producer.js
```
https://docs.bullmq.io/guide/telemetry/running-a-simple-example

The docs run Jaeger locally with Docker Compose using the `jaegertracing/all-in-one` image (OTLP HTTP on port 4318, UI on port 16686).
https://docs.bullmq.io/guide/telemetry/running-jaeger
