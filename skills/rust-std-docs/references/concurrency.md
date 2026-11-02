# Threads, locks, once cells, channels and atomics (std 1.99.1)

## Contents
- Choosing a tool
- Arc
- Mutex and RwLock
- OnceLock and LazyLock
- Condvar and mpsc
- Threads
- Atomics
- Old patterns

## Choosing a tool

- Prefer `Arc`, `Mutex`, `RwLock`, `Condvar`, `mpsc`, `OnceLock`, `LazyLock` over raw atomics and fences, which are error-prone. https://doc.rust-lang.org/std/sync/index.html
- Global initialized once by a fixed closure: `LazyLock` (1.80.0). Initializer needs caller input: `OnceLock` (1.70.0). https://doc.rust-lang.org/std/sync/struct.OnceLock.html
- Borrowing local data in threads: `thread::scope` (1.63.0), no `Arc` and no `'static` needed. https://doc.rust-lang.org/std/thread/fn.scope.html

```rust
let mut a = vec![1, 2, 3];
let mut x = 0;
thread::scope(|s| {
    s.spawn(|| println!("{a:?}"));
    s.spawn(|| x += a[0] + a[2]);
});
a.push(4);
```

## Arc

- `Arc<T>` is `Send`/`Sync` only if `T: Send + Sync`. `Arc<RefCell<T>>` is not `Send`. Pair `Arc` with `Mutex` or `RwLock`. https://doc.rust-lang.org/std/sync/struct.Arc.html
- To mutate through an `Arc`: a lock or atomic inside, `Arc::make_mut` (clones only when shared), or `Arc::get_mut` when unique. https://doc.rust-lang.org/std/sync/struct.Arc.html
- Use `Arc::into_inner(a)` when discarding the `Err`. With `Arc::try_unwrap(a).ok()` in 2 threads, both can lose the value. https://doc.rust-lang.org/std/sync/struct.Arc.html
- `Arc` cycles never free. Use `Weak` back-pointers or `Arc::new_cyclic`. https://doc.rust-lang.org/std/sync/struct.Arc.html

## Mutex and RwLock

- A thread that panics while holding a lock poisons it. `lock()` then returns `Err(PoisonError)`. Recover with `into_inner`, and clear with `clear_poison` (1.77.0). https://doc.rust-lang.org/std/sync/struct.Mutex.html

```rust
let guard = m.lock().unwrap_or_else(|e| e.into_inner());
```

- Poisoning is advisory and not exhaustive. Unsafe code must not rely on it. https://doc.rust-lang.org/std/sync/struct.Mutex.html#poisoning
- Locking a `Mutex` the current thread already holds may deadlock or panic. std has no stable reentrant lock. https://doc.rust-lang.org/std/sync/struct.Mutex.html
- Drop a guard early with an inner block or `drop(guard)`. Holding it while joining threads can deadlock. A guard used as a temporary drops at the end of the statement. https://doc.rust-lang.org/std/sync/struct.Mutex.html
- `RwLock<T>` is shared across threads only when `T: Send + Sync`. https://doc.rust-lang.org/std/sync/struct.RwLock.html
- `RwLock` priority is OS dependent. Taking a second `read()` on the same thread while a writer waits may deadlock. https://doc.rust-lang.org/std/sync/struct.RwLock.html
- `RwLock` is poisoned only by a panic in write mode. https://doc.rust-lang.org/std/sync/struct.RwLock.html
- Not stable on 1.99.1: `Mutex::get_cloned`, `set`, `replace`, the `nonpoison` and `poison` modules, `ReentrantLock`, `mpmc`, `oneshot`. https://doc.rust-lang.org/std/sync/index.html

## OnceLock and LazyLock

```rust
static CONFIG: LazyLock<String> = LazyLock::new(|| load_config());
static PORT: OnceLock<u16> = OnceLock::new();
let port = *PORT.get_or_init(|| 8080);
```

https://doc.rust-lang.org/std/sync/struct.LazyLock.html

- If a `LazyLock` init panics, it is poisoned forever. Every later access from any thread panics, with no recovery. If init can fail, use `OnceLock` and handle the error before `set`. https://doc.rust-lang.org/std/sync/struct.LazyLock.html#poisoning
- Default: infallible init uses `LazyLock`, fallible init uses `OnceLock`. Load first, handle the error outside the cell, then `get_or_init` or `set`. Do not keep a `LazyLock<Result<..>>`. `get_or_try_init` is marked nightly-only (`once_cell_try`), so it is not available on stable 1.99.1. https://doc.rust-lang.org/std/sync/struct.OnceLock.html#method.get_or_try_init

```rust
static CFG: OnceLock<Config> = OnceLock::new();
fn cfg() -> Result<&'static Config, LoadError> {
    if let Some(c) = CFG.get() { return Ok(c); }
    let loaded = load()?; // error handled outside the cell
    Ok(CFG.get_or_init(|| loaded))
}
```

- With `get_or_init`, many threads may race with different initializers but only one runs, as long as it does not panic. https://doc.rust-lang.org/std/sync/struct.OnceLock.html#method.get_or_init
- A `OnceLock` is never poisoned. If the init closure panics, it stays empty. Reentrant init from inside the closure has an unspecified outcome (deadlock in the current implementation, possibly a panic later). https://doc.rust-lang.org/std/sync/struct.OnceLock.html
- `OnceLock::get` never blocks. `set` returns `Err(value)` if already set. `OnceLock::wait` (1.86.0) blocks until another thread sets it. https://doc.rust-lang.org/std/sync/struct.OnceLock.html
- Statics never run `Drop`, so a `static LazyLock` is reported as a leak by memory checkers. This is intended. https://doc.rust-lang.org/std/sync/struct.LazyLock.html
- `LazyLock::get`, `get_mut`, `force_mut` are stable since 1.94.0. https://doc.rust-lang.org/std/sync/struct.LazyLock.html

## Condvar and mpsc

- Always re-check the condition in a loop (spurious wakeups), or use `wait_while`. For timed waits use `wait_timeout_while`, which also tracks the shrinking timeout. https://doc.rust-lang.org/std/sync/struct.Condvar.html

```rust
let (lock, cvar) = &*pair;
let guard = cvar.wait_while(lock.lock().unwrap(), |pending| *pending).unwrap();
```

- The predicate is always verified inside the mutex. Use one mutex per `Condvar` over time, else `wait` may panic. https://doc.rust-lang.org/std/sync/struct.Condvar.html
- `channel()` is unbounded. `sync_channel(n)` blocks the sender when full, `sync_channel(0)` is a rendezvous. https://doc.rust-lang.org/std/sync/mpsc/index.html
- `recv()` returns `Err` only once ALL senders are dropped. Drop the original `tx` after cloning one per thread, or the receive loop hangs. https://doc.rust-lang.org/std/sync/mpsc/index.html

```rust
for _ in 0..3 { let tx = tx.clone(); thread::spawn(move || tx.send("ok").unwrap()); }
drop(tx);
while let Ok(msg) = rx.recv() { println!("{msg}"); }
```

## Threads

- When `main` returns, the process exits and other threads stop and destructors on their stacks may not run. Join the threads you need. https://doc.rust-lang.org/std/thread/index.html
- Dropping a `JoinHandle` detaches the thread. https://doc.rust-lang.org/std/thread/fn.spawn.html
- `thread::spawn` panics if the OS cannot create a thread. Use `thread::Builder::new().name(..).stack_size(..).spawn(..)`, which returns `io::Result`. https://doc.rust-lang.org/std/thread/struct.Builder.html
- `join()` returns `Err(payload)` if the thread panicked. `thread::scope` panics if an auto-joined thread panicked, so join scoped handles yourself to handle it. https://doc.rust-lang.org/std/thread/fn.scope.html
- Default spawned-thread stack is 2 MiB on Tier-1 platforms. Set `Builder::stack_size` for deep recursion. https://doc.rust-lang.org/std/thread/index.html
- A thread name containing `\0` makes `spawn` panic. https://doc.rust-lang.org/std/thread/struct.Builder.html
- `thread::available_parallelism()` returns an estimate of the parallelism a program should use. https://doc.rust-lang.org/std/thread/index.html
- Even functions "documented not to block" may block internally. This matters on multithreaded WebAssembly main threads. https://doc.rust-lang.org/std/sync/index.html#blocking-guarantees

## Atomics

```rust
static COUNT: AtomicUsize = AtomicUsize::new(0);
COUNT.fetch_add(1, Ordering::Relaxed);
```

- Pair a `Release` store with an `Acquire` load to publish data. `Release` on a read-modify-write makes its load `Relaxed`, and `Acquire` makes its store `Relaxed`. Use `AcqRel` for both. https://doc.rust-lang.org/std/sync/atomic/enum.Ordering.html
- `Ordering` is `#[non_exhaustive]`. A `match` needs a `_` arm. https://doc.rust-lang.org/std/sync/atomic/enum.Ordering.html
- Portability: no `AtomicU64` on 32-bit PowerPC and MIPS, and `thumbv6m` has only `load`/`store`. Gate with `#[cfg(target_has_atomic = "64")]`. `AtomicUsize` is the most portable. https://doc.rust-lang.org/std/sync/atomic/index.html#portability
- Any atomic access to read-only memory is UB, even a failing `compare_exchange`. https://doc.rust-lang.org/std/sync/atomic/index.html
- Atomics are not shared by themselves. Put them in an `Arc` or a `static`. https://doc.rust-lang.org/std/sync/atomic/index.html

## Old patterns

| Old | Current |
| :-- | :-- |
| `ATOMIC_USIZE_INIT` (deprecated) | `AtomicUsize::new(0)` https://doc.rust-lang.org/std/sync/atomic/index.html |
| `atomic::spin_loop_hint` (deprecated) | `std::hint::spin_loop` https://doc.rust-lang.org/std/sync/atomic/index.html |
| `thread::sleep_ms`, `Condvar::wait_timeout_ms` | `sleep(Duration)`, `wait_timeout(Duration)` https://doc.rust-lang.org/std/thread/index.html |
| `Arc<Mutex>` plus `'static` to share a local | `thread::scope` https://doc.rust-lang.org/std/thread/fn.scope.html |
