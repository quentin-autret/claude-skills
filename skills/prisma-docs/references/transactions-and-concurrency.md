# Transactions and concurrency

## Contents
- Choosing a transaction form
- Sequential $transaction
- Interactive transactions
- Isolation levels and retries
- Races in upsert and connectOrCreate
- Optimistic concurrency control

## Choosing a transaction form

- Dependent writes: nested writes. Independent writes: bulk operations (`createMany`, `updateMany`, `deleteMany`, themselves transactional) or `$transaction([...])`. Read-modify-write: optimistic concurrency, idempotent design or an interactive transaction. https://www.prisma.io/docs/orm/v7/prisma-client/queries/transactions

## Sequential $transaction

- `$transaction([q1, q2])` runs in order, atomically, and returns an array. It cannot pass a generated id from one operation to the next: use nested writes or pre-computed ids (UUIDs). https://www.prisma.io/docs/orm/v7/prisma-client/queries/transactions
- `findUniqueOrThrow` inside the array form does not roll back the other operations when it throws. Use the interactive form for that. https://www.prisma.io/docs/orm/v7/reference/prisma-client-reference#finduniqueorthrow
- Raw `$executeRaw` calls batch in it too. https://www.prisma.io/docs/orm/v7/prisma-client/queries/transactions

## Interactive transactions

```ts
await prisma.$transaction(async (tx) => {
  const posts = await tx.post.findMany();
  for (const post of posts)
    await tx.post.update({ where: { id: post.id }, data: { status: post.published ? "Published" : "Unknown" } });
});
```

https://www.prisma.io/docs/guides/v7/database/data-migration

- Any call made on the `tx` instance is part of the transaction. Reaching the end of the function commits, throwing rolls back. https://www.prisma.io/docs/orm/v7/prisma-client/queries/transactions
- Keep it short: long-running transactions hurt performance and can cause deadlocks, so avoid network requests and slow queries inside. https://www.prisma.io/docs/orm/v7/prisma-client/queries/transactions
- `Promise.all` inside does not parallelise: one transaction is one connection running one query at a time. https://www.prisma.io/docs/orm/v7/prisma-client/queries/transactions
- `maxWait` defaults to 2000 ms (time to acquire), `timeout` to 5000 ms (then rollback). Set global defaults with `transactionOptions` on the constructor, override per call. https://www.prisma.io/docs/orm/v7/reference/prisma-client-reference#transactionoptions
- A packaged extension that references the wrapped `PrismaClient` and calls a client-level method (such as `$queryRaw` or `$transaction`) runs on a new connection and ignores the current transaction when triggered from inside an interactive or batched transaction. https://www.prisma.io/docs/orm/v7/prisma-client/client-extensions/shared-extensions
- Cloudflare D1 has no transactions. https://www.prisma.io/docs/orm/v7/core-concepts/supported-databases/sqlite

## Isolation levels and retries

- Default is the database setting: ReadCommitted on PostgreSQL and SQL Server, RepeatableRead on MySQL, Serializable on CockroachDB and SQLite (the only level they support). `Snapshot` exists on SQL Server only. https://www.prisma.io/docs/orm/v7/prisma-client/queries/transactions
- For write conflicts (concurrent `deleteMany` plus `createMany`), use Serializable and retry on `P2034`: https://www.prisma.io/docs/orm/v7/prisma-client/queries/transactions
  ```ts
  await prisma.$transaction([prisma.user.deleteMany({ where: {} }), prisma.post.createMany({ data })], {
    isolationLevel: Prisma.TransactionIsolationLevel.Serializable,
  });
  // wrap in a loop: on error.code === "P2034" retry up to MAX_RETRIES, rethrow others
  ```
  https://www.prisma.io/docs/orm/v7/prisma-client/queries/transactions

## Races in upsert and connectOrCreate

- Two concurrent `upsert` calls on a missing record can both try to create, and one throws `P2002`. Catch it and retry, the retry updates. https://www.prisma.io/docs/orm/v7/reference/prisma-client-reference#unique-key-constraint-errors-on-upserts
- A database-native upsert (`INSERT ... ON CONFLICT`, no `P2002`) is used only on PostgreSQL, CockroachDB and SQLite, and only when: no nested queries in `create`/`update`, no nested read selection, one model modified, exactly one unique field in `where`, with the same value in `create`. Otherwise Prisma reads then writes. https://www.prisma.io/docs/orm/v7/reference/prisma-client-reference#database-upsert-query-criteria
- `connectOrCreate` has the same race: catch `P2002` and retry. https://www.prisma.io/docs/orm/v7/reference/prisma-client-reference#connectorcreate
- Design retryable steps as idempotent (unique constraint plus `upsert`, or look up an external resource before creating it). https://www.prisma.io/docs/orm/v7/prisma-client/queries/transactions

## Optimistic concurrency control

- Use it when you expect many concurrent requests and rare conflicts: a `version` field checked in `where`. https://www.prisma.io/docs/orm/v7/prisma-client/queries/transactions
  ```ts
  const result = await prisma.seat.updateMany({
    where: { id, version: read.version },
    data: { ...changes, version: { increment: 1 } },
  });
  // result.count === 0 means another writer won: retry or report a conflict
  ```
  https://www.prisma.io/docs/orm/v7/prisma-client/queries/transactions
- `update` also accepts `where: { id, version }` (non-unique fields beside a unique one): the database skips the update when `version` has changed. https://www.prisma.io/docs/orm/v7/reference/prisma-client-reference#optimistic-concurrency-control-on-updates
- Atomic operators (`increment`) avoid read-then-write races for counters. https://www.prisma.io/docs/orm/v7/reference/prisma-client-reference#atomic-number-operations
