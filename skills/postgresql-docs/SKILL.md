---
name: postgresql-docs
description: Pitfalls, surprising defaults, PostgreSQL 18 changes and recommended patterns from the official PostgreSQL 18 documentation, each backed by its postgresql.org URL. Use when writing or reviewing PostgreSQL SQL, schema design, migrations and ALTER TABLE locks, indexes, transactions, isolation levels and retries, locking, upsert and MERGE, COPY, JSONB, timestamptz, generated and identity columns, partitioning, EXPLAIN, vacuum, roles, row-level security, PL/pgSQL, full-text search, connection settings, and upgrades to PostgreSQL 18.
metadata:
  version: "PostgreSQL 18 (docs for 18.6, current)"
  source: https://www.postgresql.org/docs/18/
  fetched: "2026-11-02"
---

# PostgreSQL documentation

Rules from the PostgreSQL 18 documentation that a model is likely to get wrong: surprising defaults, lock and concurrency traps, PostgreSQL 18 changes and the patterns the docs recommend. Every rule ends with the docs URL that backs it.

## Version covered

The references follow the PostgreSQL 18 docs (minor release 18.6). Facts that only hold from 18 say "PostgreSQL 18". Check the server version first. On an older server, treat every "PostgreSQL 18" rule as unavailable.

## How to navigate

Read only the reference the task needs, from the table below. Sources: [SOURCES.md](SOURCES.md).

## Quick rules

1. In PostgreSQL 18 a generated column without a keyword is VIRTUAL, computed on read. Write `STORED` for statistics or logical replication. https://www.postgresql.org/docs/18/ddl-generated-columns.html https://www.postgresql.org/docs/18/sql-altertable.html
2. Use identity columns, not `serial`. An identity column is NOT NULL but not unique: add the PRIMARY KEY. https://www.postgresql.org/docs/18/ddl-identity-columns.html
3. Index the referencing columns of every foreign key yourself. PostgreSQL only indexes the referenced side. https://www.postgresql.org/docs/18/ddl-constraints.html
4. Plain `CREATE INDEX` blocks writes for the whole build. On live tables use `CREATE INDEX CONCURRENTLY` outside a transaction block, and drop the INVALID index it leaves on failure. https://www.postgresql.org/docs/18/sql-createindex.html
5. Add constraints to big tables as `NOT VALID`, then `VALIDATE CONSTRAINT`, which does not block writes. PostgreSQL 18 accepts this for NOT NULL too. https://www.postgresql.org/docs/18/sql-altertable.html
6. `ADD COLUMN` with a volatile default, a stored generated column or an identity rewrites the whole table under ACCESS EXCLUSIVE. A non-volatile default does not. https://www.postgresql.org/docs/18/sql-altertable.html
7. Set `lock_timeout` in migration sessions, and `statement_timeout`, `lock_timeout` and `transaction_timeout` per role or session, never globally. https://www.postgresql.org/docs/18/runtime-config-client.html
8. Upsert with `INSERT ... ON CONFLICT (cols) DO UPDATE`, which in READ COMMITTED always inserts or updates under concurrency. MERGE can fail with a unique violation when another session inserts the same key. https://www.postgresql.org/docs/18/transaction-iso.html
9. Retry the whole transaction on SQLSTATE 40001 and 40P01. PostgreSQL never retries by itself. Test the SQLSTATE, not the message. https://www.postgresql.org/docs/18/mvcc-serialization-failure-handling.html
10. Default isolation is READ COMMITTED: each statement sees a new snapshot. Multi-row business rules need SERIALIZABLE or explicit locks. https://www.postgresql.org/docs/18/transaction-iso.html https://www.postgresql.org/docs/18/applevel-consistency.html
11. For job queues use `SELECT ... FOR UPDATE SKIP LOCKED`, and `FOR NO KEY UPDATE` when keys stay unchanged. https://www.postgresql.org/docs/18/sql-select.html https://www.postgresql.org/docs/18/explicit-locking.html
12. `NOT IN` with a NULL in the list or subquery is never true. Aggregates other than `count` return NULL over zero rows. https://www.postgresql.org/docs/18/functions-comparisons.html https://www.postgresql.org/docs/18/functions-aggregate.html
13. Paging and `LIMIT` need an ORDER BY on a unique key, otherwise results differ between runs. https://www.postgresql.org/docs/18/queries-limit.html
14. Store instants as `timestamptz`. `TIMESTAMP '2004-10-19 10:23:54+02'` is a timestamp without time zone and the offset is dropped. https://www.postgresql.org/docs/18/datatype-datetime.html
15. `now()` is the transaction start time. Use `clock_timestamp()` for the real time. Never default to the string `'now'`. https://www.postgresql.org/docs/18/functions-datetime.html
16. Use `jsonb`, indexed with `GIN (col jsonb_path_ops)` for `@>` queries. The operator must apply to the indexed expression for the index to be used. https://www.postgresql.org/docs/18/datatype-json.html
17. PostgreSQL 18 adds `uuidv7()`, a time-ordered UUID, next to `gen_random_uuid()`. https://www.postgresql.org/docs/18/functions-uuid.html
18. PostgreSQL 18 `RETURNING` reads `old.col` and `new.col` in INSERT, UPDATE, DELETE and MERGE. https://www.postgresql.org/docs/18/dml-returning.html
19. `EXPLAIN ANALYZE` executes the statement: wrap writes in `BEGIN ... ROLLBACK`. In PostgreSQL 18 it includes BUFFERS automatically. https://www.postgresql.org/docs/18/sql-explain.html https://www.postgresql.org/docs/18/using-explain.html
20. A multicolumn B-tree can serve a query without its leading column in PostgreSQL 18 (skip scan), but only when that column has few distinct values. https://www.postgresql.org/docs/18/indexes-multicolumn.html
21. Keep autovacuum on and avoid VACUUM FULL. Autovacuum never analyzes partitioned tables or temporary tables: ANALYZE them yourself. https://www.postgresql.org/docs/18/routine-vacuuming.html
22. Row-level security does nothing until `ENABLE ROW LEVEL SECURITY`, and table owners bypass it unless `FORCE ROW LEVEL SECURITY`. https://www.postgresql.org/docs/18/ddl-rowsecurity.html
23. A SECURITY DEFINER function needs `SET search_path = trusted_schema, pg_temp` and EXECUTE revoked from PUBLIC in the same transaction. https://www.postgresql.org/docs/18/sql-createfunction.html
24. Granting a table does not extend to the sequences it uses, including those of `serial` columns. Set sequence privileges separately. https://www.postgresql.org/docs/18/sql-grant.html
25. Clients default to `sslmode=prefer`, which does not verify the server. Use `sslmode=verify-full`. https://www.postgresql.org/docs/18/libpq-ssl.html
26. Build dynamic SQL with `format('%I', ident)` and `EXECUTE ... USING` for values, never by concatenation. https://www.postgresql.org/docs/18/plpgsql-statements.html
27. Full-text expression indexes must use the 2-argument `to_tsvector('english', col)`, and queries must repeat that exact form. https://www.postgresql.org/docs/18/textsearch-tables.html

## References

| Reference | Read it when |
| --- | --- |
| [schema-design.md](references/schema-design.md) | Creating tables: identity and sequences, generated columns, constraints, temporal keys, domains, schemas and `search_path`, partitioning design |
| [migrations-and-locks.md](references/migrations-and-locks.md) | Writing a migration: ALTER TABLE lock levels and rewrites, NOT VALID, concurrent index builds, REINDEX, partition attach and detach |
| [data-types.md](references/data-types.md) | Choosing numeric, text, bytea, enum, UUID, array or range types, exclusion constraints on ranges |
| [datetime.md](references/datetime.md) | Anything with timestamps, time zones, intervals, `extract`, `date_trunc`, `to_char`, `to_timestamp` |
| [json.md](references/json.md) | Storing, querying, indexing or updating JSONB, jsonpath, JSON_VALUE, JSON_TABLE, JSON aggregates |
| [queries.md](references/queries.md) | SELECT logic: NULL semantics, joins, LATERAL, grouping, aggregates, window functions, paging, UNION, CTEs |
| [writes.md](references/writes.md) | INSERT ON CONFLICT, UPDATE and DELETE joins, batching large changes, RETURNING, MERGE, COPY, bulk loading |
| [indexes.md](references/indexes.md) | Choosing or reviewing an index: types, column order, skip scan, partial, expression, INCLUDE, pattern_ops, GIN, BRIN |
| [concurrency.md](references/concurrency.md) | Isolation levels, retries, row and table locks, deadlocks, advisory locks, timeouts, LISTEN and NOTIFY |
| [performance.md](references/performance.md) | Reading EXPLAIN, statistics and extended statistics, vacuum, memory and planner settings, parallel query, pg_stat_statements |
| [security.md](references/security.md) | GRANT and default privileges, roles, row-level security, SECURITY DEFINER, passwords, TLS and libpq connection parameters |
| [plpgsql-triggers.md](references/plpgsql-triggers.md) | Writing functions, PL/pgSQL, dynamic SQL, exceptions, triggers, procedures |
| [text-search.md](references/text-search.md) | String functions, LIKE and regex, quoting, full-text search with tsvector, ranking, `ts_headline` |

## PostgreSQL 18 changes to keep in mind when upgrading

- `initdb` enables data checksums by default, and `pg_upgrade` needs matching settings. https://www.postgresql.org/docs/18/release-18.html
- VACUUM and ANALYZE on a partitioned table also process its partitions. Write `ONLY` for the parent alone. https://www.postgresql.org/docs/18/release-18.html
- `pg_upgrade` keeps planner statistics but not extended statistics. https://www.postgresql.org/docs/18/release-18.html
- AFTER triggers run as the role active when the event was queued. https://www.postgresql.org/docs/18/release-18.html
- Asynchronous I/O (`io_method`, default `worker`), and `effective_io_concurrency` defaults to 16. https://www.postgresql.org/docs/18/runtime-config-resource.html

## Old patterns

| Old | Current |
| --- | --- |
| `'now'` or `'today'` literals in defaults | `now()`, `CURRENT_DATE` (https://www.postgresql.org/docs/18/datatype-datetime.html) |
| MD5 passwords, `password` auth method | `scram-sha-256` (https://www.postgresql.org/docs/18/auth-password.html) |
