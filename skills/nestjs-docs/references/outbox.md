# Transactional outbox (@nestjs/outbox)

## Contents
- Why
- Store and migrations
- Adding messages
- Handlers and inboxes
- Routing to microservices
- Retries, keys and dead letters
- Instances, shutdown, testing

## Why

- Saving a row then publishing (`ClientProxy.emit()`, `EventEmitter2`, CQRS `EventBus`) is a dual write: a crash loses the event, a rollback still sends it. `Outbox.add()` inserts the message in your transaction and a relay publishes committed rows with retries. Delivery is at least once, consumers deduplicate with an inbox. https://docs.nestjs.com/reliability/outbox

## Store and migrations

- Without a store the outbox runs in memory and cannot join transactions. With `NODE_ENV=production` startup fails unless `allowInMemoryStorage: true`. https://docs.nestjs.com/reliability/outbox#keep-messages-in-your-database
- An in-memory store applies `add()` at once, so a rolled-back transaction does not undo the message. https://docs.nestjs.com/reliability/outbox#the-store-contract
- Outbox tables must live in the same database as business tables. Register the store once in the root module: https://docs.nestjs.com/reliability/outbox#keep-messages-in-your-database
  ```typescript
  { provide: PostgresOutboxStore, inject: [getDrizzleToken(), OutboxStorage],
    useFactory: (db: Database, s: OutboxStorage) =>
      new PostgresOutboxStore({ executor: fromDrizzle(db) }, s) }
  ```
- The store owns schema `nest_outbox`: do not declare it in your ORM or migrations. `migrate` is on except with `NODE_ENV=production`. Apply `npx @nestjs/outbox migrate` before the new version starts and check `status` in CI. No down migrations. https://docs.nestjs.com/reliability/outbox#keep-messages-in-your-database
- Own migration tool: `PostgresOutboxStore.migrationSql()` or `npx @nestjs/outbox sql --statement-breakpoints` (breakpoints are required by PGlite). MySQL DDL is not transactional: `migrationStatements()`, one per call. https://docs.nestjs.com/reliability/outbox#keep-messages-in-your-database
- PostgreSQL default isolation must be READ COMMITTED. MySQL needs 8.4 or 9.x with strict `sql_mode` (https://docs.nestjs.com/reliability/outbox#stores). On MySQL deadlock 1213, rerun your whole transaction. https://docs.nestjs.com/reliability/outbox#keep-messages-in-your-database
- Store registration must happen in a singleton constructor, not a request-scoped provider, lazy module or lifecycle hook. https://docs.nestjs.com/reliability/outbox#the-store-contract

## Adding messages

```typescript
const order = await this.db.transaction(async (tx) => {
  await tx.insert(orders).values(order);
  await this.outbox.add(tx, { topic: 'order.placed', payload: order });
  return order;
});
this.outbox.notify(); // publish now instead of at the next poll
```
https://docs.nestjs.com/reliability/outbox#save-the-order-and-its-messages-in-one-transaction

- Pass the transaction handle, never the database: `outbox.add(this.db, ...)` throws `OutboxTransactionRequiredError` (TypeORM: not `dataSource.manager`). Prisma: only the interactive `$transaction(async (tx) => ...)`, whose 5 s default timeout means keeping `add()` transactions short. https://docs.nestjs.com/reliability/outbox#save-the-order-and-its-messages-in-one-transaction https://docs.nestjs.com/reliability/outbox#production-checklist
- Payloads are serialized at `add()`. Each message goes to one transport: add two messages to reach two. https://docs.nestjs.com/reliability/outbox#save-the-order-and-its-messages-in-one-transaction

## Handlers and inboxes

```typescript
@OnOutboxMessage('order.placed', { consumer: 'order-confirmation-email' })
async sendConfirmation(order: Order) {}
```
https://docs.nestjs.com/reliability/outbox#handle-orderplaced-in-process

- `consumer` is required and must stay stable: renaming makes every past message look new. Handlers must be singletons. https://docs.nestjs.com/reliability/outbox#handle-orderplaced-in-process
- The default inbox records the id after success, so a crash between side effect and record repeats the effect. For writes to the same database use exactly-once: `ctx.processInTransaction(tx, work)` inside `db.transaction()`, awaited. https://docs.nestjs.com/reliability/outbox#handle-orderplaced-in-process
- Handlers get `publishTimeout` (a third of `lease` by default, https://docs.nestjs.com/reliability/outbox#module-options). Pass `ctx.signal` to I/O. Throw `NonRetryableMessageError` to dead-letter at once. https://docs.nestjs.com/reliability/outbox#handle-orderplaced-in-process
- A topic with no handler is retried, not dead-lettered (rolling deploys). https://docs.nestjs.com/reliability/outbox#retries-and-the-dead-letter-queue
- Nothing prunes inboxes: run `inbox.prune('30d')` from a cron job. https://docs.nestjs.com/reliability/outbox#production-checklist

## Routing to microservices

- `transports` holds classes and sits next to `useFactory` in `forRootAsync()`. `route(message)` picks a transport, `local` is built in. https://docs.nestjs.com/reliability/outbox#register-the-outbox
  ```typescript
  OutboxModule.forRootAsync({
    transports: { analytics: ClientProxyTransport(ANALYTICS_SERVICE) },
    useFactory: () => ({ route: (m) => (m.topic.startsWith('analytics.') ? 'analytics' : 'local') }),
  })
  ```
- Over TCP, Redis and core NATS, "published" only means bytes left the process. For events you cannot lose use Kafka, RabbitMQ with publisher confirms or NATS JetStream. https://docs.nestjs.com/reliability/outbox#delivery-guarantees https://docs.nestjs.com/reliability/outbox#publish-to-the-analytics-microservice
- Publish to BullMQ from an `OutboxTransport` with the message `id` as `jobId`: BullMQ's Postgres backend cannot join your transaction. https://docs.nestjs.com/reliability/outbox#using-bullmq-on-postgres-instead

## Retries, keys and dead letters

- Defaults: 20 attempts, backoff `'1s'` doubling up to `'5m'` with equal jitter (30 to 60 min in total). Then the message is dead-lettered and stops holding back its key. https://docs.nestjs.com/reliability/outbox#module-options https://docs.nestjs.com/reliability/outbox#production-checklist https://docs.nestjs.com/reliability/outbox#retries-and-the-dead-letter-queue
- Messages sharing a `key` publish in commit order, one at a time. A key is shared across topics and transports: give each ordered stream its own key, or an email outage blocks analytics. https://docs.nestjs.com/reliability/outbox#production-checklist https://docs.nestjs.com/reliability/outbox#publish-to-the-analytics-microservice
- `OutboxDeadLetters.requeue()` keeps the id, so consumers skip what they already handled. Filters refuse an empty target (`all: true` for everything). No HTTP routes are mounted: protect your own. https://docs.nestjs.com/reliability/outbox#retries-and-the-dead-letter-queue

## Instances, shutdown, testing

- API-only instances set `OUTBOX_RELAY=off`. `notify()` wakes only the local relay. https://docs.nestjs.com/reliability/outbox#run-several-instances
- `app.enableShutdownHooks()` lets the relay finish in-flight publishes in `onModuleDestroy()`. https://docs.nestjs.com/reliability/outbox#run-several-instances https://docs.nestjs.com/reliability/outbox#production-checklist
- Catch errors in `events$` and diagnostics subscribers: a throwing subscriber crashes the process. Alert on `relay.stats().lagMs` and `deadLetters`. https://docs.nestjs.com/reliability/outbox#run-several-instances https://docs.nestjs.com/reliability/outbox#production-checklist
- Tests: PGlite, `OUTBOX_RELAY=off`, `relay.runOnce()`, `vi.spyOn(Date, 'now')` for backoff. https://docs.nestjs.com/reliability/outbox#testing
