# block_on, Handle and sync code bridging

## Contents
- Driving the runtime and block_on
- Handle and runtime context
- Calling async code from sync code
- Unix fork

## Driving the runtime and block_on

- A current-thread runtime runs tasks only while `Runtime::block_on` executes. Once `block_on` returns, spawned tasks freeze until the next call. If they must keep running, use the multi-thread runtime. https://docs.rs/tokio/latest/tokio/runtime/index.html#driving-the-runtime and https://tokio.rs/tokio/topics/bridging#a-synchronous-interface-to-mini-redis
- `Handle::block_on` does not drive a current-thread runtime. I/O and timers do not work through it unless another thread is in `Runtime::block_on` on that runtime. https://docs.rs/tokio/latest/tokio/runtime/struct.Handle.html#method.block_on
- `rt.block_on(sleep(d))` panics because `sleep(d)` is created outside the runtime, before `block_on` enters it. Wrap the call in an async block: `rt.block_on(async { sleep(d).await })`. https://docs.rs/tokio/latest/tokio/time/fn.sleep.html
- `block_on` panics when called inside an async context (inside `block_on` or a `#[tokio::main]` function). https://docs.rs/tokio/latest/tokio/runtime/struct.Handle.html#method.block_on
- The future given to `Runtime::block_on` is not a worker task. It should mostly spawn tasks, since awaiting other futures directly in it is not as fast as in spawned tasks. https://docs.rs/tokio/latest/tokio/runtime/struct.Runtime.html#method.block_on
- To run async code from a multi-thread worker that must block, combine `block_in_place` with `Handle::current().block_on`: https://docs.rs/tokio/latest/tokio/runtime/struct.Handle.html#method.block_on

```rust
task::block_in_place(move || {
    Handle::current().block_on(async move { /* async work */ });
});
```
https://docs.rs/tokio/latest/tokio/runtime/struct.Handle.html#method.block_on

## Handle and runtime context

- An `Arc<Runtime>` keeps the runtime alive. A `Handle` does not prevent shutdown. https://docs.rs/tokio/latest/tokio/runtime/struct.Runtime.html#sharing
- `Sleep`, `TcpStream` and `tokio::spawn` need a runtime context. `Runtime::enter()` / `Handle::enter()` return an `EnterGuard` that sets it (the docs example notes it would panic without `rt.enter`). `Handle::enter` also makes `Handle::current` callable on that thread. https://docs.rs/tokio/latest/tokio/runtime/struct.Runtime.html#method.enter and https://docs.rs/tokio/latest/tokio/runtime/struct.Handle.html#method.enter
- Guards from several `Handle::enter` calls must be dropped in reverse order, otherwise it panics and may leak memory. https://docs.rs/tokio/latest/tokio/runtime/struct.Handle.html#method.enter
- `Handle::current()` panics outside a runtime context, for example in a `std::thread::spawn` thread without an active `EnterGuard`. https://docs.rs/tokio/latest/tokio/runtime/struct.Handle.html#method.current
- `Handle::try_current()` returns `Err(TryCurrentError)` instead of panicking. https://docs.rs/tokio/latest/tokio/runtime/struct.Handle.html#method.try_current
- To use the runtime from a `std::thread`, pass it `Handle::current()` and call `handle.block_on(...)` there. https://docs.rs/tokio/latest/tokio/runtime/struct.Handle.html#method.block_on

## Calling async code from sync code

- First approach in the tutorial for a sync wrapper around an async client: store a current-thread runtime built with `enable_all` next to the client, and call `self.rt.block_on(self.inner.get(key))`. https://tokio.rs/tokio/topics/bridging#a-synchronous-interface-to-mini-redis
- Third approach, the most flexible one: run the runtime on its own thread and send it messages. Build the runtime before `std::thread::spawn` so build errors surface, then use `blocking_send` from sync code. https://tokio.rs/tokio/topics/bridging#sending-messages

```rust
let (send, mut recv) = mpsc::channel(16);
let rt = Builder::new_current_thread().enable_all().build().unwrap();
std::thread::spawn(move || {
    rt.block_on(async move {
        while let Some(task) = recv.recv().await {
            tokio::spawn(handle_task(task));
        }
    });
});
```
https://tokio.rs/tokio/topics/bridging#sending-messages

- To spawn onto a runtime from sync code, use a multi-thread runtime (for example `worker_threads(1)`). On a current-thread runtime the spawned tasks only run during `block_on`. https://tokio.rs/tokio/topics/bridging#spawning-things-on-a-runtime

## Unix fork

- Code that calls `fork(2)` without an immediate `exec` must not reuse Tokio in the child. Supported only if the fork happens before the parent used Tokio, or the child never uses Tokio. A fresh runtime in the child after the parent used Tokio is not supported. https://docs.rs/tokio/latest/tokio/runtime/index.html#unix-fork
