# Deduplication

Default: a custom `jobId` when the duplicate window is the lifetime of the job record. Use the `deduplication` option when the window must end at completion, after a TTL, or must keep the latest payload.
https://docs.bullmq.io/guide/jobs/job-ids and https://docs.bullmq.io/guide/jobs/deduplication

The `deduplication.id` must represent the job: hash the data or the subset that matters.
https://docs.bullmq.io/guide/jobs/deduplication

## Modes

```typescript
// Simple: ignored until the first job completes or fails
{ deduplication: { id: 'customValue' } }
// Throttle: ignored for ttl ms
{ deduplication: { id: 'customValue', ttl: 5000 } }
// Debounce: the last payload wins
{ deduplication: { id: 'customValue', ttl: 5000, extend: true, replace: true }, delay: 5000 }
// Keep last if active
{ deduplication: { id: `deploy-${repo}`, keepLastIfActive: true } }
```
https://docs.bullmq.io/guide/jobs/deduplication

- Debounce needs `delay` plus `ttl`, `extend: true` and `replace: true`. 10 adds produce 1 completed job holding the last payload.
  https://docs.bullmq.io/guide/jobs/deduplication
- `keepLastIfActive: true` suits work that must run at least once after each new data arrival but never in parallel for the same id (sync, cache invalidation, deploys). At most 1 active and 1 waiting job per id, only the latest data kept. It ignores `ttl`: the key lives while the job exists and is cleaned when it completes or fails. If the active job retries, the next job is created only after final completion or exhausted retries.
  https://docs.bullmq.io/guide/jobs/deduplication
- `keepLastIfActive` with `delay` gives a continuous debounce: the waiting delayed job is replaced by the latest data and requeued with the original delay when the active job ends.
  https://docs.bullmq.io/guide/jobs/deduplication

## Pitfalls

- Any manual deletion (`job.remove()`) ends deduplication for that job.
  https://docs.bullmq.io/guide/jobs/deduplication
- Deduplication is not available in job scheduler templates: a scheduler adds the next job as soon as the previous one becomes active, and deduplication would block it.
  https://docs.bullmq.io/patterns/deduplication
- Never set `deduplication` on flow nodes: a deduplicated child may not attach to its parent, so the parent can run before that work completes. Parent flow nodes already reject it.
  https://docs.bullmq.io/guide/flows and https://docs.bullmq.io/changelog

## Controls and events

```typescript
await queue.getDeduplicationJobId('customValue');
await queue.removeDeduplicationKey('customValue');
await job.removeDeduplicationKey(); // in the processor: stop deduplicating once active

queueEvents.on('deduplicated', ({ jobId, deduplicationId, deduplicatedJobId }, id) => {});
```
`jobId` is the job kept, `deduplicatedJobId` the one ignored or replaced.
https://docs.bullmq.io/guide/jobs/deduplication and https://docs.bullmq.io/patterns/deduplication

## Deduplicating scheduler output

Schedule a trigger job and let its processor add the real job with deduplication.

```typescript
await queue.upsertJobScheduler('repeat', {
  pattern: '* * * * *',
}, { name: 'paint-trigger', data: {} });
// in the worker, for job.name === 'paint-trigger':
await queue.add('house', { color: 'white' }, { deduplication: { id: 'customValue', ttl: 90000 } });
```
https://docs.bullmq.io/patterns/deduplication and https://docs.bullmq.io/guide/job-schedulers (the three-argument form: id, repeat options, job template)

## Throttling with jobId

Identical `jobId` plus a `delay` collapses a burst of identical events into one job.

```typescript
queue.add('house', { color: 'white' }, { delay: 1000, jobId: 'house' });
```
https://docs.bullmq.io/patterns/throttle-jobs
