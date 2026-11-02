---
name: tokio-docs
description: "Gives current guidance from the official tokio documentation (tokio 1.54.1) on async Rust pitfalls, cancel safety, blocking code, surprising defaults and deprecated APIs. Use when writing, reviewing or debugging async Rust code that uses tokio, especially the runtime (#[tokio::main], #[tokio::test], Builder, enable_all, block_on, current_thread, LocalRuntime, shutdown, feature flags), tasks (spawn, JoinHandle, JoinSet, abort, spawn_blocking, block_in_place, LocalSet, spawn_local, task_local), select!, join!, try_join!, cancellation and graceful shutdown, sync primitives (Mutex across .await, RwLock, Semaphore, Notify, OnceCell), channels (mpsc backpressure, oneshot, broadcast lag, watch), time (sleep, interval, MissedTickBehavior, timeout, paused time, start_paused), I/O and networking (AsyncReadExt, BufWriter flush, split, io::copy, framing, TcpListener, TcpStream, UdpSocket), tokio::fs, process (Command, kill_on_drop, Child), signals (ctrl_c, Windows console events) and Windows named pipes."
metadata:
  version: "1.54.1"
  source: "https://docs.rs/tokio/latest/tokio/"
  fetched: "2026-11-02"
---

# tokio (1.54.1)

## Overview

This skill holds what the official tokio docs say that is easy to get wrong: cancel safety, blocking the runtime, surprising defaults, panics and the APIs that replace old ones. Basics every async Rust developer knows are left out. Every claim in `references/` ends with the docs URL it comes from, so you can reopen the page. https://docs.rs/tokio/latest/tokio/index.html

## Version covered

tokio 1.54.1, as served by docs.rs/tokio/latest, plus the tokio.rs tutorial and topics pages. Unstable APIs need `--cfg tokio_unstable` and may break in 1.x, so do not suggest them unless the project already enables it. https://docs.rs/tokio/latest/tokio/index.html#unstable-features

## How to navigate

1. Find the domain of the code you write or review in the table below.
2. Open only that reference file. Each one stands alone, with a table of contents at the top.
3. Apply its default. Take the escape hatch only when the stated condition holds.
4. In a review, also check the "Old patterns" table at the end of the file when there is one, and suggest the replacement.
5. In your answer, give the docs URL that follows each rule you apply, so the reader can check it.

## Highest-value rules

- A runtime built with `Builder` has no I/O or timer driver. Call `enable_all()`, or networking and time types fail. https://docs.rs/tokio/latest/tokio/runtime/index.html#resource-drivers
- `#[tokio::test]` runs on a current-thread runtime, `#[tokio::main]` on a multi-thread one. `#[tokio::main]` on a function called often builds a new runtime per call. https://docs.rs/tokio/latest/tokio/attr.test.html and https://docs.rs/tokio/latest/tokio/attr.main.html
- `rt.block_on(sleep(d))` panics: the future is created outside the runtime. Wrap it, `rt.block_on(async { sleep(d).await })`. https://docs.rs/tokio/latest/tokio/time/fn.sleep.html
- On a current-thread runtime, spawned tasks run only while `Runtime::block_on` runs. `Handle::block_on` does not drive it. https://docs.rs/tokio/latest/tokio/runtime/index.html#driving-the-runtime
- Default to `std::sync::Mutex` in async code, never held across `.await`. End the guard in a block scope, since `drop(guard)` does not satisfy the `Send` check. Use `tokio::sync::Mutex` only when the guard must cross an `.await`. https://tokio.rs/tokio/tutorial/shared-state
- Run blocking or CPU-heavy code with `spawn_blocking`, and an endless loop on a `std::thread`. `abort` does not stop a running blocking task. https://docs.rs/tokio/latest/tokio/task/fn.spawn_blocking.html
- In `select!`, losing branches are dropped. `read_exact`, `read_to_end`, `write_all` and `mpsc::Sender::send` lose data there. Use `reserve()` for sends, and pin long-lived futures outside the loop and select on `&mut fut`. https://docs.rs/tokio/latest/tokio/macro.select.html and https://docs.rs/tokio/latest/tokio/sync/mpsc/struct.Sender.html#method.send
- `select!` panics when every branch is disabled and there is no `else`. https://docs.rs/tokio/latest/tokio/macro.select.html
- Prefer a bounded `mpsc::channel(n)` for backpressure. `recv` returns `None` only once every `Sender` is dropped. https://tokio.rs/tokio/tutorial/channels and https://docs.rs/tokio/latest/tokio/sync/mpsc/index.html
- A slow broadcast receiver gets `RecvError::Lagged(n)` and stays open. Handle it instead of treating every error as closed. https://docs.rs/tokio/latest/tokio/sync/broadcast/struct.Receiver.html#method.recv
- In a watch `changed()` loop use `borrow_and_update()`, and never hold a `borrow()` across `.await`. https://docs.rs/tokio/latest/tokio/sync/watch/index.html
- Dropping a `JoinHandle` detaches the task. Dropping a `JoinSet` aborts all its tasks. https://docs.rs/tokio/latest/tokio/task/struct.JoinSet.html
- `interval` ticks immediately first and bursts missed ticks by default. Set `MissedTickBehavior::Delay` or `Skip` when catch-up bursts are wrong. https://docs.rs/tokio/latest/tokio/time/enum.MissedTickBehavior.html
- `timeout` polls the future before checking the deadline, so a future that never yields can finish past the deadline with `Ok`. https://docs.rs/tokio/latest/tokio/time/fn.timeout.html
- Test timers with `#[tokio::test(start_paused = true)]` (feature `test-util`) and `sleep` with auto-advance rather than `advance`. https://docs.rs/tokio/latest/tokio/time/fn.advance.html
- Flush a `BufWriter` and a `tokio::fs::File` before drop, or data is lost or written late. https://docs.rs/tokio/latest/tokio/io/index.html and https://docs.rs/tokio/latest/tokio/fs/struct.File.html
- A `read()` returning `Ok(0)` means closed. A loop that ignores it spins at 100 % CPU. https://tokio.rs/tokio/tutorial/io
- Every `tokio::fs` call is one `spawn_blocking`. Batch with `fs::read` / `fs::write` or `BufWriter`. https://docs.rs/tokio/latest/tokio/fs/index.html
- A dropped `Child` keeps running unless `kill_on_drop(true)`. `output()` forces piped stdout and stderr. https://docs.rs/tokio/latest/tokio/process/index.html and https://docs.rs/tokio/latest/tokio/process/struct.Command.html#method.output
- On Unix, the first `ctrl_c()` listener replaces the default SIGINT handler for good. https://docs.rs/tokio/latest/tokio/signal/fn.ctrl_c.html
- A named pipe server must create the next instance before handing off the connected one, or clients get `NotFound`. A client retries only on `ERROR_PIPE_BUSY`. `NotFound` means no server, so return the error instead of retrying. https://docs.rs/tokio/latest/tokio/net/windows/named_pipe/struct.NamedPipeServer.html and https://docs.rs/tokio/latest/tokio/net/windows/named_pipe/struct.NamedPipeClient.html

## References

| File | Covers |
| :-- | :-- |
| [references/runtime.md](references/runtime.md) | Feature flags, Builder and drivers, LocalRuntime, shutdown, #[tokio::main] and #[tokio::test] |
| [references/block-on-bridging.md](references/block-on-bridging.md) | Driving a runtime, block_on and Handle::block_on, EnterGuard and Handle::current, calling async code from sync code, Unix fork |
| [references/tasks.md](references/tasks.md) | spawn and its bounds, JoinHandle, abort, JoinSet, spawn_blocking, block_in_place, coop budget, LocalSet, spawn_local, task_local |
| [references/sync-locks.md](references/sync-locks.md) | std vs tokio Mutex, guards across .await, RwLock, Semaphore, Notify, OnceCell, SetOnce, Barrier |
| [references/channels.md](references/channels.md) | mpsc backpressure, permits and closing, oneshot, broadcast lag, watch borrow rules |
| [references/select-cancellation.md](references/select-cancellation.md) | select! rules, cancel safety lists, pinning across loops, biased, join!, try_join!, graceful shutdown |
| [references/time-testing.md](references/time-testing.md) | sleep, interval and MissedTickBehavior, timeout, Instant, paused time, mocking I/O in tests |
| [references/io-net.md](references/io-net.md) | AsyncRead/AsyncWrite ext traits, BufWriter, split, io::copy, framing, TcpListener, TcpStream, TcpSocket, UdpSocket |
| [references/fs-process.md](references/fs-process.md) | tokio::fs and its blocking pool, File flush and drop, Command, Child, kill_on_drop, stdio pipes |
| [references/signals-named-pipes.md](references/signals-named-pipes.md) | ctrl_c, Unix signals, Windows console events, Windows named pipe server and client |

All pages read are listed in [SOURCES.md](SOURCES.md).
