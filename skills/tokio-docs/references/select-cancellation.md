# select!, join! and cancellation

## Contents
- select! semantics
- Cancel safety
- Keeping a future across loop iterations
- Fairness and biased
- join! and try_join!
- Graceful shutdown
- Hand-written futures

## select! semantics

- `select!` returns when the first branch completes and drops the others. All branches run concurrently on the current task, never in parallel: a blocking branch stalls the rest. For parallelism, spawn each future and select on the `JoinHandle`s. https://docs.rs/tokio/latest/tokio/macro.select.html
- Branch form: `<pattern> = <async expression> (, if <precondition>)? => <handler>,` plus an optional `else`. https://docs.rs/tokio/latest/tokio/macro.select.html
- A false `if` precondition disables the branch for this call only, yet its async expression is still evaluated (never polled). https://docs.rs/tokio/latest/tokio/macro.select.html
- A value that does not match the pattern disables that branch for the rest of the call. `select!` panics when every branch is disabled and there is no `else`. https://docs.rs/tokio/latest/tokio/macro.select.html
- `Some(v) = rx.recv()` disables a branch once its channel closes. Add `else` to leave the loop when all are closed: https://tokio.rs/tokio/tutorial/select

```rust
loop {
    let msg = tokio::select! {
        Some(msg) = rx1.recv() => msg,
        Some(msg) = rx2.recv() => msg,
        else => { break }
    };
}
```
https://tokio.rs/tokio/tutorial/select

- `?` inside the async expression goes to that branch's output. `?` inside the handler returns from the enclosing function. https://tokio.rs/tokio/tutorial/select
- All handlers return the same type. Branches may borrow data, unlike spawned tasks. The limit is 64 branches. https://tokio.rs/tokio/tutorial/select

## Cancel safety

- A future is cancel safe when dropping it before completion and creating it again loses nothing. It matters most for `select!` in a loop, where losing branches are dropped on each iteration. https://docs.rs/tokio/latest/tokio/macro.select.html
- Cancel safe: `mpsc` `recv`, `broadcast` `recv`, `watch` `changed`, `TcpListener::accept`, `UnixListener::accept`, `signal::unix::Signal::recv`, `AsyncReadExt::read` and `read_buf`, `AsyncWriteExt::write` and `write_buf`, `StreamExt::next`. https://docs.rs/tokio/latest/tokio/macro.select.html
- Not cancel safe, data is lost: `read_exact`, `read_to_end`, `read_to_string`, `write_all`. https://docs.rs/tokio/latest/tokio/macro.select.html
- Not cancel safe, the place in the fair queue is lost: `Mutex::lock`, `RwLock::read`, `RwLock::write`, `Semaphore::acquire`, `Notify::notified`. https://docs.rs/tokio/latest/tokio/macro.select.html
- `mpsc::Sender::send` in a losing branch drops the message. Use `reserve()` then `Permit::send`. https://docs.rs/tokio/latest/tokio/sync/mpsc/struct.Sender.html#method.send
- Cancelling a non-cancel-safe future is fine when the partial data no longer matters, for example during shutdown. https://docs.rs/tokio/latest/tokio/macro.select.html
- For a `loop { select! { .. } }` over several sources, the docs suggest merging them into one stream (`tokio_stream`, `futures::stream`, `futures_concurrency`) and processing items sequentially. They say `select!` can introduce hard-to-debug cancellation safety errors. https://docs.rs/tokio/latest/tokio/macro.select.html
- Dropping a future drops its own state, but background tasks it spawned can keep running. Tokio's `oneshot::Receiver` implements `Drop` by sending a closed notification to the sender, which can watch `tx.closed()` and abort: https://tokio.rs/tokio/tutorial/select

```rust
tokio::select! {
    val = some_operation() => { let _ = tx1.send(val); }
    _ = tx1.closed() => { /* some_operation() is canceled */ }
}
```
https://tokio.rs/tokio/tutorial/select

## Keeping a future across loop iterations

- Create the future outside the loop, pin it with `tokio::pin!`, and select on `&mut fut`. The error "no method named `poll` found for `Pin<&mut &mut impl Future>`" means it is not pinned. https://tokio.rs/tokio/tutorial/select

```rust
let operation = action();
tokio::pin!(operation);
loop {
    tokio::select! {
        _ = &mut operation => break,
        Some(v) = rx.recv() => { if v % 2 == 0 { break; } }
    }
}
```
https://tokio.rs/tokio/tutorial/select

- Polling a completed `&mut fut` again panics with "`async fn` resumed after completion". Guard it with `res = &mut operation, if !done`, and reset `done` when `operation.set(action(..))` installs a new future. https://tokio.rs/tokio/tutorial/select
- For a deadline over the whole loop, pin one `sleep` outside the loop and select on `&mut sleep`. Do not guard it with `if !sleep.is_elapsed()`: that can be true before the poll returned `Ready`, so the timeout is missed. https://docs.rs/tokio/latest/tokio/macro.select.html

```rust
let sleep = time::sleep(Duration::from_secs(1));
tokio::pin!(sleep);
loop {
    tokio::select! {
        maybe_v = stream.next() => { /* ... */ }
        _ = &mut sleep => { println!("timeout"); break; }
    }
}
```
https://docs.rs/tokio/latest/tokio/macro.select.html

## Fairness and biased

- Default: `select!` polls a random branch first on each call, so an always-ready branch cannot starve the others. https://docs.rs/tokio/latest/tokio/macro.select.html
- Escape hatch: `biased;` as the first token polls top to bottom, when the order matters or to skip the random number cost. Fairness is then yours: put the shutdown branch before a stream that is almost always ready. https://docs.rs/tokio/latest/tokio/macro.select.html

```rust
tokio::select! {
    biased;
    _ = shutdown.recv() => break,
    Some(v) = stream.next() => handle(v),
}
```
Shape of the documented advice, not a docs example. https://docs.rs/tokio/latest/tokio/macro.select.html

## join! and try_join!

- `join!` waits for every branch, even after an `Err`. `try_join!` returns when all branches are `Ok` or at the first `Err`. https://docs.rs/tokio/latest/tokio/macro.join.html and https://docs.rs/tokio/latest/tokio/macro.try_join.html
- Both are concurrent on one task, not parallel. For parallelism, spawn and join the handles. With `try_join!` on handles, flatten `Result<Result<T, E>, JoinError>` into one `Result` first. https://docs.rs/tokio/latest/tokio/macro.try_join.html
- Both rotate which branch is polled first. `biased;` polls in order. https://docs.rs/tokio/latest/tokio/macro.join.html

## Graceful shutdown

- Three parts: decide when to shut down, tell every task, wait for them. https://tokio.rs/tokio/topics/shutdown
- Decide: select on `tokio::signal::ctrl_c()` and an mpsc channel that collects other shutdown reasons. Treat an `Err` from `ctrl_c()` as a reason to stop too. https://tokio.rs/tokio/topics/shutdown

```rust
tokio::select! {
    _ = signal::ctrl_c() => {},
    _ = shutdown_recv.recv() => {},
}
```
https://tokio.rs/tokio/topics/shutdown

- Tell: clone a `CancellationToken` (tokio-util) into each task, which selects on `token.cancelled()` and may clean up first. Cancelling any clone cancels all. https://tokio.rs/tokio/topics/shutdown
- Wait: spawn through a `TaskTracker` (tokio-util), call `close()`, then `wait().await`. `wait` resolves only once the tracker is closed and every task is done. https://tokio.rs/tokio/topics/shutdown

```rust
let tracker = TaskTracker::new();
for i in 0..10 { tracker.spawn(some_operation(i)); }
tracker.close();
tracker.wait().await;
```
https://tokio.rs/tokio/topics/shutdown

## Hand-written futures

- A future returning `Poll::Pending` must arrange for its waker to be called, otherwise the task hangs forever. In the tutorial's `MainFuture`, `cx` is passed to the inner `Delay` future and `Pending` is returned only when it returns `Pending`. https://tokio.rs/tokio/tutorial/async
- Calling `cx.waker().wake_by_ref()` before every `Pending` creates a busy loop. https://tokio.rs/tokio/tutorial/async
