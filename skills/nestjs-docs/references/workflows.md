# Durable workflows (@nestjs/workflows)

## Contents
- When to use
- Registration and store
- Writing a workflow
- Starting, signals, timers
- Rules for workflow code
- Deploying changes
- Operating and production

## When to use

- Queues (BullMQ) fit one unit of background work. CQRS sagas react in memory and lose state on restart. Workflows fit multi-step processes that wait hours or days, survive deploys and undo completed steps. https://docs.nestjs.com/reliability/workflows#workflows-queues-or-sagas
- A workflow instance is a row in your database, so `start()` can join the transaction that saves the order. A job queue writes through its own connection, so saving an order and enqueueing its job are two writes. https://docs.nestjs.com/reliability/workflows#workflows-queues-or-sagas
- A workflow is one `async` method. After a restart it runs again from the top and completed steps return their journaled results. https://docs.nestjs.com/reliability/workflows

## Registration and store

- Import `WorkflowsModule` once in the root module. Workflow classes are ordinary providers. The worker runs in the app process. `worker: false` on API-only processes. https://docs.nestjs.com/reliability/workflows#register-the-module
- Without a store everything is in memory. With `NODE_ENV=production` and no store startup fails unless `allowInMemoryStorage: true`. Register `PostgresWorkflowStore` (or `MySqlWorkflowStore`) through `WorkflowStorage`: https://docs.nestjs.com/reliability/workflows#keep-workflows-in-your-database
  ```typescript
  { provide: PostgresWorkflowStore, inject: [getDrizzleToken(), WorkflowStorage],
    useFactory: (db: Database, s: WorkflowStorage) =>
      new PostgresWorkflowStore({ executor: fromDrizzle(db) }, s) }
  ```
- The store owns schema `nest_workflows`. `migrate` is off with `NODE_ENV=production`: run `npx @nestjs/workflows migrate` before deploy, `status` in CI. https://docs.nestjs.com/reliability/workflows#keep-workflows-in-your-database
- PostgreSQL needs READ COMMITTED (also for a `signal()` in your transaction). MariaDB is refused. https://docs.nestjs.com/reliability/workflows#keep-workflows-in-your-database
- Steps default to 3 attempts, backoff from 1 s doubling to 5 min. https://docs.nestjs.com/reliability/workflows#register-the-module
- `app.enableShutdownHooks()` lets the worker stop, abort running steps and hand instances back within `shutdownTimeout` (keep it below the platform grace period). https://docs.nestjs.com/reliability/workflows#register-the-module https://docs.nestjs.com/reliability/workflows#production-checklist

## Writing a workflow

```typescript
@Workflow('order-fulfilment')
export class OrderFulfilmentWorkflow implements WorkflowRunner<Order, FulfilmentResult> {
  async run(ctx: WorkflowContext, order: Order): Promise<FulfilmentResult> {
    const charge = await ctx.step(
      'charge-payment',
      ({ idempotencyKey }) => this.paymentsService.charge(order, idempotencyKey),
      { retry: { attempts: 5, backoff: { delay: '2s' } },
        compensate: (charge, { idempotencyKey }) => this.paymentsService.refund(charge.chargeId, idempotencyKey) },
    );
  }
}
```
https://docs.nestjs.com/reliability/workflows#write-the-workflow

- A step runs at least once: pass its `idempotencyKey` (`<instance id>:<step name>`) to every side effect, compensations included. https://docs.nestjs.com/reliability/workflows#write-the-workflow
- Step results come back as JSON even on the first run (a `Date` is a string, typed `Journaled<T>`). https://docs.nestjs.com/reliability/workflows#write-the-workflow
- Throw `NonRetryableStepError` when retrying cannot help. Compensations run in reverse order on failure or cancel. `ctx.commit('name')` marks the point of no return. https://docs.nestjs.com/reliability/workflows#write-the-workflow https://docs.nestjs.com/reliability/workflows#sleep-for-a-week-then-ask-for-a-review
- Long steps checkpoint with `heartbeat(progress)` and must overwrite rather than append. https://docs.nestjs.com/reliability/workflows#long-running-steps

## Starting, signals, timers

```typescript
return this.db.transaction(async (tx) => {
  await tx.insert(orders).values(order);
  const fulfilment = await this.workflowClient.start(OrderFulfilmentWorkflow, order, {
    id: fulfilmentId(order.id), transaction: tx,
  });
  return { ...order, fulfilment };
});
```
https://docs.nestjs.com/reliability/workflows#start-it-with-the-order

- Derive the id from a business key so `start()` is idempotent (same input returns `created: false`, other input throws `WorkflowIdConflictError`). https://docs.nestjs.com/reliability/workflows#start-it-with-the-order
- Define signals once: `new WorkflowSignal<CarrierEvent>('shipment.delivered')`. `ctx.waitForSignal('await-delivery', sig, { key, timeout: '3d' })` resolves `null` on timeout. Send with `signal(sig, event, { key, id, transaction: tx })`: `id` deduplicates retried webhooks. Signals arriving before the wait still count. https://docs.nestjs.com/reliability/workflows#wait-for-the-delivery-webhook
- `ctx.sleep('name', '7d')` parks the instance. `ctx.waitForAny()` picks the first of several signals or timers. https://docs.nestjs.com/reliability/workflows#sleep-for-a-week-then-ask-for-a-review https://docs.nestjs.com/reliability/workflows#wait-for-whichever-comes-first
- `startAndWait()` refuses `transaction`: start in the transaction, `result(id)` after commit. A wait timeout leaves the instance running (answer 202). https://docs.nestjs.com/reliability/workflows#wait-for-a-result
- Concurrency, rate limit and priority: instances without a priority run before prioritized ones, so once some have one give all one. https://docs.nestjs.com/reliability/workflows#limit-how-many-run-at-once

## Rules for workflow code

- Anything touching the outside world goes in a step. Outside steps use `ctx.now()`, `ctx.random()`, `ctx.uuid()`. Only await `ctx` operations. https://docs.nestjs.com/reliability/workflows#rules-for-workflow-code
- Names of steps, sleeps and waits are unique per run (add the loop index). No `ctx` calls inside a step. No per-run state on `this` (the class is a singleton). https://docs.nestjs.com/reliability/workflows#rules-for-workflow-code
- `Promise.all()` over steps is fine. `Promise.race()` and `Promise.any()` are not replay-safe. Rethrow errors you do not handle (`isWorkflowInterrupt()`). https://docs.nestjs.com/reliability/workflows#rules-for-workflow-code

## Deploying changes

- Step names are a contract: renaming `charge-payment` would charge again under a key the provider has never seen, so the engine instead fails the instance, without compensating, when code no longer reaches journaled steps (`WorkflowNonDeterminismError`). https://docs.nestjs.com/reliability/workflows#deploy-changes-safely https://docs.nestjs.com/reliability/workflows#production-checklist
- Without a new version you may change a step body without changing meaning, append steps after the last reached point, and change retries or timeouts. Anything else is `@Workflow('name', { version: 2 })`, with both classes registered until version 1 has no unfinished instances. https://docs.nestjs.com/reliability/workflows#deploy-changes-safely
- Declared schedules belong to the code: a schedule no process declares for 5 minutes is deleted. `missed` defaults to `'skip'`. https://docs.nestjs.com/reliability/workflows#run-workflows-on-a-schedule

## Operating and production

- `cancel()` runs compensations, `terminate()` skips them, `retry()` reruns a failed instance only if no compensation ran. Protect these routes with your own guard. https://docs.nestjs.com/reliability/workflows#retry-delete-or-terminate
- Finished instances stay until `purge({ olderThan: '30d' })`, run nightly with `@OnOneInstance()`. https://docs.nestjs.com/reliability/workflows#purge-finished-instances
- Journals fail at 10,000 entries or 10 MB: continue endless processes in a new instance. https://docs.nestjs.com/reliability/workflows#keep-journals-short
- Encrypt with `codec: new AesGcmPayloadCodec({ keys, current })`. Ids, names and keys stay readable: keep them free of personal data. https://docs.nestjs.com/reliability/workflows#encrypt-what-workflows-store
- Alert on `compensation_failed` and on `WorkflowNonDeterminismError`. https://docs.nestjs.com/reliability/workflows#production-checklist
- Tests: `ManualWorkflowClock` (`advance('3d')`), worker disabled, `WorkflowWorker.drain()`. https://docs.nestjs.com/reliability/workflows#test-with-a-manual-clock
