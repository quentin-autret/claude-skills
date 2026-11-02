# Writes: upsert, UPDATE, DELETE, RETURNING, MERGE, COPY, bulk loads

## Contents
- INSERT ... ON CONFLICT
- UPDATE and DELETE
- Batching large changes
- RETURNING
- MERGE
- COPY
- Bulk loading
- Old patterns

## INSERT ... ON CONFLICT

- Default upsert: `ON CONFLICT (cols) DO UPDATE`. It guarantees insert or update atomically under concurrency. Refer to the proposed row as `EXCLUDED`. https://www.postgresql.org/docs/18/sql-insert.html
  ```sql
  INSERT INTO distributors (did, dname)
      VALUES (5, 'Gizmo Transglobal'), (6, 'Associated Computing, Inc')
      ON CONFLICT (did) DO UPDATE SET dname = EXCLUDED.dname;
  ```
- Name the columns (index inference) rather than `ON CONSTRAINT name`: inference keeps working when the index is rebuilt under another name. For a partial unique index, repeat its predicate: `ON CONFLICT (did) WHERE is_active`. https://www.postgresql.org/docs/18/sql-insert.html
- `DO UPDATE` requires a conflict target. `DO NOTHING` without one covers every unique constraint. https://www.postgresql.org/docs/18/sql-insert.html
- Arbiters must be non-deferrable. Exclusion constraints cannot be arbiters for `DO UPDATE`. https://www.postgresql.org/docs/18/sql-insert.html
- One statement cannot update the same row twice: two VALUES rows with the same key raise a cardinality violation. Deduplicate the input first. https://www.postgresql.org/docs/18/sql-insert.html
- `DO UPDATE ... WHERE cond` leaves non-matching rows unchanged but still locks them, and RETURNING omits them. https://www.postgresql.org/docs/18/sql-insert.html
- Do not qualify the SET target: `SET d.dname = ...` is invalid, `SET dname = ...` is right. https://www.postgresql.org/docs/18/sql-insert.html
- On a partitioned table, `DO UPDATE` cannot move the row to another partition. https://www.postgresql.org/docs/18/sql-insert.html
- `OVERRIDING SYSTEM VALUE` writes explicit ids into a `GENERATED ALWAYS` column. `OVERRIDING USER VALUE` ignores supplied ids when copying between tables. https://www.postgresql.org/docs/18/sql-insert.html

## UPDATE and DELETE

- `UPDATE ... FROM` must join at most one source row per target row, otherwise one arbitrary row is used. A `SET (a, b) = (SELECT ...)` subquery errors on duplicates and sets NULL when nothing matches. https://www.postgresql.org/docs/18/sql-update.html
- Do not repeat the target table in FROM or USING unless you want a self-join. Other databases read that differently. https://www.postgresql.org/docs/18/sql-update.html https://www.postgresql.org/docs/18/sql-delete.html
- The command tag `UPDATE n` counts matched rows, changed or not. https://www.postgresql.org/docs/18/sql-update.html
- Updating a partition key moves the row. A concurrent change to the moving row raises SQLSTATE 40001: retry the transaction. https://www.postgresql.org/docs/18/sql-update.html
- To remove all rows, TRUNCATE is faster than DELETE. https://www.postgresql.org/docs/18/sql-delete.html

## Batching large changes

- Large UPDATEs and DELETEs cause bloat, replica lag and lock contention. There is no LIMIT on UPDATE or DELETE: select a batch of `ctid` in a CTE and rerun until no row is left. ORDER BY in the CTE avoids deadlocks with other writers using the same order. https://www.postgresql.org/docs/18/sql-update.html https://www.postgresql.org/docs/18/sql-delete.html
  ```sql
  WITH delete_batch AS (
    SELECT l.ctid FROM user_logs AS l
      WHERE l.status = 'archived'
      ORDER BY l.creation_date
      FOR UPDATE
      LIMIT 10000
  )
  DELETE FROM user_logs AS dl
    USING delete_batch AS del
    WHERE dl.ctid = del.ctid;
  ```
- With `SKIP LOCKED` in the batch query, finish with one run without it to catch skipped rows. https://www.postgresql.org/docs/18/sql-update.html

## RETURNING

- Use RETURNING to get generated ids and trigger-modified values without a second query. https://www.postgresql.org/docs/18/dml-returning.html
- PostgreSQL 18: `old` and `new` give the values before and after, in INSERT, UPDATE, DELETE and MERGE. `old` is non-NULL for an `ON CONFLICT DO UPDATE` that hit a conflict. https://www.postgresql.org/docs/18/dml-returning.html https://www.postgresql.org/docs/18/release-18.html
  ```sql
  UPDATE products SET price = price * 1.10
    WHERE price <= 99.99
    RETURNING name, old.price AS old_price, new.price AS new_price;
  ```
- `RETURNING WITH (OLD AS o, NEW AS n)` renames them when a table is called `old` or `new`. https://www.postgresql.org/docs/18/dml-returning.html

## MERGE

- MERGE runs, per source row, the first WHEN clause whose condition holds. Join at most one source row per target row: a second attempt to change the same row raises an error (a cardinality violation for UPDATE or DELETE, a uniqueness violation for INSERT). https://www.postgresql.org/docs/18/sql-merge.html
- Put only target-matching columns in `ON`. Filter with `WHEN ... AND cond`, or the chosen action changes in surprising ways. https://www.postgresql.org/docs/18/sql-merge.html
- MERGE is not a concurrency-safe upsert: under READ COMMITTED a concurrent insert makes its INSERT fail with a unique violation. Use `INSERT ... ON CONFLICT` for insert-or-update. https://www.postgresql.org/docs/18/transaction-iso.html
- `WHEN NOT MATCHED BY SOURCE THEN DELETE` syncs a table to a source. `merge_action()` in RETURNING says which action ran. https://www.postgresql.org/docs/18/sql-merge.html
  ```sql
  MERGE INTO wines w
  USING new_wine_list s
  ON s.winename = w.winename
  WHEN NOT MATCHED BY TARGET THEN
    INSERT VALUES(s.winename, s.stock)
  WHEN MATCHED AND w.stock != s.stock THEN
    UPDATE SET stock = s.stock
  WHEN NOT MATCHED BY SOURCE THEN
    DELETE
  RETURNING merge_action(), w.winename;
  ```
- Order the source query to avoid deadlocks between concurrent MERGEs. https://www.postgresql.org/docs/18/sql-merge.html

## COPY

- `COPY ... FROM 'file'` and `PROGRAM` act on the server machine and need superuser or `pg_read_server_files`, `pg_write_server_files`, `pg_execute_server_program`. For client files use `COPY ... FROM STDIN` through the driver or psql `\copy`. https://www.postgresql.org/docs/18/sql-copy.html
- Never put untrusted input in a `PROGRAM` command: it runs through the shell. https://www.postgresql.org/docs/18/sql-copy.html
- `ON_ERROR ignore` skips rows with conversion errors. PostgreSQL 18 adds `REJECT_LIMIT n` to fail beyond n bad rows. https://www.postgresql.org/docs/18/sql-copy.html https://www.postgresql.org/docs/18/release-18.html
- `COPY FROM` is refused on tables with row-level security enabled: use INSERT. https://www.postgresql.org/docs/18/sql-copy.html
- A failed `COPY FROM` leaves dead rows: run VACUUM. https://www.postgresql.org/docs/18/sql-copy.html
- In CSV, NULL is an unquoted empty field and an empty string is `""`. https://www.postgresql.org/docs/18/sql-copy.html

## Bulk loading

- Load with COPY (one command, so no autocommit concern), then create indexes and foreign keys, then ANALYZE. Prepared INSERTs are the fallback when COPY is impossible. https://www.postgresql.org/docs/18/populate.html
- A load into a table with foreign keys queues one trigger event per row, which can exhaust memory over millions of rows. Drop and re-add the foreign key, or load in smaller transactions. https://www.postgresql.org/docs/18/populate.html
- Raise `maintenance_work_mem` (index and FK builds) and `max_wal_size` (fewer checkpoints) temporarily. https://www.postgresql.org/docs/18/populate.html

## Old patterns

- Update-then-insert loop catching `unique_violation` -> `INSERT ... ON CONFLICT DO UPDATE`. https://www.postgresql.org/docs/18/plpgsql-control-structures.html
- CSV files ending with `\.` as end-of-data -> PostgreSQL 18 no longer treats it as end-of-data when reading CSV from a separate file (psql still ends inline COPY data at it). https://www.postgresql.org/docs/18/sql-copy.html
- `COPY t FROM f CSV HEADER` -> `COPY t FROM f WITH (FORMAT csv, HEADER)`. https://www.postgresql.org/docs/18/sql-copy.html
