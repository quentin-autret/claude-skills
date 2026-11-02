# Upgrades and migrations

## Contents

- Upgrade strategy
- From Bull
- From v5 to v6
- v6 behavior changes
- Removed APIs and replacements

## Upgrade strategy

- Read the changelog before every upgrade and avoid large jumps: bugfix releases first, then feature releases, then majors.
  https://docs.bullmq.io/guide/migrations/migration-to-newer-versions
- Feature upgrades: upgrade every `Queue` and `Worker` instance first, and deploy code using the feature only once all run it. An older worker can stop working when a newer queue adds jobs using a new feature.
  https://docs.bullmq.io/guide/migrations/migration-to-newer-versions
- Destructive data changes can make rollback impossible. Default: pause queues, wait until the queued jobs are processed, upgrade all instances, unpause (less useful when the breaking change affects `Queue` instances). Drastic alternative: new queues (for example `myQueueV2`, or another Redis host).
  https://docs.bullmq.io/guide/migrations/migration-to-newer-versions

## From Bull

Bull and BullMQ diverged too far to guarantee backwards compatibility, so the safest path is not to reuse Bull queues. Create new queues (new names or a custom `prefix`), run Bull and BullMQ workers side by side, make producers add only to BullMQ queues, then retire the Bull queues once drained.
https://docs.bullmq.io/guide/migrations/bull-to-bullmq

## From v5 to v6

1. Upgrade to the latest v5.
2. Replace the repeatable-job APIs with job schedulers and recreate the schedules.
3. Remove the legacy repeatable configs from Redis.
4. Deploy v6 only when every producer and worker uses job schedulers.

Legacy repeatable metadata left in Redis makes v6 raise an error. Migrate while still on v5.
https://docs.bullmq.io/guide/migrations/migrate-from-v5-to-v6

`repeat.pattern`, `every`, `limit`, `startDate`, `endDate` and `tz` keep their names. Job name, data and options move into the template.

```typescript
// v5: await queue.add('paint', { color: 'blue' }, { repeat: { pattern: '0 15 3 * * *' }, attempts: 5 })
await queue.upsertJobScheduler(
  'paint-daily',
  { pattern: '0 15 3 * * *' },
  { name: 'paint', data: { color: 'blue' }, opts: { attempts: 5, backoff: 3000 } },
);
```
https://docs.bullmq.io/guide/migrations/migrate-from-v5-to-v6

## v6 behavior changes

- `ioredis` is an optional peer dependency. Redis users install it explicitly.
  https://docs.bullmq.io/changelog
- `Queue.resume()` and `Worker#resume()` are async. Await them.
  https://docs.bullmq.io/guide/migrations/migrate-from-v5-to-v6 and https://docs.bullmq.io/changelog
- Flow jobs without `opts.jobId` get a UUID instead of an incremental id.
  https://docs.bullmq.io/guide/migrations/migrate-from-v5-to-v6
- The `paused` state is gone from `JobType` and from the default `getJobCounts()`. Jobs of a paused queue count as waiting.
  https://docs.bullmq.io/changelog
- Deduplication is rejected on parent flow nodes.
  https://docs.bullmq.io/changelog
- Minimum Node.js is 14.17.0.
  https://docs.bullmq.io/guide/migrations/migrate-from-v5-to-v6
- Telemetry: `Queue#clean()` reports only the cleaned count, custom adapters need `createGauge`, job metrics use `state` instead of `status`.
  https://docs.bullmq.io/guide/migrations/migrate-from-v5-to-v6

## Removed APIs and replacements

| Removed | Replacement | Source |
| --- | --- | --- |
| `queue.add(..., { repeat })`, `addBulk` with `repeat`, class `Repeat` | `queue.upsertJobScheduler(id, repeatOpts, template)` | https://docs.bullmq.io/guide/migrations/migrate-from-v5-to-v6 |
| `getRepeatableJobs()`, `removeRepeatable()`, `removeRepeatableByKey()` | `getJobSchedulers()`, `removeJobScheduler(...)` | https://docs.bullmq.io/guide/migrations/migrate-from-v5-to-v6 |
| `repeat.utc: true` | `tz: 'UTC'` | https://docs.bullmq.io/guide/job-schedulers/repeat-options |
| `RepeatOptions.currentDate`, `nthDayOfWeek` | none | https://docs.bullmq.io/changelog |
| `debounce` option, `Job#debounceId`, `debounced` event | `deduplication`, `Job#deduplicationId`, `deduplicated` event | https://docs.bullmq.io/guide/migrations/migrate-from-v5-to-v6 |
| `Job#discard()` | `UnrecoverableError` | https://docs.bullmq.io/changelog |
| `Job#nextSchedulerJobId` | none | https://docs.bullmq.io/changelog |
| `Queue#client`, `Queue#redisVersion`, `Queue#databaseType`, `Worker#blockingClient`, `FlowProducer#client` | `getBackend()`, which returns the `RedisQueueBackend` giving access to the raw Redis client | https://docs.bullmq.io/changelog |
| optional `Connection` constructor parameter | optional `BackendFactory` | https://docs.bullmq.io/changelog |
| exports `Scripts`, `createScripts`, `JobJsonRaw`, `RedisJobOptions` | backend APIs | https://docs.bullmq.io/changelog |
| `TelemetryAttributes.JobFinishedTimestamp`, `JobStatus` | `JobState` | https://docs.bullmq.io/changelog |
| `IQueueBackend.reprocessJob`, `retryJobs`, `cleanJobsInSet`, `isJobInList` / `isJobInZSet` | `retryFinishedJob`, `retryFinishedJobs`, `cleanJobsByState`, `isJobInState` | https://docs.bullmq.io/guide/migrations/migrate-from-v5-to-v6 |

Older removals still met in old code: `QueueScheduler` (deprecated from BullMQ 2.0, its page is only relevant for older versions) and `limiter.groupKey` (group keys support removed from BullMQ 3.0).
https://docs.bullmq.io/guide/queuescheduler and https://docs.bullmq.io/guide/rate-limiting
