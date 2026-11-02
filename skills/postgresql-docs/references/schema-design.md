# Schema design

## Contents
- Keys: identity columns and sequences
- Generated columns
- Constraints
- Domains
- Schemas and search_path
- Dependencies
- Partitioning design
- Table options
- Old patterns

## Keys: identity columns and sequences

- Use `GENERATED ALWAYS AS IDENTITY` when only the database assigns keys: an explicit value then needs `OVERRIDING SYSTEM VALUE`. `BY DEFAULT` lets a supplied value win. https://www.postgresql.org/docs/18/ddl-identity-columns.html
- An identity column is NOT NULL but not unique: sequences can be reset and values inserted by hand. Add PRIMARY KEY or UNIQUE yourself. https://www.postgresql.org/docs/18/ddl-identity-columns.html
- UPDATE of a `GENERATED ALWAYS` identity column to anything but DEFAULT is rejected. There is no OVERRIDING clause for UPDATE. https://www.postgresql.org/docs/18/sql-createtable.html
- `COPY` always writes the supplied identity values, ALWAYS or not, so rows loaded with explicit ids leave the sequence behind. Resync it in the same transaction: https://www.postgresql.org/docs/18/sql-createtable.html https://www.postgresql.org/docs/18/sql-createsequence.html
  ```sql
  BEGIN;
  COPY distributors FROM 'input_file';
  SELECT setval('serial', max(id)) FROM distributors;
  END;
  ```
- Sequences always leave gaps: `nextval` and `setval` are never rolled back, and `INSERT ... ON CONFLICT` consumes a value before the conflict is detected. Gapless numbering needs a counter table under exclusive lock, far more expensive. https://www.postgresql.org/docs/18/functions-sequence.html https://www.postgresql.org/docs/18/sql-createsequence.html
- A sequence value used outside the database (sent to a client, printed on an invoice) is only durable once the transaction that called `nextval` has committed. A crash can revert it. https://www.postgresql.org/docs/18/functions-sequence.html

## Generated columns

- PostgreSQL 18: a generated column without a keyword is VIRTUAL (computed on read, no storage). Write `STORED` to compute on write. https://www.postgresql.org/docs/18/ddl-generated-columns.html
  ```sql
  height_in numeric GENERATED ALWAYS AS (height_cm / 2.54) STORED
  ```
- The expression may use only immutable functions and the current row: no subquery, no other generated column. VIRTUAL also allows only built-in functions and types (no user-defined type). https://www.postgresql.org/docs/18/ddl-generated-columns.html
- A generated column cannot have a default or identity, cannot be in a partition key, and cannot be written (DEFAULT is allowed). https://www.postgresql.org/docs/18/ddl-generated-columns.html
- In a BEFORE trigger the NEW row does not yet hold the stored generated value and should not be accessed. Virtual columns are never computed when triggers fire. https://www.postgresql.org/docs/18/trigger-definition.html
- Virtual columns have no statistics and are never replicated logically. Stored ones are published only when the publication sets `publish_generated_columns` or lists them in its column list (PostgreSQL 18). Adding a stored generated column rewrites the table, adding a virtual one never does. https://www.postgresql.org/docs/18/sql-altertable.html https://www.postgresql.org/docs/18/ddl-generated-columns.html

## Constraints

- Name every constraint (`CONSTRAINT positive_price CHECK (price > 0)`): names clarify error messages and let you refer to the constraint when changing it. https://www.postgresql.org/docs/18/ddl-constraints.html
- CHECK passes when the expression is NULL. Add NOT NULL, which is also cheaper than `CHECK (col IS NOT NULL)`. https://www.postgresql.org/docs/18/ddl-constraints.html
- A CHECK must not read other rows or tables: it cannot guarantee the invariant and breaks dump/restore. Use UNIQUE, EXCLUDE, FOREIGN KEY, or a trigger. https://www.postgresql.org/docs/18/ddl-constraints.html
- Changing a function used in a CHECK does not recheck existing rows. Drop the constraint, change the function, re-add it. https://www.postgresql.org/docs/18/ddl-constraints.html
- UNIQUE treats NULLs as distinct, so duplicates containing NULL pass. Write `UNIQUE NULLS NOT DISTINCT` to reject them. Uniqueness over a subset of rows is a unique partial index. https://www.postgresql.org/docs/18/ddl-constraints.html
- A foreign key does not index its referencing columns. Create that index, or every DELETE or key UPDATE on the referenced table scans the referencing one. https://www.postgresql.org/docs/18/ddl-constraints.html
- Default `MATCH SIMPLE`: a row with any NULL referencing column escapes the check. Declare the columns NOT NULL or use `MATCH FULL`. https://www.postgresql.org/docs/18/sql-createtable.html
- ON DELETE: CASCADE for components (order lines), NO ACTION (default, deferrable) for independent rows, SET NULL for optional links. A column list keeps a key column intact: https://www.postgresql.org/docs/18/ddl-constraints.html
  ```sql
  FOREIGN KEY (tenant_id, author_id) REFERENCES users ON DELETE SET NULL (author_id)
  ```
- Non-deferrable UNIQUE and PRIMARY KEY are checked per row, not at statement end. `DEFERRABLE` gives standard behavior but is slower and cannot be an `ON CONFLICT` arbiter. https://www.postgresql.org/docs/18/sql-createtable.html
- Foreign key column pairs need deterministic collations or the same collation, otherwise restore and `pg_upgrade` fail in PostgreSQL 18. https://www.postgresql.org/docs/18/release-18.html
- PostgreSQL 18: CHECK and FOREIGN KEY accept `NOT ENFORCED`. Nothing is checked, yet the planner may assume the rule holds. https://www.postgresql.org/docs/18/sql-createtable.html
- PostgreSQL 18 temporal keys: `PRIMARY KEY (id, valid_at WITHOUT OVERLAPS)` and `FOREIGN KEY (id, PERIOD valid_at) REFERENCES t (id, PERIOD valid_at)`. They need a range or multirange type and GiST (`btree_gist` for scalar parts). Temporal FKs support no CASCADE, SET NULL, SET DEFAULT or RESTRICT. https://www.postgresql.org/docs/18/sql-createtable.html
- For equality-only rules UNIQUE is faster than EXCLUDE with `=`. EXCLUDE needs GiST or SP-GiST. https://www.postgresql.org/docs/18/sql-createtable.html

## Domains

- `CREATE DOMAIN posint AS integer CHECK (VALUE > 0);` Operators return the base type, so `id - 1` is an integer. Cast back (`(id - 1)::posint`) to recheck. https://www.postgresql.org/docs/18/domains.html
- Adding a column whose domain has constraints rewrites the table. https://www.postgresql.org/docs/18/sql-altertable.html

## Schemas and search_path

- Any schema on `search_path` trusts every role with CREATE on it: such a role can hijack unqualified function and operator calls. https://www.postgresql.org/docs/18/ddl-schemas.html
- Databases upgraded from PostgreSQL 14 or earlier still grant CREATE on `public` to everyone. Run `REVOKE CREATE ON SCHEMA public FROM PUBLIC;`. https://www.postgresql.org/docs/18/ddl-schemas.html
- Put extensions and shared code in their own schemas and grant USAGE explicitly. https://www.postgresql.org/docs/18/ddl-schemas.html
- `pg_catalog` is searched first unless listed. The temp schema is searched first for tables and types, never for functions. `current_schemas(true)` shows the resolved path. https://www.postgresql.org/docs/18/runtime-config-client.html

## Dependencies

- A function with a string body records no dependency on the tables it reads: dropping one breaks the function at call time. A `BEGIN ATOMIC` body is parsed at creation and its dependencies are enforced: https://www.postgresql.org/docs/18/ddl-depend.html
  ```sql
  CREATE FUNCTION get_color_note (rainbow) RETURNS text
  BEGIN ATOMIC
    SELECT note FROM my_colors WHERE color = $1;
  END;
  ```

## Partitioning design

- Rule of thumb: partitioning pays off when the table size would exceed the server's physical memory. Use declarative `PARTITION BY`. https://www.postgresql.org/docs/18/ddl-partitioning.html
- Key: the columns most WHERE clauses filter on, so that data removed together sits in one partition. Planning copes with a few thousand partitions only if queries prune to a few. https://www.postgresql.org/docs/18/ddl-partitioning.html
- PRIMARY KEY and UNIQUE must include every partition key column, with no expression, which often forces keys like `(id, logdate)`. https://www.postgresql.org/docs/18/ddl-partitioning.html
- Range bounds are lower-inclusive, upper-exclusive. A row with no matching partition errors unless a DEFAULT partition exists (none for HASH). https://www.postgresql.org/docs/18/ddl-partitioning.html
  ```sql
  CREATE TABLE measurement (city_id int not null, logdate date not null, peaktemp int)
      PARTITION BY RANGE (logdate);
  CREATE TABLE measurement_y2006m02 PARTITION OF measurement
      FOR VALUES FROM ('2006-02-01') TO ('2006-03-01');
  ```
- A regular table cannot become partitioned in place. A partition's own column default is ignored when inserting through the parent. https://www.postgresql.org/docs/18/ddl-partitioning.html https://www.postgresql.org/docs/18/sql-createtable.html
- Autovacuum never analyzes the partitioned parent: run `ANALYZE parent` after first load and after big distribution changes. https://www.postgresql.org/docs/18/routine-vacuuming.html

## Table options

- `UNLOGGED` tables are truncated after a crash and not replicated. PostgreSQL 18 refuses unlogged partitioned tables. https://www.postgresql.org/docs/18/sql-createtable.html https://www.postgresql.org/docs/18/release-18.html
- Autovacuum cannot analyze temporary tables: run ANALYZE after filling one used in complex queries. https://www.postgresql.org/docs/18/sql-createtable.html

## Old patterns

- Generated columns were always stored before PostgreSQL 18 -> write `STORED` explicitly when you need storage. https://www.postgresql.org/docs/18/release-18.html
