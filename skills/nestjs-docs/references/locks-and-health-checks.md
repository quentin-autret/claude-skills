# Distributed locks (@nestjs/locks) and health checks (@nestjs/terminus)

## Contents
- Why locks
- Store
- Job decorators
- Fencing and the abort signal
- Manual locks and leader election
- Health checks

## Why locks

- `@nestjs/schedule` fires every job on every instance. A `RUN_JOBS=true` instance breaks during rolling deploys, and a Redis `SET NX` breaks when the holder pauses past the TTL. `waitForCompletion` only prevents overlap on one instance. https://docs.nestjs.com/reliability/locks

## Store

- The package ships no shared store, only the `LockStore` contract. The default in-memory store only excludes callers in one process. With `NODE_ENV=production` and no store, startup fails (`allowInMemoryStorage: true` for a single instance only). https://docs.nestjs.com/reliability/locks#keep-locks-in-your-database
- A store is a singleton provider calling `storage.registerSource(this)` on `LocksStorage` in its constructor: https://docs.nestjs.com/reliability/locks#keep-locks-in-your-database
  ```typescript
  @Injectable()
  export class DrizzleLockStore implements LockStore {
    constructor(@InjectDrizzle() private readonly db: Database, storage: LocksStorage) {
      storage.registerSource(this);
    }
  }
  ```
- `acquire(key, owner, ttl)`, `renew()`, `release()` must each be one atomic statement checking the owner in the write. Use a table, not `pg_advisory_lock()` (bound to one session, no expiry, no token). https://docs.nestjs.com/reliability/locks#keep-locks-in-your-database
- PostgreSQL details: `INSERT ... ON CONFLICT DO UPDATE ... WHERE owner IS NULL OR expires_at <= clock_timestamp()`, token from `greatest(nextval(seq), row.token + 1)`, `release()` clears the owner without deleting the row, `clock_timestamp()` not `now()`. `pg` returns `bigint` as a string: convert it. https://docs.nestjs.com/reliability/locks#keep-locks-in-your-database
- Redis stores need persistence and `maxmemory-policy noeviction`, and a hash tag (`{locks}:...`) on Redis Cluster. Async replica failover can lose fencing increments. https://docs.nestjs.com/reliability/locks#the-store-on-redis
- Test stores with `lockStoreContract()` from `@nestjs/locks/testing`, `concurrent: true`, on a pooled server. https://docs.nestjs.com/reliability/locks#the-store-contract

## Job decorators

```typescript
@Cron(process.env.INVOICE_EXPORT_CRON ?? '0 2 * * *', { name: 'invoice-export' })
@OnOneInstance({ key: 'invoices:nightly-export' })
@WithoutOverlapping()
async exportInvoices() {}
```
https://docs.nestjs.com/reliability/locks#run-the-export-by-hand

- `@OnOneInstance()` gives sticky ownership: the first instance to fire keeps the lease (renewed every `ttl / 3`) and others skip ticks (nothing is queued). A dead owner is replaced after `ttl` (default `'30s'`). https://docs.nestjs.com/reliability/locks#run-each-job-on-one-instance
- It does not stop the owner overlapping itself. Add `@WithoutOverlapping()`. https://docs.nestjs.com/reliability/locks#dont-let-a-long-run-overlap-itself
- Set `key` explicitly: the default `ClassName.methodName` changes on rename, and old and new instances would each run the job during a deploy. https://docs.nestjs.com/reliability/locks#run-each-job-on-one-instance
- Missed ticks are not caught up. Write jobs that process everything due, not "yesterday's" work. https://docs.nestjs.com/reliability/locks#run-each-job-on-one-instance
- Call `app.enableShutdownHooks()` so leases are handed over at deploy instead of expiring. https://docs.nestjs.com/reliability/locks#production-checklist

## Fencing and the abort signal

- A paused holder can still write after losing the lease. Fence irreversible writes with the token in the same statement: https://docs.nestjs.com/reliability/locks#fence-the-writes
  ```typescript
  const { fencingToken, signal } = this.locksContext;
  const counts = await this.warehouseClient.stockCounts({ signal });
  await this.stockRepository.writeStock(counts, fencingToken!);
  // WHERE id = $1 AND stock_fencing_token <= $token
  ```
- The signal aborts when the lock is lost (`LockLostError`). Nothing is killed: code that ignores it keeps running. The package emits a `lock-lost` event: alert on it. https://docs.nestjs.com/reliability/locks#fence-the-writes https://docs.nestjs.com/reliability/locks#production-checklist

## Manual locks and leader election

- `locks.withLock(KEY, fn, { wait: '2s' })` releases whatever `fn` does and rejects with `LockNotAcquiredError` (map it to 409 yourself). Share the scheduled job's key so manual runs never overlap it. `acquire()` returns a `Lock` usable with `await using`. https://docs.nestjs.com/reliability/locks#run-the-export-by-hand
- `@LeaderElection(key)` on a singleton implementing `OnLeadershipAcquired`/`OnLeadershipLost`. Actions taken for the group must carry the fencing token. https://docs.nestjs.com/reliability/locks#hold-the-warehouse-feed-on-one-instance
- Tests: `ManualLockClock` through `LOCKS_MODULE_OPTIONS`, `await clock.advance('5m')`. https://docs.nestjs.com/reliability/locks#testing

## Health checks

```typescript
@Get()
@HealthCheck()
check() {
  return this.health.check([
    () => this.db.pingCheck('database').withTimeout(1500).cacheFor(5000),
  ]);
}
```
https://docs.nestjs.com/reliability/terminus#caching

- `HttpHealthIndicator` needs `HttpModule` from `@nestjs/axios` in the health module. https://docs.nestjs.com/reliability/terminus#http-healthcheck
- Custom indicators inject `HealthIndicatorService`. `attempt()` turns a throw into `'down'` and passes an `AbortSignal` honored by `withTimeout()`: https://docs.nestjs.com/reliability/terminus#custom-health-indicator
  ```typescript
  return this.healthIndicatorService.check(key)
    .attempt(async ({ signal }) => { await fetch(url, { signal }); return {}; })
    .withTimeout(1000);
  ```
- With `up()`/`down()`/`degraded()`, a thrown error is a bug: it aborts the whole check with 500 instead of 503. Use `attempt()` around operations that can throw. https://docs.nestjs.com/reliability/terminus#degraded-state
- `degraded()` keeps HTTP 200 and status `'degraded'`, so orchestrators keep routing. https://docs.nestjs.com/reliability/terminus#degraded-state
- `cacheFor(ms)` shares one result and one in-flight run across requests. https://docs.nestjs.com/reliability/terminus#caching
- `gracefulShutdownTimeoutMs` reports `'shutting_down'` with 503 before shutting down, on `SIGTERM` only, and requires shutdown hooks. https://docs.nestjs.com/reliability/terminus#graceful-shutdown-timeout

## Old patterns

| Old | Current |
| --- | --- |
| extend `HealthIndicator`, throw `HealthCheckError` (removed in v12) | `HealthIndicatorService.check(key)` with `up()`/`down()` or `attempt()` |
| `pingCheck('db', { timeout: 1500 })` (deprecated) | `pingCheck('db').withTimeout(1500)` |

https://docs.nestjs.com/migration-guide#terminus-module https://docs.nestjs.com/reliability/terminus#timeouts
