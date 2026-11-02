# PostgreSQL backend

- Requires PostgreSQL 13 or newer (14+ recommended). `pg` is an optional peer dependency.
  https://docs.bullmq.io/guide/postgresql
- The TypeScript SDK does not migrate on connect (the Python, Elixir and .NET SDKs do). Run `runMigrations(client)` from one deployment step. It is idempotent and concurrency-safe (advisory lock per schema).
  https://docs.bullmq.io/guide/postgresql and https://docs.bullmq.io/quick-start

```typescript
await runMigrations(client); // pg client from a Pool, once per deployment
const queue = new Queue('foo', { connection }, createPostgresBackend);
const worker = new Worker('foo', async job => {}, { connection }, createPostgresBackend);
```
https://docs.bullmq.io/quick-start and https://docs.bullmq.io/guide/postgresql

## Typing trap

`new Queue<MyData, MyResult>(name, opts, createPostgresBackend)` does not compile: the backend and connection type parameters fall back to the Redis defaults. Use `withBackend`, or list every type argument.

```typescript
const { Queue, Worker } = withBackend(createPostgresBackend);
const queue = new Queue<MyData, MyResult>('my-queue', opts);
```
https://docs.bullmq.io/guide/postgresql

`setDefaultBackendFactory(createPostgresBackend)` changes runtime behavior only. TypeScript infers nothing from it.
https://docs.bullmq.io/guide/postgresql

## Schema and pool

- Default schema is `bullmq`. Prefer it and a dedicated database.
  https://docs.bullmq.io/guide/postgresql
- The `schema` option applies only when BullMQ builds the pool. With your own `pg.Pool`, set `search_path` yourself.
  https://docs.bullmq.io/guide/postgresql
- `close()` closes a pool BullMQ built, never a pool you passed.
  https://docs.bullmq.io/guide/postgresql
- Size pool `max` and server `max_connections` for every worker, queue and queue-events instance, plus one dedicated `LISTEN` connection each.
  https://docs.bullmq.io/guide/postgresql

## Versions

- `skipVersionCheck: true` bypasses the minimum server check (`UnsupportedPostgresVersionError`), at your own risk.
  https://docs.bullmq.io/guide/postgresql
- A client older than the recorded schema major fails with `SchemaVersionMismatchError`. Schema downgrades are not supported.
  https://docs.bullmq.io/guide/postgresql

## Performance

- Processing throughput is usually 1.5 to 2 times lower than Redis. `addBulk` and flows come much closer.
  https://docs.bullmq.io/guide/postgresql
- Keep the database close to workers (same host or zone). Consider `synchronous_commit = off` or `local` only if durability allows. Let `removeOnComplete` and `removeOnFail` bound table growth.
  https://docs.bullmq.io/guide/postgresql
- On PostgreSQL, the TypeScript SDK adds an `addBulk` batch in a single database transaction, so a failed batch is rolled back. On Redis, the TypeScript SDK pipelines individual additions, so a failure can leave some jobs added.
  https://docs.bullmq.io/guide/queues/adding-bulks
