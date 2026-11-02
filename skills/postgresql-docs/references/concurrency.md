# Transactions, isolation, locking

## Contents
- Isolation levels
- Retrying failed transactions
- Row locks and queues
- Table locks
- Deadlocks
- Advisory locks
- Timeouts
- LISTEN and NOTIFY

## Isolation levels

- Default is READ COMMITTED. READ UNCOMMITTED behaves as READ COMMITTED. Set the level on BEGIN: it cannot change after the first query. https://www.postgresql.org/docs/18/transaction-iso.html https://www.postgresql.org/docs/18/sql-set-transaction.html
  ```sql
  BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
  ```
- READ COMMITTED takes a new snapshot per statement. An UPDATE that meets a concurrently changed row re-checks its WHERE on the new version, so one statement can act on an inconsistent view. Fine for single-row updates, wrong for complex conditions. https://www.postgresql.org/docs/18/transaction-iso.html
- REPEATABLE READ (snapshot at the first query, not at BEGIN) also prevents phantoms. Writing a row changed since the snapshot fails with `could not serialize access due to concurrent update`. Read-only transactions never fail. https://www.postgresql.org/docs/18/transaction-iso.html
- REPEATABLE READ alone does not enforce business rules across rows. Use SERIALIZABLE or explicit locks. https://www.postgresql.org/docs/18/transaction-iso.html
- SERIALIZABLE detects dangerous patterns without blocking, at the cost of more 40001 failures. If SERIALIZABLE is used for all writes and all reads needing a consistent view, no other effort is required for consistency. Results read in a serializable transaction are valid only after it commits. https://www.postgresql.org/docs/18/applevel-consistency.html https://www.postgresql.org/docs/18/transaction-iso.html
- Serializable tips: declare `READ ONLY` when possible, keep transactions short, pool connections, drop the `FOR UPDATE` it makes useless. Long reports: `SERIALIZABLE READ ONLY DEFERRABLE`, which never fails. https://www.postgresql.org/docs/18/transaction-iso.html https://www.postgresql.org/docs/18/sql-set-transaction.html
- Serializable guarantees do not reach hot standbys or logical replicas. https://www.postgresql.org/docs/18/applevel-consistency.html
- `INSERT ... ON CONFLICT DO UPDATE` always inserts or updates even in READ COMMITTED. MERGE does not: a concurrent insert raises a unique violation. https://www.postgresql.org/docs/18/transaction-iso.html

## Retrying failed transactions

- Retry the whole transaction, including the code that chose the values, on SQLSTATE 40001 (serialization_failure) and 40P01 (deadlock_detected). PostgreSQL never retries by itself. https://www.postgresql.org/docs/18/mvcc-serialization-failure-handling.html
- 23505 (unique_violation) and 23P01 (exclusion_violation) are worth retrying only when they come from a race, since they can be permanent. https://www.postgresql.org/docs/18/mvcc-serialization-failure-handling.html
- Test the SQLSTATE, never the message text. Other useful codes: 55P03 lock_not_available, 57014 query_canceled, 25P02 in_failed_sql_transaction. https://www.postgresql.org/docs/18/errcodes-appendix.html

## Row locks and queues

- Strength: `FOR UPDATE` > `FOR NO KEY UPDATE` > `FOR SHARE` > `FOR KEY SHARE`. Use `FOR NO KEY UPDATE` when key columns stay unchanged: it does not block foreign key checks (`FOR KEY SHARE`). https://www.postgresql.org/docs/18/explicit-locking.html
- Row locks never block plain reads. In REPEATABLE READ and SERIALIZABLE, a locking SELECT that meets a concurrently changed row raises a serialization failure (SQLSTATE 40001). https://www.postgresql.org/docs/18/explicit-locking.html https://www.postgresql.org/docs/18/sql-select.html
- `SELECT ... FOR UPDATE` does not stop a later change once the lock is released. To freeze a row until commit, update it. https://www.postgresql.org/docs/18/applevel-consistency.html
- Job queue with several consumers: `FOR UPDATE SKIP LOCKED` (gives an inconsistent view, fine for queues). `NOWAIT` errors instead of waiting. Both apply to row locks only. https://www.postgresql.org/docs/18/sql-select.html
- In READ COMMITTED, `ORDER BY ... FOR UPDATE` can return rows out of order: the sort happens before the lock wait. https://www.postgresql.org/docs/18/sql-select.html
- With LIMIT, rows skipped by OFFSET are locked too. Lock only some tables of a join with `FOR UPDATE OF alias`. https://www.postgresql.org/docs/18/sql-select.html

## Table locks

- PostgreSQL always takes the least restrictive lock mode possible by itself. `LOCK TABLE` without a mode takes ACCESS EXCLUSIVE. There is no UNLOCK: locks last until the transaction ends. https://www.postgresql.org/docs/18/sql-lock.html
- In REPEATABLE READ or SERIALIZABLE, run LOCK TABLE before any query, or your snapshot predates the lock. https://www.postgresql.org/docs/18/sql-lock.html
- Taking SHARE and then writing deadlocks with another session doing the same. Take SHARE ROW EXCLUSIVE instead. https://www.postgresql.org/docs/18/sql-lock.html
  ```sql
  BEGIN WORK;
  LOCK TABLE films IN SHARE ROW EXCLUSIVE MODE;
  DELETE FROM films_user_comments WHERE id IN
      (SELECT id FROM films WHERE rating < 5);
  DELETE FROM films WHERE rating < 5;
  COMMIT WORK;
  ```
- Prefer MVCC over explicit locks when it is enough: it performs better. https://www.postgresql.org/docs/18/mvcc-intro.html

## Deadlocks

- Plain UPDATEs deadlock when 2 transactions touch the same rows in opposite order. Lock objects in the same order everywhere and take the strongest lock first. https://www.postgresql.org/docs/18/explicit-locking.html
- Without a deadlock a lock wait never ends: do not keep transactions open while waiting for user input. https://www.postgresql.org/docs/18/explicit-locking.html

## Advisory locks

- Transaction-level `pg_advisory_xact_lock(key)` is released at transaction end, with no manual unlock. Session locks (`pg_advisory_lock`) survive rollback and stack: N locks need N unlocks. `pg_try_advisory_*` returns false instead of waiting. https://www.postgresql.org/docs/18/explicit-locking.html https://www.postgresql.org/docs/18/functions-admin.html
- Never call a lock function in a query with LIMIT: it may run on more rows than returned. Lock in an outer query: https://www.postgresql.org/docs/18/explicit-locking.html
  ```sql
  SELECT pg_advisory_lock(q.id) FROM
  (
    SELECT id FROM foo WHERE id > 12345 LIMIT 100
  ) q;
  ```
- Advisory locks share the lock table sized by `max_locks_per_transaction` and `max_connections`. https://www.postgresql.org/docs/18/explicit-locking.html

## Timeouts

- Setting `statement_timeout`, `lock_timeout` or `transaction_timeout` in postgresql.conf is not recommended, because it affects all sessions. A value of zero (the default) disables each. https://www.postgresql.org/docs/18/runtime-config-client.html
- `transaction_timeout` terminates any session whose transaction (explicit or implicit) runs longer than the limit. https://www.postgresql.org/docs/18/runtime-config-client.html
- Set `idle_in_transaction_session_timeout`: an idle open transaction holds locks and blocks vacuum cleanup. Be wary of `idle_session_timeout` on pooled connections: the pooling layer may not react well to the closure. https://www.postgresql.org/docs/18/runtime-config-client.html
- `lock_timeout` at or above `statement_timeout` is useless. https://www.postgresql.org/docs/18/runtime-config-client.html
- `deadlock_timeout` (1s) also sets when `log_lock_waits` reports a wait. https://www.postgresql.org/docs/18/runtime-config-locks.html

## LISTEN and NOTIFY

- Notifications are delivered only on commit, and listeners in a transaction get them after it ends. Keep transactions short. https://www.postgresql.org/docs/18/sql-notify.html
- Race-free start: LISTEN and commit, then read the current state, then process notifications. https://www.postgresql.org/docs/18/sql-listen.html
- Payloads stay under 8000 bytes: send a key and store the data in a table. Identical payloads in one transaction collapse into one. https://www.postgresql.org/docs/18/sql-notify.html
- Use `pg_notify(channel, payload)` for computed values. Fire NOTIFY from a statement trigger so writers cannot forget it. https://www.postgresql.org/docs/18/sql-notify.html
- A listener stuck in a long transaction blocks queue cleanup. When the queue is full, NOTIFY fails at commit. https://www.postgresql.org/docs/18/sql-notify.html
