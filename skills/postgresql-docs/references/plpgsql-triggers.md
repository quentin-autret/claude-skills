# Functions, PL/pgSQL, triggers, procedures

## Contents
- Volatility
- SQL-body functions
- PL/pgSQL statements
- Dynamic SQL
- Errors and exceptions
- Plan caching and naming
- Triggers
- Procedures
- Old patterns

## Volatility

- Functions default to VOLATILE. Mark each with the strictest true category: IMMUTABLE (same result forever, needed in indexes), STABLE (same result within a statement, no writes), VOLATILE (side effects or changing results). https://www.postgresql.org/docs/18/xfunc-volatility.html
- A function whose result depends on a setting such as `TimeZone` is STABLE, not IMMUTABLE. An IMMUTABLE label on it can freeze a stale value into cached plans. https://www.postgresql.org/docs/18/xfunc-volatility.html
- Reading tables in an IMMUTABLE function is unwise and not enforced. https://www.postgresql.org/docs/18/xfunc-volatility.html
- STABLE and IMMUTABLE functions use the calling query's snapshot and do not see its changes. Only VOLATILE ones see them, which also applies to trigger functions. https://www.postgresql.org/docs/18/xfunc-volatility.html https://www.postgresql.org/docs/18/trigger-datachanges.html
- PARALLEL defaults to UNSAFE. Mislabeling SAFE can give wrong answers. https://www.postgresql.org/docs/18/sql-createfunction.html

## SQL-body functions

- Prefer a `BEGIN ATOMIC` or `RETURN` body for SQL functions: parsed at creation, dependencies tracked. A string body is parsed at execution time. https://www.postgresql.org/docs/18/sql-createfunction.html
  ```sql
  CREATE FUNCTION add(a integer, b integer) RETURNS integer
      LANGUAGE SQL
      IMMUTABLE
      RETURNS NULL ON NULL INPUT
      RETURN a + b;
  ```
- `CREATE OR REPLACE FUNCTION` keeps owner, grants and dependents. It cannot change argument or return types. https://www.postgresql.org/docs/18/sql-createfunction.html
- Prefer `anycompatible` to `anyelement`: mixed arguments are promoted to a common type. https://www.postgresql.org/docs/18/plpgsql-declarations.html

## PL/pgSQL statements

- Declare parameters by name in CREATE FUNCTION, not with `ALIAS FOR $1`. Type variables with `%TYPE` and `%ROWTYPE`. https://www.postgresql.org/docs/18/plpgsql-declarations.html
- A DECLARE default is evaluated each time the block is entered. https://www.postgresql.org/docs/18/plpgsql-declarations.html
- `SELECT ... INTO v` takes the first row, or NULLs if none: check `FOUND`. `INTO STRICT` requires exactly one row and raises NO_DATA_FOUND or TOO_MANY_ROWS. https://www.postgresql.org/docs/18/plpgsql-statements.html
- `INSERT/UPDATE/DELETE ... RETURNING ... INTO` errors on more than one row even without STRICT. https://www.postgresql.org/docs/18/plpgsql-statements.html
- Discard a result with `PERFORM`. Row count: `GET DIAGNOSTICS n = ROW_COUNT`. https://www.postgresql.org/docs/18/plpgsql-statements.html
- CASE with no matching WHEN and no ELSE raises CASE_NOT_FOUND. An IF with a NULL condition takes ELSE. https://www.postgresql.org/docs/18/plpgsql-control-structures.html
- `RETURN NEXT` and `RETURN QUERY` buffer the whole set before returning: avoid very large results. https://www.postgresql.org/docs/18/plpgsql-control-structures.html
- In development, `SET plpgsql.extra_warnings = 'all'` reports shadowed variables, multi-row INTO and mismatched INTO targets. https://www.postgresql.org/docs/18/plpgsql-development-tips.html

## Dynamic SQL

- Use `EXECUTE format(...) USING ...`: `%I` for identifiers, `$n` with USING for values. Never concatenate values. https://www.postgresql.org/docs/18/plpgsql-statements.html
  ```sql
  EXECUTE format('UPDATE tbl SET %I = $1 WHERE key = $2', colname)
     USING newvalue, keyvalue;
  ```
- Building a literal by hand: `quote_literal(NULL)` is NULL and nulls the whole string, so use `quote_nullable` or `%L`. https://www.postgresql.org/docs/18/plpgsql-statements.html
- Utility commands (DDL) cannot take PL/pgSQL variables: build them with EXECUTE. EXECUTE replans every run and leaves FOUND unchanged. https://www.postgresql.org/docs/18/plpgsql-statements.html

## Errors and exceptions

- An `EXCEPTION` block rolls back the block's changes and is significantly more expensive to enter. Use it only where needed. https://www.postgresql.org/docs/18/plpgsql-control-structures.html
- `WHEN OTHERS` misses QUERY_CANCELED and ASSERT_FAILURE. Read details with `GET STACKED DIAGNOSTICS`. https://www.postgresql.org/docs/18/plpgsql-control-structures.html
- Raise with a hint or a standard code: https://www.postgresql.org/docs/18/plpgsql-errors-and-messages.html
  ```sql
  RAISE EXCEPTION 'Nonexistent ID --> %', user_id
        USING HINT = 'Please check your user ID';
  RAISE 'Duplicate user ID: %', user_id USING ERRCODE = 'unique_violation';
  ```
- Custom SQLSTATEs: 5 digits or upper-case letters, other than `00000`; avoid codes ending in `000`. `ASSERT` is for bugs, not user errors. https://www.postgresql.org/docs/18/plpgsql-errors-and-messages.html

## Plan caching and naming

- A name that is both a variable and a column raises an error. Prefix variables (`v_`), qualify columns, or start the function with `#variable_conflict use_column`. https://www.postgresql.org/docs/18/plpgsql-implementation.html
- Statements are prepared on first run per session and may switch to a generic plan. Errors beyond syntax appear only when that path runs. https://www.postgresql.org/docs/18/plpgsql-implementation.html
- A `'now'` literal in a statement is frozen for the session: use `now()`. https://www.postgresql.org/docs/18/plpgsql-implementation.html

## Triggers

- Prefer BEFORE row triggers to check or modify the row. Use AFTER row triggers to touch other tables, since only they see the final row. https://www.postgresql.org/docs/18/trigger-definition.html
- A BEFORE row trigger returns NEW (or OLD for DELETE) to proceed. Returning NULL skips the operation for that row. https://www.postgresql.org/docs/18/trigger-definition.html
- Triggers on one event fire in alphabetical order of name. https://www.postgresql.org/docs/18/trigger-definition.html
- `UPDATE OF col` fires whenever col is in SET, changed or not. Fire on real changes with WHEN, which also saves work in AFTER triggers: https://www.postgresql.org/docs/18/sql-createtrigger.html
  ```sql
  CREATE TRIGGER check_update
      BEFORE UPDATE ON accounts
      FOR EACH ROW
      WHEN (OLD.balance IS DISTINCT FROM NEW.balance)
      EXECUTE FUNCTION check_account_update();
  ```
- For statements touching many rows, a statement trigger with transition tables can be much faster than a row trigger. https://www.postgresql.org/docs/18/plpgsql-trigger.html
  ```sql
  CREATE TRIGGER emp_audit_upd
      AFTER UPDATE ON emp
      REFERENCING OLD TABLE AS old_table NEW TABLE AS new_table
      FOR EACH STATEMENT EXECUTE FUNCTION process_emp_audit();
  ```
- `INSERT ... ON CONFLICT DO UPDATE` can fire BEFORE INSERT then BEFORE UPDATE on one row. https://www.postgresql.org/docs/18/trigger-definition.html
- Row triggers created on a partitioned table are cloned to every partition. https://www.postgresql.org/docs/18/sql-createtrigger.html
- A partition-moving UPDATE fires AFTER DELETE and AFTER INSERT row triggers, not AFTER UPDATE. https://www.postgresql.org/docs/18/trigger-definition.html
- Foreign key actions fire the referencing table's triggers: a trigger blocking them breaks integrity. Recursion has no depth limit. https://www.postgresql.org/docs/18/trigger-definition.html
- PostgreSQL 18 runs AFTER triggers as the role active when the event was queued. https://www.postgresql.org/docs/18/release-18.html
- Use `TG_TABLE_NAME` and `TG_TABLE_SCHEMA` in trigger functions. https://www.postgresql.org/docs/18/plpgsql-trigger.html

## Procedures

- Procedures run with `CALL` and may COMMIT, except when SECURITY DEFINER or with a `SET` clause. https://www.postgresql.org/docs/18/sql-createprocedure.html
- From PL/pgSQL, pass a variable for each OUT or INOUT argument. https://www.postgresql.org/docs/18/plpgsql-control-structures.html

## Old patterns

- `EXECUTE PROCEDURE` in CREATE TRIGGER -> `EXECUTE FUNCTION`. https://www.postgresql.org/docs/18/sql-createtrigger.html
- `TG_RELNAME` -> `TG_TABLE_NAME`. https://www.postgresql.org/docs/18/plpgsql-trigger.html
- Single-quoted bodies with doubled quotes -> dollar quoting. https://www.postgresql.org/docs/18/plpgsql-development-tips.html
- Update-or-insert loop with `EXCEPTION WHEN unique_violation` -> `INSERT ... ON CONFLICT`. https://www.postgresql.org/docs/18/plpgsql-control-structures.html
