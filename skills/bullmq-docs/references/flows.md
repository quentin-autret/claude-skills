# Flows (parent and child jobs)

## Contents

- Creating flows
- Parent lifecycle
- Child failure policies
- Reading dependencies
- Removal rules
- Pitfalls

## Creating flows

- Flows are created only with `FlowProducer.add({ name, queueName, data?, prefix?, opts?, children? })`. `Queue.add` cannot build them. All jobs of the tree are added atomically, and the parent queue may differ from the child queues.
  https://docs.bullmq.io/guide/flows
- The parent reads child results with `job.getChildrenValues()`, keyed by job key.
  https://docs.bullmq.io/guide/flows
- Serial pipeline: nest 1 child per level. The deepest job runs first.
  https://docs.bullmq.io/guide/flows
- Flows supersede the older pattern where each queue enqueues the next step.
  https://docs.bullmq.io/patterns/flows

```typescript
const flowProducer = new FlowProducer({ connection });
await flowProducer.add({
  name: 'renovate-interior',
  queueName: 'renovate',
  children: [
    { name: 'paint', data: { place: 'ceiling' }, queueName: 'steps' },
    { name: 'fix', data: { place: 'floor' }, queueName: 'steps' },
  ],
});

const renovateWorker = new Worker('renovate', async job => {
  const childrenValues = await job.getChildrenValues();
  const total = Object.values(childrenValues).reduce((a, b) => a + b, 0);
  await sendInvoice(total);
});
```
https://docs.bullmq.io/guide/flows

- Queue options belong to queue instances. Pass per-queue options in the second argument of `add`, or behavior is unexpected:

```typescript
await flowProducer.add(flowJob, {
  queuesOptions: { [queueName]: { defaultJobOptions: { removeOnComplete: true } } },
});
```
https://docs.bullmq.io/guide/flows

- `flowProducer.addBulk([...])` adds several flows atomically. `flow.getFlow({ id, queueName, depth, maxChildren })` loads a tree, limit large ones with `depth` and `maxChildren`.
  https://docs.bullmq.io/guide/flows/adding-bulks and https://docs.bullmq.io/guide/flows/get-flow-tree
- Flow jobs without `opts.jobId` get a UUID, not an incremental id.
  https://docs.bullmq.io/guide/migrations/migrate-from-v5-to-v6

## Parent lifecycle

The parent stays in `waiting-children` until every child is processed successfully, then moves to `wait`.
https://docs.bullmq.io/guide/flows

## Child failure policies

By default a failed child leaves the parent waiting. Choose one option per child in `opts`:

| Option | Effect |
| --- | --- |
| `failParentOnFailure: true` | Parent fails, lazily: a worker must still pick the parent, which fails with `UnrecoverableError` `child {childKey} failed`. It propagates upward only through ancestors that also set it. |
| `continueParentOnFailure: true` | Parent becomes active as soon as this child fails, even while other children run. |
| `removeDependencyOnFailure: true` | Dependency dropped, parent moves on once no child is pending. |
| `ignoreDependencyOnFailure: true` | Same, and the failure is kept for `job.getIgnoredChildrenFailures()`. |

https://docs.bullmq.io/guide/flows/fail-parent , https://docs.bullmq.io/guide/flows/continue-parent , https://docs.bullmq.io/guide/flows/remove-dependency , https://docs.bullmq.io/guide/flows/ignore-dependency

With `continueParentOnFailure`, branch in the parent. `removeUnprocessedChildren()` removes only waiting or delayed children.

```typescript
const processor = async job => {
  const failedChildren = await job.getFailedChildrenValues(); // { [jobId]: errorMessage }
  if (Object.keys(failedChildren).length > 0) {
    await job.removeUnprocessedChildren();
  } else {
    // aggregate results
  }
};
```
https://docs.bullmq.io/guide/flows/continue-parent

`job.removeChildDependency()` detaches a child. It throws if the job has no parent. If it was the last pending child, the parent moves to `waiting`.
https://docs.bullmq.io/guide/flows/remove-child-dependency

## Reading dependencies

`job.getDependencies()` returns direct children only, paginated by per-type cursors (`processed`, `unprocessed`, `failed`, `ignored`).

```typescript
const { processed, nextProcessedCursor } = await job.getDependencies({ processed: { count: 5, cursor: 0 } });
const { failed, ignored, processed: p, unprocessed } = await job.getDependenciesCount();
```
https://docs.bullmq.io/guide/flows

## Removal rules

1. Removing a parent removes all its children.
2. Removing a child drops the parent's dependency on it. If it was the last child, the parent completes.
3. A job that is both parent and child triggers both rules.
4. If any job to remove is locked, none is removed and an exception is thrown.

https://docs.bullmq.io/guide/flows

## Pitfalls

- A custom `jobId` in a flow must not contain `:`.
  https://docs.bullmq.io/guide/flows
- Never use `deduplication` on flow nodes: a deduplicated child may not attach to its parent, which then runs too early. Parent nodes already reject it. Child opts also omit `parent` and `repeat`.
  https://docs.bullmq.io/guide/flows and https://docs.bullmq.io/changelog
