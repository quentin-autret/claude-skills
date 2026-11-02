# Tasks, blocking work and cooperative scheduling

## Contents
- spawn
- JoinHandle and abort
- JoinSet
- Blocking and CPU-bound work
- Cooperative scheduling
- LocalSet and spawn_local
- Task-locals
- Old patterns

## spawn

- The future starts running when `spawn` is called, even if the `JoinHandle` is never awaited. https://docs.rs/tokio/latest/tokio/task/fn.spawn.html
- `spawn` never polls the future synchronously, so spawning while holding a lock cannot deadlock with the new task. https://docs.rs/tokio/latest/tokio/task/fn.spawn.html
- A spawned task is not guaranteed to finish: runtime shutdown drops all outstanding tasks. Returning from `#[tokio::main]` cancels them immediately. https://docs.rs/tokio/latest/tokio/task/fn.spawn.html and https://docs.rs/tokio/latest/tokio/task/index.html#cancellation
- The future must be `Send + 'static`. A task is `Send` when everything held across an `.await` is `Send`. A `!Send` value such as `Rc` is fine if its scope ends before the next `.await`. https://tokio.rs/tokio/tutorial/spawning
- A compile error ending in "cannot be sent between threads safely" means a `!Send` value lives across an `.await`. https://docs.rs/tokio/latest/tokio/task/fn.spawn.html
- Code that is not `async` but called from async code still runs in the async context. Do not block there, destructors of values dropped in async code included. https://docs.rs/tokio/latest/tokio/task/index.html
- An accept loop that awaits `process(socket)` inline serves one connection at a time. Spawn one task per connection. https://tokio.rs/tokio/tutorial/spawning

## JoinHandle and abort

- `handle.await` returns `Result<T, JoinError>`. A panic in the task becomes a `JoinError` (`is_panic()`). For a task returning `Result`, use `handle.await??`. https://docs.rs/tokio/latest/tokio/task/struct.JoinHandle.html
- Dropping a `JoinHandle` detaches the task: it keeps running and its output is lost. https://docs.rs/tokio/latest/tokio/task/struct.JoinHandle.html
- Awaiting `&mut JoinHandle` is cancel safe in `select!`. https://docs.rs/tokio/latest/tokio/task/struct.JoinHandle.html
- `abort()` only schedules cancellation. The task stops at its next `.await`. Await the handle to wait for it. If the task finishes without yielding, the handle reports normal completion, not a cancelled error. https://docs.rs/tokio/latest/tokio/task/index.html#cancellation
- `is_finished()` can stay false after `abort()` until cancellation completes. https://docs.rs/tokio/latest/tokio/task/struct.JoinHandle.html#method.is_finished
- `abort_handle()` returns an `AbortHandle` that aborts but cannot wait. A task has several abort handles but one `JoinHandle`. https://docs.rs/tokio/latest/tokio/task/index.html#cancellation

## JoinSet

- Default for a group of tasks of one output type: `JoinSet`. Results come in completion order. https://docs.rs/tokio/latest/tokio/task/struct.JoinSet.html
- Dropping a `JoinSet` aborts all its tasks, unlike dropping a `JoinHandle`. https://docs.rs/tokio/latest/tokio/task/struct.JoinSet.html

```rust
let mut set = JoinSet::new();
for i in 0..10 { set.spawn(async move { i }); }
while let Some(res) = set.join_next().await { let idx = res.unwrap(); }
```
https://docs.rs/tokio/latest/tokio/task/struct.JoinSet.html

- `join_next()` is cancel safe in `select!` and returns `None` when the set is empty. https://docs.rs/tokio/latest/tokio/task/struct.JoinSet.html#method.join_next
- `join_all()` panics and cancels the rest if any task fails with `JoinError`. To handle errors, loop on `join_next`. https://docs.rs/tokio/latest/tokio/task/struct.JoinSet.html#method.join_all
- `abort_all()` does not remove tasks. Keep calling `join_next` until empty, or call `shutdown()`, which does both and ignores panics. https://docs.rs/tokio/latest/tokio/task/struct.JoinSet.html#method.abort_all
- `detach_all()` removes tasks without aborting them. https://docs.rs/tokio/latest/tokio/task/struct.JoinSet.html#method.detach_all
- Escape hatch when results must come back in spawn order: push `JoinHandle`s into a `Vec` and await each. https://docs.rs/tokio/latest/tokio/task/fn.spawn.html

## Blocking and CPU-bound work

- Default for bounded blocking work: `spawn_blocking`, for non-async operations that finish on their own. Awaiting the handle yields `Result<T, JoinError>`. https://docs.rs/tokio/latest/tokio/task/fn.spawn_blocking.html and https://docs.rs/tokio/latest/tokio/task/struct.JoinHandle.html

```rust
let res = task::spawn_blocking(move || { v.push_str("world"); v }).await?;
```
https://docs.rs/tokio/latest/tokio/task/fn.spawn_blocking.html

- The blocking pool limit is very large. For many CPU-bound jobs, limit parallelism with a `Semaphore`, or use a CPU pool such as rayon and return the result through a `oneshot`. https://docs.rs/tokio/latest/tokio/task/fn.spawn_blocking.html and https://docs.rs/tokio/latest/tokio/index.html#cpu-bound-tasks-and-blocking-code
- For a long-lived or endless loop, use a dedicated `std::thread::spawn` thread. Each such task would hold a pool thread and delay queued work. https://docs.rs/tokio/latest/tokio/task/fn.spawn_blocking.html
- `abort` has no effect on a running `spawn_blocking` task. Runtime shutdown waits for started blocking tasks indefinitely, and `shutdown_timeout` stops waiting without cancelling them. https://docs.rs/tokio/latest/tokio/task/fn.spawn_blocking.html
- A current-thread runtime still spawns extra threads for `spawn_blocking`. https://docs.rs/tokio/latest/tokio/task/fn.spawn_blocking.html
- A second Tokio runtime for CPU-bound work must run only CPU-bound tasks. https://docs.rs/tokio/latest/tokio/index.html#cpu-bound-tasks-and-blocking-code
- `block_in_place` turns the current worker into a blocking thread. It panics on a current-thread runtime and inside a local task driven by `LocalSet::block_on`. Other futures of the same task (in `join!` for example) are suspended meanwhile. Prefer `spawn_blocking`. https://docs.rs/tokio/latest/tokio/task/fn.block_in_place.html and https://docs.rs/tokio/latest/tokio/task/struct.LocalSet.html#method.block_on
- To stream data between sync and async code, use `mpsc` with `blocking_send` / `blocking_recv`, or `tokio_util::io::SyncIoBridge` for bytes. https://docs.rs/tokio/latest/tokio/task/fn.spawn_blocking.html

## Cooperative scheduling

- A task that never returns `Pending`, such as a loop over an always-ready stream, starves the other tasks on its thread. Tokio's own resources consume a task budget and force a yield. https://docs.rs/tokio/latest/tokio/task/coop/index.html
- `futures::channel::mpsc` does not consume budget. Wrap such a future in `task::coop::cooperative(fut)`. https://docs.rs/tokio/latest/tokio/task/coop/fn.cooperative.html
- `task::coop::unconstrained(fut)` opts out of the budget, so the future is never forced to yield. https://docs.rs/tokio/latest/tokio/task/coop/index.html
- `yield_now().await` may not reach the runtime, for example when `select!` completes another branch in the same poll. Task order is not a stable API, so code must be correct under any order. https://docs.rs/tokio/latest/tokio/task/fn.yield_now.html
- In a hand-written future use `coop::poll_proceed(cx)` and call `made_progress()` on the returned guard when progress was made. https://docs.rs/tokio/latest/tokio/task/coop/fn.poll_proceed.html

## LocalSet and spawn_local

- For `!Send` futures use a `LocalSet`, a `LocalRuntime` or `#[tokio::main(flavor = "local")]`. `spawn_local` panics outside a `LocalSet` or `LocalRuntime`. https://docs.rs/tokio/latest/tokio/task/fn.spawn_local.html
- With a `LocalSet`, spawn inside `run_until`: https://docs.rs/tokio/latest/tokio/task/struct.LocalSet.html

```rust
let local = task::LocalSet::new();
local.run_until(async move {
    task::spawn_local(async move { /* !Send */ }).await.unwrap();
}).await;
```
https://docs.rs/tokio/latest/tokio/task/struct.LocalSet.html

- `run_until` and awaiting a `LocalSet` work only in `#[tokio::main]`, `#[tokio::test]` or directly in `Runtime::block_on`, not inside a `tokio::spawn` task. From there, run the `LocalSet` on its own thread and send it work through `mpsc`. https://docs.rs/tokio/latest/tokio/task/struct.LocalSet.html
- `tokio::spawn` inside a `LocalSet` creates a task outside the set. https://docs.rs/tokio/latest/tokio/task/fn.spawn_local.html
- Tasks still pending when `run_until` returns stay on the set and resume at the next run. https://docs.rs/tokio/latest/tokio/task/struct.LocalSet.html#method.run_until

## Task-locals

- `tokio::task_local! { static NUMBER: u32; }` has no lazy init. The value exists only inside `NUMBER.scope(value, fut).await` or `sync_scope`. https://docs.rs/tokio/latest/tokio/task/struct.LocalKey.html
- `with` and `get` panic when no value is set. Use `try_with` / `try_get` to get an `AccessError` instead. https://docs.rs/tokio/latest/tokio/task/struct.LocalKey.html#method.with

## Old patterns

| Old | Replacement |
| :-- | :-- |
| `task::consume_budget`, `task::unconstrained`, `task::Unconstrained` (deprecated) | the `task::coop::` versions. https://docs.rs/tokio/latest/tokio/task/index.html |
