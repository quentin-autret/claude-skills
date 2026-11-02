# Events, task scheduling and queues

## Contents
- Events (@nestjs/event-emitter)
- Task scheduling (@nestjs/schedule)
- Queues (@nestjs/bullmq)
- Old patterns: Bull

## Events (@nestjs/event-emitter)

- `EventEmitterModule` is global by default. Listeners register in `onApplicationBootstrap`, so events emitted from constructors or `onModuleInit()` can be lost. Wait first: https://docs.nestjs.com/application/events#preventing-event-loss
  ```typescript
  await this.eventEmitterReadinessWatcher.waitUntilReady();
  this.eventEmitter.emit('order.created', new OrderCreatedEvent({ /* ... */ }));
  ```
- `suppressErrors` defaults to `true`: listener errors are logged, not rethrown. https://docs.nestjs.com/application/events#listening-to-events
- Wildcards need `wildcard: true`. `order.*` matches one segment only, use `**` for several. https://docs.nestjs.com/application/events#listening-to-events
- A request-scoped listener gets a new instance per event, with the payload injected as `REQUEST`. `inheritRequestContextId: true` reuses the request's sub-tree when the payload carries it. https://docs.nestjs.com/application/events#listening-to-events
- In-memory emitters (`EventEmitter2`, CQRS `EventBus`, `ClientProxy.emit()`) publish outside your database transaction: a crash loses the event and a rollback still sends it. Use the transactional outbox for events that must follow a commit. https://docs.nestjs.com/reliability/outbox

## Task scheduling (@nestjs/schedule)

- Call `ScheduleModule.forRoot()` in one module only. Each extra import registers every `@Cron()`, `@Interval()` and `@Timeout()` again, so jobs run several times. https://docs.nestjs.com/application/task-scheduling#installation
- Every process runs every job: 3 instances run `@Cron('0 2 * * *')` 3 times a night. Add `@OnOneInstance()` from `@nestjs/locks` with a shared lock store: https://docs.nestjs.com/application/task-scheduling#running-on-several-instances
  ```typescript
  @Cron('0 2 * * *')
  @OnOneInstance({ key: 'invoices:nightly-export' })
  async exportInvoices() {}
  ```
- Cron patterns accept an optional leading seconds field (6 positions). `@Cron(new Date(...))` runs once. https://docs.nestjs.com/application/task-scheduling#declarative-cron-jobs
- `waitForCompletion: true` skips ticks while a run is in progress. It stops a job from overlapping itself on one instance only. `timeZone` and `utcOffset` must not be combined. https://docs.nestjs.com/application/task-scheduling#declarative-cron-jobs https://docs.nestjs.com/reliability/locks
- Handlers are wrapped in try/catch and errors only logged. A job that silently stops (crash, typo in the expression) goes unnoticed without monitoring. https://docs.nestjs.com/application/task-scheduling#knowing-a-cron-job-actually-ran
- Name jobs to reach them through `SchedulerRegistry` (`getCronJob(name).stop()`). `nextDate()` throws for a job with no future date and returns a Luxon `DateTime`. https://docs.nestjs.com/application/task-scheduling#dynamic-cron-jobs
- `@Interval()` takes milliseconds. `@Timeout()` counts from application startup. https://docs.nestjs.com/application/task-scheduling#declarative-intervals
- Tests can fire a job with `schedulerRegistry.getCronJob(name).fireOnTick()`. https://docs.nestjs.com/reliability/locks#testing

## Queues (@nestjs/bullmq)

- BullMQ is installed with `@nestjs/bullmq bullmq`. Bull is in maintenance mode. Both need Redis. https://docs.nestjs.com/application/queues
  ```typescript
  BullModule.forRoot({ connection: { host: 'localhost', port: 6379 } })
  BullModule.registerQueue({ name: 'audio' })
  await this.audioQueue.add('transcode', { foo: 'bar' }, { delay: 3000 });
  ```
- Consumers extend `WorkerHost` and implement `process(job)`. The Bull-style `@Process('name')` does not work in BullMQ: `switch (job.name)` inside `process()`. https://docs.nestjs.com/application/queues#consumers
  ```typescript
  @Processor('audio')
  export class AudioConsumer extends WorkerHost {
    async process(job: Job): Promise<any> {
      switch (job.name) { case 'transcode': /* ... */ }
    }
  }
  ```
- Completed and failed jobs are kept by default. Set `removeOnComplete` / `removeOnFail` (boolean, count or `KeepJobs`). https://docs.nestjs.com/application/queues#job-options
- `attempts` defaults to 1. A `jobId` that already exists is not added. https://docs.nestjs.com/application/queues#job-options
- Jobs without `priority` run before prioritized ones. Priorities cost some performance. https://docs.nestjs.com/application/queues#job-options
- BullMQ removed the `repeat` option of `Queue.add()`: use `Queue#upsertJobScheduler()`. https://docs.nestjs.com/application/queues#job-options
- `defaultJobOptions` do not apply to jobs added through a `FlowProducer`. https://docs.nestjs.com/application/queues#bullmq-installation
- Queues resume unfinished jobs from earlier sessions at startup (they persist in Redis). https://docs.nestjs.com/application/queues#bullmq-installation
- Sandboxed processors (`processors: [join(import.meta.dirname, 'processor.js')]`) run without DI: create dependencies yourself. https://docs.nestjs.com/application/queues#separate-processes
- `registerQueueAsync({ name, useFactory })`: `name` stays outside the factory. https://docs.nestjs.com/application/queues#async-configuration
- `manualRegistration: true` in `extraOptions` stops all processing until `BullRegistrar.register()` is called. https://docs.nestjs.com/application/queues#manual-registration
- Request-scoped consumers get a new instance per job, with the job under `@Inject(JOB_REF)`. https://docs.nestjs.com/application/queues#request-scoped-consumers
- `Queue.add()` writes through BullMQ's own connection, so enqueueing inside a database transaction is a dual write. Publish from the outbox with the message id as `jobId`, or enqueue from a workflow step with `{ jobId: idempotencyKey }`. https://docs.nestjs.com/reliability/outbox#using-bullmq-on-postgres-instead https://docs.nestjs.com/reliability/workflows#long-running-steps

## Old patterns: Bull

| Bull (`@nestjs/bull`) | BullMQ (`@nestjs/bullmq`) |
| --- | --- |
| `redis: { host, port }` | `connection: { host, port }` |
| `@Process('name')` per job name | one `process(job)` with `switch (job.name)` |
| `job.progress(n)` | `job.updateProgress(n)` |
| `repeat` job option | `upsertJobScheduler()` |
| `@OnQueueActive()` | `@OnWorkerEvent('active')`, `@OnQueueEvent()` on a `QueueEventsHost` |

https://docs.nestjs.com/application/queues#bull-installation

- With Bull, multiple consumers on one queue ignore `concurrency: 1`. https://docs.nestjs.com/application/queues#consumers-1
