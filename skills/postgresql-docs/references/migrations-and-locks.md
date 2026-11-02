# Migrations: ALTER TABLE, index builds, lock levels

## Contents
- Lock levels that matter for DDL
- Rewrite or not
- Adding constraints without long locks
- Building and rebuilding indexes
- Partition maintenance
- Old patterns

## Lock levels that matter for DDL

- Only ACCESS EXCLUSIVE blocks plain SELECT. It is taken by DROP TABLE, TRUNCATE, REINDEX (on the index), CLUSTER, VACUUM FULL, non-concurrent REFRESH MATERIALIZED VIEW, and many forms of ALTER TABLE and ALTER INDEX. https://www.postgresql.org/docs/18/explicit-locking.html
- ALTER TABLE defaults to ACCESS EXCLUSIVE. With several subcommands the strictest lock applies. Weaker ones: `ADD FOREIGN KEY` takes SHARE ROW EXCLUSIVE on both tables, `VALIDATE CONSTRAINT`, `SET STATISTICS` and autovacuum storage options take SHARE UPDATE EXCLUSIVE. https://www.postgresql.org/docs/18/sql-altertable.html
- `CREATE INDEX` takes SHARE: reads continue, writes wait for the whole build. `CREATE INDEX CONCURRENTLY` takes SHARE UPDATE EXCLUSIVE. https://www.postgresql.org/docs/18/explicit-locking.html
- Set `lock_timeout` in the migration session (setting it in postgresql.conf is not recommended, it would affect all sessions). The limit applies separately to each lock acquisition attempt, and a statement that waits longer is aborted (SQLSTATE 55P03). https://www.postgresql.org/docs/18/runtime-config-client.html https://www.postgresql.org/docs/18/errcodes-appendix.html
  ```sql
  SET lock_timeout = '5s';
  ```
- A long-running index build, like any long transaction, holds back VACUUM on other tables. https://www.postgresql.org/docs/18/sql-createindex.html

## Rewrite or not

- No rewrite, near instant on any size: ADD COLUMN with no default or a non-volatile default, ADD COLUMN of a virtual generated column, type changes where `USING` does not change the contents and the old type is binary coercible (`varchar` to `text`) or an unconstrained domain over the new type (indexes are still rebuilt unless the new index is logically equivalent). https://www.postgresql.org/docs/18/sql-altertable.html https://www.postgresql.org/docs/18/ddl-alter.html
- Full rewrite of table and indexes: ADD COLUMN with a volatile default (`clock_timestamp()`), a stored generated column, an identity column, or a constrained domain, and most type changes (up to double disk space). https://www.postgresql.org/docs/18/sql-altertable.html
- Volatile default without a rewrite under lock: add the column without default, fill the values with UPDATE, then `SET DEFAULT`. https://www.postgresql.org/docs/18/ddl-alter.html
- Rewriting forms and TRUNCATE are not MVCC-safe: a concurrent transaction with an older snapshot sees the table as empty. https://www.postgresql.org/docs/18/mvcc-caveats.html
- A type change drops the column's statistics: run ANALYZE after it. https://www.postgresql.org/docs/18/sql-altertable.html
- `USING` is not applied to the column default. Drop the default, change the type, set a new default in one statement: https://www.postgresql.org/docs/18/sql-altertable.html
  ```sql
  ALTER TABLE foo
      ALTER COLUMN foo_timestamp DROP DEFAULT,
      ALTER COLUMN foo_timestamp TYPE timestamp with time zone
      USING timestamp with time zone 'epoch' + foo_timestamp * interval '1 second',
      ALTER COLUMN foo_timestamp SET DEFAULT now();
  ```
- Combine subcommands in one ALTER TABLE to pay for one table pass. https://www.postgresql.org/docs/18/sql-altertable.html
- DROP COLUMN is fast but frees no space until rows are rewritten. https://www.postgresql.org/docs/18/sql-altertable.html

## Adding constraints without long locks

- Adding CHECK, NOT NULL or FOREIGN KEY scans the table while blocking writes. Add it `NOT VALID` (no scan, enforced for new rows), then `VALIDATE CONSTRAINT`, which takes only SHARE UPDATE EXCLUSIVE. PostgreSQL 18 accepts NOT VALID on NOT NULL too. https://www.postgresql.org/docs/18/sql-altertable.html
  ```sql
  ALTER TABLE distributors ADD CONSTRAINT distfk FOREIGN KEY (address)
      REFERENCES addresses (address) NOT VALID;
  ALTER TABLE distributors VALIDATE CONSTRAINT distfk;
  ```
- `SET NOT NULL` skips its scan when a valid CHECK already proves no NULLs. https://www.postgresql.org/docs/18/sql-altertable.html
- UNIQUE or PRIMARY KEY: build the index concurrently, then attach it. The index must be a plain B-tree (no expression, no predicate). Not supported on partitioned tables. https://www.postgresql.org/docs/18/sql-altertable.html
  ```sql
  CREATE UNIQUE INDEX CONCURRENTLY dist_id_temp_idx ON distributors (dist_id);
  ALTER TABLE distributors DROP CONSTRAINT distributors_pkey,
      ADD CONSTRAINT distributors_pkey PRIMARY KEY USING INDEX dist_id_temp_idx;
  ```

## Building and rebuilding indexes

- On live tables use `CREATE INDEX CONCURRENTLY`. It cannot run in a transaction block, scans twice, waits for older transactions, and allows one concurrent build per table. https://www.postgresql.org/docs/18/sql-createindex.html
- A failed concurrent build leaves an INVALID index that still costs writes. DROP it and retry, or `REINDEX INDEX CONCURRENTLY`. A unique one keeps enforcing uniqueness while invalid. https://www.postgresql.org/docs/18/sql-createindex.html
- While a unique index is built or rebuilt concurrently, `INSERT ... ON CONFLICT` on the table may fail with a unique violation. https://www.postgresql.org/docs/18/sql-insert.html
- Plain REINDEX blocks writes and takes ACCESS EXCLUSIVE on the index, which in practice blocks most queries on the table. Use `REINDEX ... CONCURRENTLY`. After a failure, drop the leftover `_ccnew` index and retry. A leftover `_ccold` index just needs DROP. https://www.postgresql.org/docs/18/sql-reindex.html
- Exclusion-constraint indexes cannot be reindexed concurrently. https://www.postgresql.org/docs/18/sql-reindex.html

## Partition maintenance

- `CREATE TABLE ... PARTITION OF` and dropping a partition take ACCESS EXCLUSIVE on the parent. `ATTACH PARTITION` takes only SHARE UPDATE EXCLUSIVE on the parent. https://www.postgresql.org/docs/18/ddl-partitioning.html
- Attach without a long validation scan: load a standalone table, add a CHECK matching the bound, attach, drop the CHECK. With a DEFAULT partition, also add a CHECK on it excluding the new range, or it is scanned under ACCESS EXCLUSIVE. https://www.postgresql.org/docs/18/ddl-partitioning.html
  ```sql
  CREATE TABLE measurement_y2008m02
    (LIKE measurement INCLUDING DEFAULTS INCLUDING CONSTRAINTS);
  ALTER TABLE measurement_y2008m02 ADD CONSTRAINT y2008m02
     CHECK ( logdate >= DATE '2008-02-01' AND logdate < DATE '2008-03-01' );
  ALTER TABLE measurement ATTACH PARTITION measurement_y2008m02
      FOR VALUES FROM ('2008-02-01') TO ('2008-03-01' );
  ```
- `DETACH PARTITION ... CONCURRENTLY` takes SHARE UPDATE EXCLUSIVE, cannot run in a transaction block or with a DEFAULT partition. `FINALIZE` completes an interrupted one. https://www.postgresql.org/docs/18/sql-altertable.html
- Index a partitioned table without blocking writes: create it on the parent `ON ONLY` (invalid), build each partition's index concurrently, attach each. The parent index turns valid once all are attached. https://www.postgresql.org/docs/18/ddl-partitioning.html
  ```sql
  CREATE INDEX measurement_usls_idx ON ONLY measurement (unitsales);
  CREATE INDEX CONCURRENTLY measurement_usls_200602_idx
      ON measurement_y2006m02 (unitsales);
  ALTER INDEX measurement_usls_idx ATTACH PARTITION measurement_usls_200602_idx;
  ```

## Old patterns

- Plain `CREATE INDEX` or `REINDEX` on a live table -> the `CONCURRENTLY` forms. https://www.postgresql.org/docs/18/sql-reindex.html
- `ADD CONSTRAINT` with an immediate scan on a big table -> `NOT VALID` then `VALIDATE CONSTRAINT`. https://www.postgresql.org/docs/18/sql-altertable.html
