# Locks and coordination primitives

## Contents
- Choosing between a mutex and a task
- std Mutex in async code
- tokio::sync::Mutex
- RwLock
- Semaphore
- Notify
- OnceCell and SetOnce
- Barrier
- Runtime independence

## Choosing between a mutex and a task

- Share plain data behind a mutex. For state that needs async work such as I/O, spawn a task that owns it and talk to it by message passing. https://tokio.rs/tokio/tutorial/shared-state
- Default: `std::sync::Mutex` (or `parking_lot`) in async code. The docs call it "ok and often preferred". It stays correct while contention is low and the lock is never held across `.await`. https://docs.rs/tokio/latest/tokio/sync/struct.Mutex.html and https://tokio.rs/tokio/tutorial/shared-state
- Escape hatch: `tokio::sync::Mutex` only when the guard must be held across `.await`, typically for an I/O resource such as a database connection. Even then a task that owns the resource is often better. https://docs.rs/tokio/latest/tokio/sync/struct.Mutex.html
- Switching to the Tokio mutex usually does not help performance, because it uses a synchronous mutex internally. https://tokio.rs/tokio/tutorial/shared-state
- When contention on a sync mutex hurts, use a dedicated task, shard the mutex or restructure the code. https://tokio.rs/tokio/tutorial/shared-state

```rust
// Sharding: N maps, pick one by hash(key) % db.len()
type ShardedDb = Arc<Vec<Mutex<HashMap<String, Vec<u8>>>>>;
```
https://tokio.rs/tokio/tutorial/shared-state

- On a current-thread runtime a mutex is never contended. https://tokio.rs/tokio/tutorial/shared-state

## std Mutex in async code

- `std::sync::MutexGuard` is not `Send`, so a future holding it across `.await` cannot be spawned ("future cannot be sent between threads safely"). https://tokio.rs/tokio/tutorial/shared-state
- `drop(guard)` before the `.await` does not fix the error. The compiler computes `Send` from scopes only. End the guard in a block scope. https://tokio.rs/tokio/tutorial/shared-state
- Do not silence the error by spawning the task in a way that does not require `Send`, or by using a mutex crate whose guard is `Send`. Both compile and deadlock when another task on the same thread locks while the first is suspended. https://tokio.rs/tokio/tutorial/shared-state
- Safest pattern: wrap the mutex in a struct and lock only in non-async methods. https://tokio.rs/tokio/tutorial/shared-state

```rust
struct CanIncrement { mutex: Mutex<i32> }
impl CanIncrement {
    // This function is not marked async.
    fn increment(&self) {
        let mut lock = self.mutex.lock().unwrap();
        *lock += 1;
    }
}
```
https://tokio.rs/tokio/tutorial/shared-state

## tokio::sync::Mutex

- `lock().await` waits without blocking and the guard is `Send`. Tasks acquire the lock in FIFO order of their `lock` calls. https://docs.rs/tokio/latest/tokio/sync/struct.Mutex.html
- No poisoning: if the holder panics, the mutex is unlocked and the data may be inconsistent if the panic is caught. https://docs.rs/tokio/latest/tokio/sync/struct.Mutex.html
- Cancelling `lock` loses the place in the FIFO queue. https://docs.rs/tokio/latest/tokio/sync/struct.Mutex.html#method.lock
- `lock_owned` on an `Arc<Mutex<_>>` returns a `'static` `OwnedMutexGuard` that can move into a spawned task. https://docs.rs/tokio/latest/tokio/sync/struct.Mutex.html#method.lock_owned
- `blocking_lock` panics in an async context. From async code call it inside `spawn_blocking`: https://docs.rs/tokio/latest/tokio/sync/struct.Mutex.html#method.blocking_lock

```rust
let mutex1 = Arc::clone(&mutex);
tokio::task::spawn_blocking(move || {
    let mut n = mutex1.blocking_lock();
    *n = 2;
});
```
https://docs.rs/tokio/latest/tokio/sync/struct.Mutex.html#method.blocking_lock

- `static LOCK: Mutex<i32> = Mutex::const_new(5);` declares a static mutex. https://docs.rs/tokio/latest/tokio/sync/struct.Mutex.html#method.const_new

## RwLock

- Fair and write-preferring: a read lock waits behind every earlier queued writer. This differs from `std::sync::RwLock`, whose policy depends on the OS. https://docs.rs/tokio/latest/tokio/sync/struct.RwLock.html
- Never take a second read lock in a task that already holds one. If a writer queued in between, the second read waits behind it and the task deadlocks. https://docs.rs/tokio/latest/tokio/sync/struct.RwLock.html#method.read
- Cancelling `read` or `write` loses the place in the queue. https://docs.rs/tokio/latest/tokio/sync/struct.RwLock.html#method.write
- `blocking_read` and `blocking_write` panic in an async context. https://docs.rs/tokio/latest/tokio/sync/struct.RwLock.html#method.blocking_read
- `with_max_readers` panics when `max_reads` is 0 or above `u32::MAX >> 3`. https://docs.rs/tokio/latest/tokio/sync/struct.RwLock.html#method.with_max_readers

## Semaphore

- Use it to bound concurrency. The docs show limiting open files and incoming requests. https://docs.rs/tokio/latest/tokio/sync/struct.Semaphore.html
- Acquire an owned permit before spawning, so the number of tasks stays bounded. The permit is released when the task drops it. https://docs.rs/tokio/latest/tokio/sync/struct.Semaphore.html#method.acquire_owned

```rust
let permit = semaphore.clone().acquire_owned().await.unwrap();
let (mut socket, _) = listener.accept().await?;
tokio::spawn(async move { /* work, then permit dropped */ });
```
https://docs.rs/tokio/latest/tokio/sync/struct.Semaphore.html#method.acquire_owned

- Permits go out in request order. An `acquire_many` at the head of the queue that needs more permits than available blocks later `acquire` calls even when enough permits are free for them. https://docs.rs/tokio/latest/tokio/sync/struct.Semaphore.html
- Cancelling an acquire loses the place in the queue. https://docs.rs/tokio/latest/tokio/sync/struct.Semaphore.html#method.acquire
- Acquire methods return `AcquireError` once the semaphore is closed. https://docs.rs/tokio/latest/tokio/sync/struct.Semaphore.html#method.acquire
- `Semaphore::new` panics above `Semaphore::MAX_PERMITS`. https://docs.rs/tokio/latest/tokio/sync/struct.Semaphore.html#method.new
- Rate limiting with `add_permits` plus `SemaphorePermit::forget` is suboptimal for small durations, because it consumes a lot of CPU looping and sleeping. https://docs.rs/tokio/latest/tokio/sync/struct.Semaphore.html

## Notify

- `Notify` signals a task without data. `notify_one` before anyone waits stores one permit, so the next `notified().await` returns at once. Several calls still store a single permit. https://docs.rs/tokio/latest/tokio/sync/struct.Notify.html#method.notify_one
- `notify_waiters` wakes only waiters already registered and stores no permit. A `Notified` future registers when `notified()` creates it. https://docs.rs/tokio/latest/tokio/sync/struct.Notify.html#method.notify_waiters
- With several consumers, call `enable()` on the pinned `Notified` before checking the condition, otherwise a wakeup can be lost and a consumer sleeps forever. https://docs.rs/tokio/latest/tokio/sync/futures/struct.Notified.html#method.enable

```rust
let future = self.notify_on_sent.notified();
tokio::pin!(future);
loop {
    future.as_mut().enable();
    if let Some(msg) = self.try_recv() { return msg; }
    future.as_mut().await;
    future.set(self.notify_on_sent.notified());
}
```
https://docs.rs/tokio/latest/tokio/sync/futures/struct.Notified.html#method.enable

- Cancelling `notified` loses the place in the queue. https://docs.rs/tokio/latest/tokio/sync/struct.Notify.html#method.notified

## OnceCell and SetOnce

- `OnceCell` initializes once with an async closure, for example a global initialized on first use. https://docs.rs/tokio/latest/tokio/sync/struct.OnceCell.html

```rust
static ONCE: OnceCell<u32> = OnceCell::const_new();
async fn get_global_integer() -> &'static u32 {
    ONCE.get_or_init(|| async { 1 + 1 }).await
}
```
https://docs.rs/tokio/latest/tokio/sync/struct.OnceCell.html

- If the init future is cancelled, panics or (with `get_or_try_init`) returns an error, one waiting task starts a new attempt. https://docs.rs/tokio/latest/tokio/sync/struct.OnceCell.html#method.get_or_try_init
- Initializing the same cell recursively from inside its init closure deadlocks. https://docs.rs/tokio/latest/tokio/sync/struct.OnceCell.html#method.get_or_init
- `set` fails with `AlreadyInitializedError` or, while another task initializes, `InitializingError`. https://docs.rs/tokio/latest/tokio/sync/struct.OnceCell.html#method.set
- `SetOnce` suits a value supplied by another task: waiters call `wait().await` until `set`, and later `set` calls fail with `SetOnceError`. https://docs.rs/tokio/latest/tokio/sync/struct.SetOnce.html

## Barrier

- `Barrier::new(n)` blocks `n`-1 tasks that call `wait` and wakes them all when the n-th task calls `wait`. https://docs.rs/tokio/latest/tokio/sync/struct.Barrier.html#method.new
- One arbitrary waiting task gets `is_leader()` true, the others get false. The barrier is reusable after all tasks rendezvous. `Barrier::wait` is not cancel safe. https://docs.rs/tokio/latest/tokio/sync/struct.Barrier.html#method.wait

## Runtime independence

- `tokio::sync` primitives work outside Tokio and across runtimes. Methods ending in `_timeout` are the exception, they need the Tokio timer. https://docs.rs/tokio/latest/tokio/sync/index.html
- There is no spsc channel (use mpsc) and no spmc channel (use broadcast). A `JoinHandle` can return a task result without a `oneshot`. https://docs.rs/tokio/latest/tokio/sync/index.html
