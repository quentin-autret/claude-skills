# Syncing and reindexing

Keeping Typesense in step with a primary database, high-volume writes, backpressure errors and zero-downtime reindexing in Typesense 30.2.

## Contents

- Sync strategies
- High-volume writes
- 503 and resource errors
- Zero-downtime reindex with an alias
- In-place reindex
- Database plus search index layout

## Sync strategies

- Bulk polling: an `updated_at` on every row, a soft-delete flag (`is_deleted`) or a table of deleted ids. Every ~30s, read rows changed since `last_synced_at` (persisted durably), import them with `action=upsert`, then delete the deleted ids with `filter_by: 'id:=[id1,id2,id3]'`. A database view joining several tables can be polled instead.
- Change listeners (triggers, CDC such as Debezium on the PostgreSQL WAL, MySQL binlogs, MongoDB, DynamoDB, Firestore) or ORM `on_save` hooks: push changes into a queue, a job bulk-imports the queue every ~5s.
- Airbyte supports Typesense as a destination.
- A full nightly reindex closes gaps from validation errors, network issues and failed retries.

https://typesense.org/docs/guide/syncing-data-into-typesense.html and https://typesense.org/docs/guide/postgres-typesense-migration-fts.html

- Real time: a single-document write is acceptable when an edit must show at once. Bulk import uses far less CPU for the same count, so prefer bulk even with an interval cut to about 2s.
  https://typesense.org/docs/guide/syncing-data-into-typesense.html

Layered pattern from the Node and Prisma guide:
- real-time upsert on each CRUD write, a periodic worker (every 60s) on `updated_at > lastSyncTime` as safety net, and a manual full sync after outages.
- keyset pagination (`where id > lastId`, `take: 1000`, `orderBy id asc`) with one `import(docs, { action: 'upsert' })` per batch.
- deletions read from `deleted_at > lastSyncTime`. Hard-deleted rows are invisible to the polling query.
- at startup, a full sync when the collection has `num_documents === 0`, otherwise an incremental sync.
- the guide's samples set `connectionTimeoutSeconds: 5` and never inspect `success: false` lines. Raise the timeout for imports and check each line.

https://typesense.org/docs/guide/node-prisma-search-api.html

## High-volume writes

- Beyond tens of writes per second, switch to bulk import. 10,000 single calls cost an order of magnitude more CPU than one call with 10,000 documents.
- Buffer table pattern: the app writes `record_id`, `operation_type`, `record_data`, `created_at`, `processed` into a table and moves on. A job every 5 to 10s groups unprocessed rows, imports inserts and updates with `action=upsert`, applies deletes, then marks rows processed.
- Concurrent import workers: at most `N-2`, N being the CPU cores available to Typesense (6 workers on 8 vCPU), so searches keep capacity.
- A large backlog of pending write batches usually means too many single-document writes.

https://typesense.org/docs/guide/syncing-data-into-typesense.html and https://typesense.org/docs/guide/faqs.html

## 503 and resource errors

- `503 Not Ready or Lagging` under heavy writes is built-in backpressure that protects reads. Remedies in order:
  1. at least 4 CPU cores for high-volume writes
  2. a client timeout up to ~60 minutes
  3. a longer retry interval, or a random 10 to 60s for jitter (serverless especially)
  4. single writes moved to bulk import
  5. display-only fields left out of the schema
  6. faster disks for large documents
  7. last, the server parameters `healthy-read-lag` and `healthy-write-lag`
- `Rejecting write: running out of resource type: OUT_OF_MEMORY` means add RAM. `OUT_OF_DISK` means add disk, then restart Typesense.
- A restarted node returns 503 until its data is reloaded into RAM. An HA cluster serves meanwhile.
- Typesense reports memory per process only (`GET /metrics.json`), never per collection or field.

https://typesense.org/docs/guide/syncing-data-into-typesense.html and https://typesense.org/docs/guide/faqs.html

## Zero-downtime reindex with an alias

- Name physical collections with a version suffix (`products_v1`, `products_v2`). The alias carries the clean name `products`. Collections and aliases share one namespace.
- Application code and the search UI always use the alias, for searches and for writes.

1. Create the new versioned collection with the schema.
2. Bulk import into it.
3. Point the alias to it, an atomic switch.
4. Drop the old collection to free memory and disk.

```js
await client.aliases().upsert('products', { collection_name: 'products_v2' })
await client.collections('products_v1').delete()
```
https://typesense.org/docs/guide/syncing-data-into-typesense.html

- Writes sent to the alias land in the collection it points to at that moment. During the rebuild, also write changes to the new collection, as the collections page does with its dual-write step.
  https://typesense.org/docs/guide/syncing-data-into-typesense.html and https://typesense.org/docs/30.2/api/collections.html
- Collections joined by a `reference` must be reindexed together when one of them is swapped through an alias, because references store internal ids.
  https://typesense.org/docs/30.2/api/joins.html

## In-place reindex

Bulk upsert every record with a `last_synced_at` timestamp or batch id, then delete stale ones by query, for example `filter_by: 'last_synced_at:<1700000000'`.
https://typesense.org/docs/guide/syncing-data-into-typesense.html

## Database plus search index layout

- The primary database stays the source of truth. Typesense indexes only the fields needed to search, filter and render. Search returns ids and the app reads full rows from the database when needed.
- A backend proxy gives control over response shape, business logic, extra auth and rate limits. It costs a network hop, and Typesense Cloud geo routing then sees the backend location instead of the user.

https://typesense.org/docs/guide/node-prisma-search-api.html and https://typesense.org/docs/guide/postgres-typesense-migration-fts.html

| PostgreSQL habit | Typesense equivalent | Source |
| --- | --- | --- |
| `ILIKE '%x%'`, `pg_trgm` | `q` + `query_by`, prefix on by default | https://typesense.org/docs/guide/postgres-typesense-migration-fts.html |
| trigram plus tsvector for typos | typo tolerance on by default | https://typesense.org/docs/guide/postgres-typesense-migration-fts.html |
| per-column weights | `query_by_weights` | https://typesense.org/docs/guide/postgres-typesense-migration-fts.html |
| `GROUP BY` counts | `facet_by` | https://typesense.org/docs/guide/postgres-typesense-migration-fts.html |
