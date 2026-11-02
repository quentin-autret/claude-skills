# Runtime, feature flags and entry points

## Contents
- Feature flags
- Choosing and building a runtime
- Shutdown
- #[tokio::main] and #[tokio::test]
- Old patterns

## Feature flags

- Tokio enables no feature by default. Each item lists the feature it needs. https://docs.rs/tokio/latest/tokio/index.html#feature-flags
- Default for applications: `tokio = { version = "1", features = ["full"] }`. https://docs.rs/tokio/latest/tokio/index.html#authoring-applications
- Libraries enable only what they use, for example `features = ["rt", "net"]` for `tokio::spawn` plus `TcpStream`. https://docs.rs/tokio/latest/tokio/index.html#authoring-libraries
- `full` does not include `test-util` or unstable features. Add `test-util` explicitly for paused time and `start_paused`. https://docs.rs/tokio/latest/tokio/index.html#feature-flags
- `rt` gives `tokio::spawn`, the current-thread scheduler and non-scheduler utilities. `rt-multi-thread` gives the work-stealing scheduler, needed by `Runtime::new`, `Builder::new_multi_thread` and the default `#[tokio::main]` flavor. `macros` gives `#[tokio::main]`, `#[tokio::test]` and `select!`. `io-util` gives the `Ext` traits. `io-std` gives stdin/stdout/stderr. https://docs.rs/tokio/latest/tokio/index.html#feature-flags
- `#[tokio::main]` without `rt` or `rt-multi-thread` fails with "The #[tokio::main] macro requires rt or rt-multi-thread." https://docs.rs/tokio/latest/tokio/attr.main.html
- Unstable features (`tracing`, `io-uring`, `taskdump`, unstable APIs) need `--cfg tokio_unstable`, and their API may break in 1.x. Put it in `.cargo/config.toml`, never in `Cargo.toml`: https://docs.rs/tokio/latest/tokio/index.html#unstable-features

```toml
[build]
rustflags = ["--cfg", "tokio_unstable"]
```
https://docs.rs/tokio/latest/tokio/index.html#unstable-features

- All Wasm targets build with `sync`, `macros`, `io-util`, `rt` and `time`. Any other feature, `full` included, fails to compile (`wasm32-unknown-emscripten` allows more, see the page). Wasm support is experimental. https://docs.rs/tokio/latest/tokio/index.html#wasm-support

## Choosing and building a runtime

- Decision tree of the docs: work-stealing wanted, use the multi-thread runtime. Otherwise `LocalRuntime` for `!Send` futures, else the current-thread runtime. The tree is not exhaustive. https://docs.rs/tokio/latest/tokio/runtime/index.html#choose-your-runtime
- The multi-thread runtime starts one worker per CPU core. `TOKIO_WORKER_THREADS` or `Builder::worker_threads(n)` override it (the builder wins, 0 panics). The default is the number of cores available to the system. https://docs.rs/tokio/latest/tokio/runtime/struct.Builder.html#method.worker_threads
- Surprising default: a runtime built by hand with `Builder` has no resource driver. Networking and time types then fail. Call `enable_all()` (or `enable_io()` / `enable_time()`). https://docs.rs/tokio/latest/tokio/runtime/index.html#resource-drivers

```rust
let rt = tokio::runtime::Builder::new_current_thread().enable_all().build()?;
```
https://tokio.rs/tokio/topics/bridging#a-synchronous-interface-to-mini-redis

- `Runtime::new()` equals `Builder::new_multi_thread().enable_all().build()`. https://docs.rs/tokio/latest/tokio/runtime/struct.Runtime.html#method.new
- Use `#[tokio::main]` when no fine tuning is needed, `Builder` otherwise. https://docs.rs/tokio/latest/tokio/attr.main.html
- `start_paused(true)` needs the `test-util` feature and a current-thread runtime, otherwise construction panics. https://docs.rs/tokio/latest/tokio/runtime/struct.Builder.html#method.start_paused
- `on_thread_park` / `on_thread_unpark` hold one callback per runtime, a second call replaces the first. https://docs.rs/tokio/latest/tokio/runtime/struct.Builder.html#method.on_thread_park
- `max_blocking_threads` defaults to 512 and counts only the blocking pool. Its queue has no backpressure. `fs`, DNS via `ToSocketAddrs` and stdio also use this pool, so a low value can hang `spawn_blocking`. https://docs.rs/tokio/latest/tokio/runtime/struct.Builder.html#method.max_blocking_threads
- The runtime is not NUMA aware. Consider several runtimes on NUMA systems. https://docs.rs/tokio/latest/tokio/runtime/index.html#numa-awareness

### LocalRuntime

- `LocalRuntime` runs `!Send` tasks with `spawn_local`, no `LocalSet` needed. It is `!Send` and `!Sync`. https://docs.rs/tokio/latest/tokio/runtime/struct.LocalRuntime.html
- Never drive a `LocalSet` inside a `LocalRuntime`, they are incompatible. https://docs.rs/tokio/latest/tokio/runtime/struct.LocalRuntime.html
- Its `handle()` can move to other threads, so it only spawns `Send` tasks. https://docs.rs/tokio/latest/tokio/runtime/struct.LocalRuntime.html#method.handle
- Build it with `Builder::new_current_thread().enable_all().build_local(LocalOptions::default())` or `#[tokio::main(flavor = "local")]`. `build_local` panics on a `new_multi_thread()` builder. https://docs.rs/tokio/latest/tokio/attr.main.html and https://docs.rs/tokio/latest/tokio/runtime/struct.Builder.html#method.build_local

## Shutdown

- On drop, spawned tasks run until their next yield and are then dropped. They are not guaranteed to finish. Blocking tasks keep running until they return, and `Drop` waits for them forever. https://docs.rs/tokio/latest/tokio/runtime/struct.Runtime.html#shutdown
- Prefer `shutdown_timeout(d)` to bound the wait. On timeout, remaining work and its threads leak. https://docs.rs/tokio/latest/tokio/runtime/struct.Runtime.html#shutdown and https://docs.rs/tokio/latest/tokio/runtime/struct.Runtime.html#method.shutdown_timeout

```rust
runtime.shutdown_timeout(Duration::from_millis(100));
```
https://docs.rs/tokio/latest/tokio/runtime/struct.Runtime.html#method.shutdown_timeout

- Dropping a runtime inside another runtime's async context is not permitted because the drop blocks. Use `shutdown_background()` there. https://docs.rs/tokio/latest/tokio/runtime/struct.Runtime.html#method.shutdown_background
- After the runtime is dropped, its I/O resources return errors (detect with `tokio::runtime::is_rt_shutdown_err`) and timers panic. https://docs.rs/tokio/latest/tokio/runtime/fn.is_rt_shutdown_err.html and https://docs.rs/tokio/latest/tokio/runtime/struct.Handle.html#method.block_on

## #[tokio::main] and #[tokio::test]

- `#[tokio::main]` builds a new multi-thread runtime with `enable_all` on every call. It works on any function, but a function called often should reuse one runtime built with `Builder`. https://docs.rs/tokio/latest/tokio/attr.main.html and https://tokio.rs/tokio/topics/bridging#what-tokiomain-expands-to
- Options: `flavor = "multi_thread" | "current_thread" | "local"`, `worker_threads = N`, `start_paused = true` (needs `test-util`), `crate = "..."` for a renamed dependency. https://docs.rs/tokio/latest/tokio/attr.main.html
- `#[tokio::test]` defaults to a current-thread runtime per test, the opposite of `#[tokio::main]`. Ask for threads explicitly: https://docs.rs/tokio/latest/tokio/attr.test.html

```rust
#[tokio::test(flavor = "multi_thread", worker_threads = 1)]
async fn my_test() {}
```
https://docs.rs/tokio/latest/tokio/attr.test.html

## Old patterns

| Old | Replacement |
| :-- | :-- |
| `Builder::max_threads` (counted workers too) | `max_blocking_threads`, which counts only the blocking pool. https://docs.rs/tokio/latest/tokio/runtime/struct.Builder.html#method.max_blocking_threads |
| `enable_metrics_poll_count_histogram` (deprecated, hidden from docs) | `enable_metrics_poll_time_histogram`, unstable (`tokio_unstable`). https://docs.rs/tokio/latest/tokio/runtime/struct.Builder.html#method.enable_metrics_poll_time_histogram |
| `metrics_poll_count_histogram_scale` / `_resolution` / `_buckets` | `metrics_poll_time_histogram_configuration` (also unstable), with `LogHistogram::builder().precision_exact(0)` to match the legacy log histogram. https://docs.rs/tokio/latest/tokio/runtime/struct.Builder.html#method.metrics_poll_time_histogram_configuration |
