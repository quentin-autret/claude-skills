# Performance: EXPLAIN, statistics, vacuum, settings, monitoring

## Contents
- EXPLAIN
- Statistics
- Vacuum and autovacuum
- Planner and memory settings
- Parallel query
- pg_stat_statements and statistics views

## EXPLAIN

- `EXPLAIN ANALYZE` runs the statement. Wrap writes in a transaction you roll back: https://www.postgresql.org/docs/18/sql-explain.html
  ```sql
  BEGIN;
  EXPLAIN ANALYZE UPDATE ...;
  ROLLBACK;
  ```
- First check whether estimated rows are close to actual rows. Cost units and milliseconds are not comparable. https://www.postgresql.org/docs/18/using-explain.html
- PostgreSQL 18: ANALYZE includes BUFFERS automatically (`BUFFERS OFF` to hide). Nodes show fractional rows, `Index Searches: N` and `Disabled: true`. https://www.postgresql.org/docs/18/using-explain.html https://www.postgresql.org/docs/18/release-18.html
- With `loops` above 1, time and rows are per loop: multiply by loops. https://www.postgresql.org/docs/18/using-explain.html
- `TIMING OFF` when only row counts matter. `FORMAT JSON` for tools. `SETTINGS` lists changed planner settings. `SERIALIZE` measures output conversion. https://www.postgresql.org/docs/18/sql-explain.html
- Generic plan of a parameterized query: https://www.postgresql.org/docs/18/sql-explain.html
  ```sql
  EXPLAIN (GENERIC_PLAN)
    SELECT sum(bar) FROM test WHERE id > $1::integer AND id < $2::integer GROUP BY foo;
  ```
- Never judge plans on toy tables: a table of 1 page nearly always gets a Seq Scan. https://www.postgresql.org/docs/18/using-explain.html
- Execution time omits AFTER triggers (listed apart) and deferred constraint triggers. https://www.postgresql.org/docs/18/using-explain.html
- `Subplans Removed: N` means partitions pruned at executor start. https://www.postgresql.org/docs/18/using-explain.html

## Statistics

- Run ANALYZE after bulk loads and big changes rather than waiting for autovacuum. https://www.postgresql.org/docs/18/populate.html https://www.postgresql.org/docs/18/sql-explain.html
- Read statistics in `pg_stats`, not `pg_statistic`. https://www.postgresql.org/docs/18/planner-stats.html
- For a skewed column used in WHERE, raise its target: `ALTER TABLE t ALTER COLUMN c SET STATISTICS 500`. The default target is 100. A wrong `n_distinct` can be overridden with `SET (n_distinct = ...)`. https://www.postgresql.org/docs/18/planner-stats.html https://www.postgresql.org/docs/18/sql-analyze.html
- The planner multiplies selectivities of conditions on different columns. For correlated columns create extended statistics, then ANALYZE: https://www.postgresql.org/docs/18/planner-stats.html
  ```sql
  CREATE STATISTICS stts (dependencies) ON city, zip FROM zipcodes;
  CREATE STATISTICS stts3 (mcv) ON city, state FROM zipcodes;
  ANALYZE zipcodes;
  ```
- `dependencies` covers equality only, `ndistinct` fixes GROUP BY counts, `mcv` stores the most common value combinations of several columns. None is used for join estimates. https://www.postgresql.org/docs/18/planner-stats.html https://www.postgresql.org/docs/18/sql-createstatistics.html
- `CREATE STATISTICS s ON (expr) FROM t` gives expression statistics without an index. Without them the planner assumes 0.5% selectivity. https://www.postgresql.org/docs/18/sql-createstatistics.html
- PostgreSQL 18 `pg_upgrade` keeps planner statistics but not extended statistics: run ANALYZE after upgrade. https://www.postgresql.org/docs/18/release-18.html

## Vacuum and autovacuum

- Keep autovacuum on. Plain VACUUM runs alongside traffic and makes space reusable. Avoid VACUUM FULL: ACCESS EXCLUSIVE, full rewrite, extra disk equal to the table. https://www.postgresql.org/docs/18/routine-vacuuming.html
- Autovacuum never analyzes partitioned parents or temporary tables: run ANALYZE on them yourself. https://www.postgresql.org/docs/18/routine-vacuuming.html
- PostgreSQL 18: VACUUM and ANALYZE on a parent also process its partitions. Write `ONLY` for the parent alone. https://www.postgresql.org/docs/18/release-18.html
- A command that takes a conflicting lock (frequent ANALYZE, DDL) interrupts autovacuum, except one running to prevent transaction ID wraparound. https://www.postgresql.org/docs/18/routine-vacuuming.html
- An open transaction, even idle, stops vacuum from removing recently dead rows. https://www.postgresql.org/docs/18/runtime-config-client.html
- To wipe a whole table, TRUNCATE instead of DELETE plus VACUUM. https://www.postgresql.org/docs/18/routine-vacuuming.html
- Monitor transaction ID age: https://www.postgresql.org/docs/18/routine-vacuuming.html
  ```sql
  SELECT datname, age(datfrozenxid) FROM pg_database;
  ```
- Near wraparound: end long transactions, drop stale replication slots and prepared transactions, then run plain VACUUM, not VACUUM FULL or FREEZE. https://www.postgresql.org/docs/18/routine-vacuuming.html
- Per-table settings are storage parameters (`autovacuum_vacuum_scale_factor`...). PostgreSQL 18 caps the dead-row threshold with `autovacuum_vacuum_max_threshold`. https://www.postgresql.org/docs/18/routine-vacuuming.html
- `pg_stat_reset()` also resets the counters autovacuum relies on: run ANALYZE on the database after it. https://www.postgresql.org/docs/18/monitoring-stats.html

## Planner and memory settings

- `work_mem` (4MB) applies per sort or hash operation, and several run at once across sessions: total use can be many times it. https://www.postgresql.org/docs/18/runtime-config-resource.html
- `shared_buffers`: about 25% of RAM on a dedicated server, rarely above 40%. https://www.postgresql.org/docs/18/runtime-config-resource.html
- `maintenance_work_mem` can exceed `work_mem`. Set `autovacuum_work_mem` apart since each autovacuum worker may use it. https://www.postgresql.org/docs/18/runtime-config-resource.html
- `random_page_cost` (4.0): on a fully cached database, lower it toward `seq_page_cost`. `effective_cache_size` only informs the planner. https://www.postgresql.org/docs/18/runtime-config-query.html
- `enable_*` settings are a crude temporary workaround. Fix statistics or costs instead. https://www.postgresql.org/docs/18/runtime-config-query.html
- Many joins: with at least `geqo_threshold` (12) FROM items the planner uses genetic search. `join_collapse_limit = 1` keeps explicit JOIN order as written. https://www.postgresql.org/docs/18/explicit-joins.html https://www.postgresql.org/docs/18/runtime-config-query.html
- PostgreSQL 18 asynchronous I/O: `io_method` (`worker` default, `io_uring`, `sync`). `effective_io_concurrency` defaults to 16. https://www.postgresql.org/docs/18/runtime-config-resource.html

## Parallel query

- User-defined functions are PARALLEL UNSAFE by default and disable parallelism. Mark safe ones `PARALLEL SAFE`, never when unsure. https://www.postgresql.org/docs/18/when-can-parallel-query-be-used.html https://www.postgresql.org/docs/18/sql-createfunction.html
- No parallel plan for queries that write or lock rows, for cursors, or for PL/pgSQL `FOR ... IN query` loops. https://www.postgresql.org/docs/18/when-can-parallel-query-be-used.html

## pg_stat_statements and statistics views

- Enable with `shared_preload_libraries = 'pg_stat_statements'` (restart), `compute_query_id` set to `auto` or `on`, then `CREATE EXTENSION pg_stat_statements`. https://www.postgresql.org/docs/18/pgstatstatements.html
  ```sql
  SELECT query, calls, total_exec_time, rows, 100.0 * shared_blks_hit /
         nullif(shared_blks_hit + shared_blks_read, 0) AS hit_percent
    FROM pg_stat_statements ORDER BY total_exec_time DESC LIMIT 5;
  ```
- Identify statements by `queryid` with `dbid` and `userid`, not by text. queryid is stable across minor versions only on the same architecture with matching catalog details, and depends on internal object identifiers. https://www.postgresql.org/docs/18/pgstatstatements.html
- I/O timing columns stay zero unless `track_io_timing` is on. https://www.postgresql.org/docs/18/pgstatstatements.html
- Statistics read inside a transaction are cached until it ends. Counters flush at most once a second. https://www.postgresql.org/docs/18/monitoring-stats.html
- In `pg_stat_activity`, look for `idle in transaction` sessions. `backend_xmin` shows each session's xmin horizon. Find lock contention in `pg_locks` (ungranted rows). https://www.postgresql.org/docs/18/monitoring-stats.html https://www.postgresql.org/docs/18/monitoring-locks.html
