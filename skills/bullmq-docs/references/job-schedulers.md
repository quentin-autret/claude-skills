# Job schedulers

## Contents

- Creating and updating
- Timing behavior
- Repeat options
- Cron patterns
- Custom repeat strategy
- Managing schedulers

## Creating and updating

- `queue.upsertJobScheduler(schedulerId, repeatOpts, template?)` creates or updates without duplicates, so the same startup code can run on every deploy. Call it again with the same id to change the repeat options or the template.
  https://docs.bullmq.io/guide/job-schedulers
- The template `{ name, data, opts }` is inherited by every produced job.
  https://docs.bullmq.io/guide/job-schedulers
- Produced jobs get a special id and no custom `jobId` is possible. Tell them apart by job name.
  https://docs.bullmq.io/guide/job-schedulers
- `upsertJobScheduler` returns the first job, in `delayed` state.
  https://docs.bullmq.io/guide/job-schedulers

```typescript
await queue.upsertJobScheduler(
  'my-scheduler-id',
  { pattern: '0 15 3 * * *' },
  {
    name: 'my-job-name',
    data: { foo: 'bar' },
    opts: { backoff: 3, attempts: 5, removeOnFail: 1000 },
  },
);
```
https://docs.bullmq.io/guide/job-schedulers

## Timing behavior

- The next job is produced only when the previous one starts processing. On a busy queue or with too few workers, jobs arrive less often than the interval.
  https://docs.bullmq.io/guide/job-schedulers
- While active, a scheduler always has exactly 1 job in `delayed`.
  https://docs.bullmq.io/guide/job-schedulers
- `every` aligns on the clock, not on the add time: `every: 2000` fires at even seconds.
  https://docs.bullmq.io/guide/job-schedulers/repeat-options
- A new scheduler runs its first repetition immediately, then follows `every`. Re-upserting an existing scheduler does not trigger an immediate run. The `immediately` option is deprecated because this is now always the behavior.
  https://docs.bullmq.io/guide/job-schedulers/repeat-options

## Repeat options

- Two predefined strategies: `every` (ms) for fixed intervals, `pattern` (cron) for calendar times. Anything else needs a custom `repeatStrategy`.
  https://docs.bullmq.io/guide/job-schedulers/repeat-strategies
- `tz` evaluates cron patterns in a given timezone, and cron handles daylight saving transitions. The legacy `utc` option is removed: use `tz: 'UTC'`.
  https://docs.bullmq.io/guide/job-schedulers/repeat-options and https://docs.bullmq.io/guide/job-schedulers/repeat-strategies
- `startDate`, `endDate` bound the schedule, `limit` caps repetitions.
  https://docs.bullmq.io/guide/job-schedulers/repeat-options

```typescript
await queue.upsertJobScheduler('utc-cron-job', { pattern: '0 15 3 * * *', tz: 'UTC' }, { name: 'utc-execution-job', data: {} });
await queue.upsertJobScheduler('limited-job', { every: 10000, limit: 10 }, { name: 'limited-execution-job' });
await queue.upsertJobScheduler('start-later-job', { every: 60000, startDate: new Date('2024-10-15T00:00:00Z') });
```
https://docs.bullmq.io/guide/job-schedulers/repeat-options

## Cron patterns

The parser accepts an optional leading seconds field. 6 fields: second minute hour day-of-month month day-of-week. `0 15 3 * * *` is 03:15 every day, `0 0 9 * * 1-5` is 09:00 Monday to Friday. Day of week accepts 0 to 7 (0 and 7 are Sunday), day of month accepts `L` for the last day.
https://docs.bullmq.io/guide/job-schedulers/repeat-strategies and https://docs.bullmq.io/guide/job-schedulers

## Custom repeat strategy

A custom `repeatStrategy` (for example RRULE) must be passed in `settings` to both the `Queue` and the `Worker`: the queue computes the first occurrence, then the worker uses its own settings. Only one strategy per queue.

```typescript
const settings = {
  repeatStrategy: (millis: number, opts: RepeatOptions, _jobName: string) => {
    const currentDate =
      opts.startDate && new Date(opts.startDate) > new Date(millis)
        ? new Date(opts.startDate)
        : new Date(millis);
    const rrule = rrulestr(opts.pattern);
    if (rrule.origOptions.count && !rrule.origOptions.dtstart) {
      throw new Error('DTSTART must be defined to use COUNT with rrule');
    }
    return rrule.after(currentDate, false)?.getTime();
  },
};
const queue = new Queue('Paint', { settings });
const worker = new Worker('Paint', async () => { doSomething(); }, { settings });
```
https://docs.bullmq.io/guide/job-schedulers/repeat-strategies

## Managing schedulers

The scheduler identifier is `key` in `JobSchedulerJson`, not its `id`, its `name` or the ids of produced jobs. `getJobSchedulers(start, end, asc)` returns the schedulers in the given range, in ascending order of next execution time when `asc` is `true`. `removeJobScheduler` returns `true` when it removed one.

```typescript
const page = await queue.getJobSchedulers(0, 9, true);
const one = await queue.getJobScheduler('test');
const [scheduler] = await queue.getJobSchedulers(0, 0);
if (scheduler) await queue.removeJobScheduler(scheduler.key);
```
https://docs.bullmq.io/guide/job-schedulers/manage-job-schedulers

Deduplication options are not available in the template. To deduplicate scheduled work, schedule a trigger job whose processor adds the real job with `deduplication`.
https://docs.bullmq.io/patterns/deduplication
