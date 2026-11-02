# Indexes

## Contents
- Choosing an index type
- Multicolumn indexes and skip scan
- Sort order and combining indexes
- Expression and partial indexes
- Index-only scans and INCLUDE
- Operator classes and collations
- GIN and BRIN
- Checking index use
- Old patterns

## Choosing an index type

- B-tree is the default and the only type that sorts and enforces uniqueness. It serves `< <= = >= >`, BETWEEN, IN, IS NULL and anchored prefixes (`LIKE 'foo%'`, `~ '^foo'`), never `LIKE '%bar'`. https://www.postgresql.org/docs/18/indexes-types.html https://www.postgresql.org/docs/18/indexes-unique.html
- Hash serves `=` only. GiST and SP-GiST serve geometry, ranges and nearest neighbour (`ORDER BY location <-> point '(101,456)' LIMIT 10`). GIN serves multi-valued data (arrays, jsonb, tsvector). BRIN serves huge tables whose column follows the physical row order. https://www.postgresql.org/docs/18/indexes-types.html https://www.postgresql.org/docs/18/rangetypes.html
- B-tree, GiST and SP-GiST use short page-level locks, which give the highest concurrency without deadlock conditions. https://www.postgresql.org/docs/18/locking-indexes.html
- Every index slows writes and can prevent HOT updates. Drop indexes that are never used. https://www.postgresql.org/docs/18/indexes-intro.html
- PRIMARY KEY and UNIQUE already create a unique B-tree. Do not add a second index on the same columns. https://www.postgresql.org/docs/18/indexes-unique.html

## Multicolumn indexes and skip scan

- Put the columns tested with equality first. Equality on leading columns plus one range on the next column bound the scan. Conditions on later columns only filter inside the index. https://www.postgresql.org/docs/18/indexes-multicolumn.html
- PostgreSQL 18 B-tree skip scan: an index on `(x, y)` can serve `WHERE y = 7700` by searching each distinct `x`. It pays off when the leading column has few distinct values (up to several hundred). https://www.postgresql.org/docs/18/indexes-multicolumn.html https://www.postgresql.org/docs/18/indexes-bitmap-scans.html
- Indexes with more than 3 columns rarely help. A single-column index is usually enough. https://www.postgresql.org/docs/18/indexes-multicolumn.html
- Multicolumn GiST depends mostly on its first column. Multicolumn GIN and BRIN work the same whichever columns the query uses. https://www.postgresql.org/docs/18/indexes-multicolumn.html

## Sort order and combining indexes

- An index on `(x, y)` serves `ORDER BY x, y` and `ORDER BY x DESC, y DESC`, not `ORDER BY x, y DESC`: that needs `(x ASC, y DESC)`. https://www.postgresql.org/docs/18/indexes-ordering.html
- Index order pays off for `ORDER BY ... LIMIT n`. For a large fraction of the table, scan and sort is faster. https://www.postgresql.org/docs/18/indexes-ordering.html
- `WHERE a = 5 OR b = 6` cannot use one `(a, b)` index. The planner may combine several indexes through bitmaps, which loses index order. https://www.postgresql.org/docs/18/indexes-bitmap-scans.html
- Mixed queries on x, y and both: two single-column indexes, or `(x, y)` plus an index on `y`. https://www.postgresql.org/docs/18/indexes-bitmap-scans.html

## Expression and partial indexes

- Index an expression to search on it, and wrap it in parentheses unless it is a single function call. Every function in it must be IMMUTABLE. https://www.postgresql.org/docs/18/indexes-expressional.html https://www.postgresql.org/docs/18/sql-createindex.html
  ```sql
  CREATE INDEX test1_lower_col1_idx ON test1 (lower(col1));
  CREATE INDEX people_names ON people ((first_name || ' ' || last_name));
  ```
- A UNIQUE index on `lower(col)` enforces case-insensitive uniqueness. https://www.postgresql.org/docs/18/indexes-expressional.html
- A new expression index has no statistics until ANALYZE runs. https://www.postgresql.org/docs/18/sql-createindex.html
- A partial unique index enforces uniqueness over a subset: https://www.postgresql.org/docs/18/indexes-partial.html
  ```sql
  CREATE UNIQUE INDEX tests_success_constraint ON tests (subject, target)
      WHERE success;
  ```
- A partial index is used only when the planner can prove the query's WHERE implies the predicate: write the same condition. A parameter (`x < $1`) never matches a predicate. https://www.postgresql.org/docs/18/indexes-partial.html
- Do not replace partitioning with many partial indexes (`WHERE category = 1`, `= 2`, ...). Use one index on `(category, data)`. https://www.postgresql.org/docs/18/indexes-partial.html

## Index-only scans and INCLUDE

- Add payload columns with `INCLUDE` so a query can skip the heap. Uniqueness applies to key columns only. https://www.postgresql.org/docs/18/indexes-index-only-scans.html
  ```sql
  CREATE UNIQUE INDEX title_idx ON films (title) INCLUDE (director, rating);
  ```
- Index-only scans skip the heap only for pages marked all-visible by VACUUM. Frequently updated tables gain little. https://www.postgresql.org/docs/18/indexes-index-only-scans.html
- Keep INCLUDE columns few and narrow: a too-large index tuple makes INSERT fail. B-tree deduplication is disabled on such indexes. https://www.postgresql.org/docs/18/indexes-index-only-scans.html https://www.postgresql.org/docs/18/sql-createindex.html
- For an index-only scan on `f(x)`, add `x`: `CREATE INDEX tab_f_x ON tab (f(x)) INCLUDE (x);` https://www.postgresql.org/docs/18/indexes-index-only-scans.html
- GIN never supports index-only scans. https://www.postgresql.org/docs/18/indexes-index-only-scans.html

## Operator classes and collations

- Outside the C locale a default B-tree on text cannot serve LIKE or regex prefixes. Add an index with `text_pattern_ops` (or `varchar_pattern_ops`). It cannot serve `<` or `>`, so keep a default index too if needed. https://www.postgresql.org/docs/18/indexes-opclass.html
  ```sql
  CREATE INDEX test_index ON test_table (col varchar_pattern_ops);
  ```
- An index serves one collation per column. A query with another `COLLATE` needs its own index. https://www.postgresql.org/docs/18/indexes-collations.html

## GIN and BRIN

- GIN for jsonb: `jsonb_path_ops` supports fewer operators (`@>`, `@?`, `@@`, not the key-exists operators) but performs better than the default `jsonb_ops`. https://www.postgresql.org/docs/18/gin.html https://www.postgresql.org/docs/18/datatype-json.html
- GIN updates are slow, so `fastupdate` (on) queues them in a pending list that searches must also scan. Set `fastupdate = off` when steady response time matters more than insert speed. https://www.postgresql.org/docs/18/gin.html
- For bulk insertions, drop the GIN index and recreate it after, with a generous `maintenance_work_mem` (build time is very sensitive to it). https://www.postgresql.org/docs/18/gin.html
- BRIN is lossy and tiny. New page ranges stay unsummarized (never skipped) until VACUUM runs, unless `autosummarize = on` (off by default). https://www.postgresql.org/docs/18/brin.html
- BRIN with outliers: `minmax_multi` operator classes. Equality only: `bloom` classes. https://www.postgresql.org/docs/18/brin.html

## Checking index use

- Run ANALYZE first, test with realistic data volumes: on very small tables an index will hardly be chosen over a sequential scan. https://www.postgresql.org/docs/18/indexes-examine.html
- Find unused indexes with `pg_stat_user_indexes.idx_scan` and `last_idx_scan`. https://www.postgresql.org/docs/18/monitoring-stats.html
- To test whether an index is usable at all, `SET enable_seqscan = off` in a session and read the plan. https://www.postgresql.org/docs/18/indexes-examine.html
- B-tree `fillfactor` defaults to 90. Use 100 only for static tables. https://www.postgresql.org/docs/18/sql-createindex.html
- Parallel builds exist for B-tree, GIN (PostgreSQL 18) and BRIN. `maintenance_work_mem` is the maximum memory for each index build as a whole, however many workers run. https://www.postgresql.org/docs/18/sql-createindex.html https://www.postgresql.org/docs/18/release-18.html

## Old patterns

- `USING rtree` -> `USING gist`. https://www.postgresql.org/docs/18/sql-createindex.html
- Payload as extra key columns `(x, y)` -> `(x) INCLUDE (y)`. https://www.postgresql.org/docs/18/indexes-index-only-scans.html
