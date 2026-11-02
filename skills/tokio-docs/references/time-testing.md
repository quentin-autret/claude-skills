# Time and testing

## Contents
- Timer context
- sleep
- interval and MissedTickBehavior
- timeout
- Instant
- Paused time in tests
- Mocking I/O

## Timer context

- `sleep`, `sleep_until`, `timeout` and `timeout_at` panic without a current timer: the runtime was built without `enable_time` / `enable_all`, or the call happens outside a runtime. https://docs.rs/tokio/latest/tokio/time/fn.sleep.html
- `rt.block_on(sleep(d))` panics because `sleep(d)` is created outside the runtime. Wrap it: `rt.block_on(async { sleep(d).await })`. https://docs.rs/tokio/latest/tokio/time/fn.sleep.html

## sleep

- `Sleep` has millisecond granularity, coarser on Windows. It is not a high-resolution timer. https://docs.rs/tokio/latest/tokio/time/fn.sleep.html
- `Sleep` is `!Unpin`. Pin it with `tokio::pin!` or `Box::pin` before using it in `select!` or `poll`. A plain `.await` needs no pinning. https://docs.rs/tokio/latest/tokio/time/struct.Sleep.html
- Default to reuse a timer: `reset` the pinned sleep instead of creating a new one. https://docs.rs/tokio/latest/tokio/time/struct.Sleep.html#method.reset

```rust
let sleep = time::sleep(Duration::from_millis(10));
tokio::pin!(sleep);
loop {
    tokio::select! {
        () = &mut sleep => {
            sleep.as_mut().reset(Instant::now() + Duration::from_millis(50));
        },
    }
}
```
https://docs.rs/tokio/latest/tokio/time/struct.Sleep.html

- In a struct, store `Pin<Box<Sleep>>` to keep the struct `Unpin`, or use pin projection. https://docs.rs/tokio/latest/tokio/time/struct.Sleep.html

## interval and MissedTickBehavior

- Default for periodic work: `interval`, not `sleep` in a loop. An interval measures from the last tick, so a 2 s interval with a 1 s job runs every 2 s, while `sleep(2s)` after the job runs every 3 s. https://docs.rs/tokio/latest/tokio/time/fn.interval.html
- The first tick completes immediately. Use `interval_at(start, period)` to delay it. A zero period panics. https://docs.rs/tokio/latest/tokio/time/fn.interval.html
- Surprising default: `MissedTickBehavior::Burst` fires missed ticks back to back until caught up. Set `Delay` (keeps the period between ticks, schedule drifts) or `Skip` (jumps to the next multiple of the period). https://docs.rs/tokio/latest/tokio/time/enum.MissedTickBehavior.html

```rust
let mut interval = interval(Duration::from_millis(50));
interval.set_missed_tick_behavior(MissedTickBehavior::Delay);
```
https://docs.rs/tokio/latest/tokio/time/enum.MissedTickBehavior.html

- Missed-tick handling only applies when the delay exceeds 5 ms. https://docs.rs/tokio/latest/tokio/time/enum.MissedTickBehavior.html
- `tick()` is cancel safe in `select!`. https://docs.rs/tokio/latest/tokio/time/struct.Interval.html#method.tick
- `reset()`, `reset_immediately()` and `reset_after(d)` ignore `MissedTickBehavior`. `reset_at(t)` applies it when `t` is in the past. https://docs.rs/tokio/latest/tokio/time/struct.Interval.html#method.reset_at
- `IntervalStream` (tokio-stream) turns an `Interval` into a stream. https://docs.rs/tokio-stream/latest/tokio_stream/wrappers/struct.IntervalStream.html (linked from https://docs.rs/tokio/latest/tokio/time/struct.Interval.html)

## timeout

- `timeout(d, fut)` returns `Result<T, Elapsed>` and drops the future on expiry. https://docs.rs/tokio/latest/tokio/time/fn.timeout.html
- The future is polled before the deadline is checked. A future that never yields can finish past the deadline with `Ok`, and an immediately ready future always gives `Ok`. https://docs.rs/tokio/latest/tokio/time/fn.timeout.html
- `Timeout::into_inner` recovers the wrapped future. `timeout_at` takes an absolute `Instant`. https://docs.rs/tokio/latest/tokio/time/fn.timeout.html and https://docs.rs/tokio/latest/tokio/time/fn.timeout_at.html

## Instant

- Default in async code: `tokio::time::Instant`, which follows the Tokio clock and honors `pause()` and `advance()`. `std::time::Instant` keeps running when time is paused. https://docs.rs/tokio/latest/tokio/time/struct.Instant.html and https://docs.rs/tokio/latest/tokio/time/fn.pause.html
- `duration_since` and `elapsed` saturate to zero. `checked_duration_since` returns `None` when the other instant is later. https://docs.rs/tokio/latest/tokio/time/struct.Instant.html#method.checked_duration_since

## Paused time in tests

- `pause`, `resume`, `advance` and `start_paused` need the `test-util` feature and a current-thread runtime (the `#[tokio::test]` default). `pause()` panics if time is already paused or on another runtime flavor. https://docs.rs/tokio/latest/tokio/time/fn.pause.html
- Default: `#[tokio::test(start_paused = true)]` for tests paused from the start. https://tokio.rs/tokio/topics/testing
- With time paused and no other work ready, the clock jumps to the next timer. A 500 ms `sleep` then takes about 0 ms, and timers still fire in time order. https://tokio.rs/tokio/topics/testing and https://docs.rs/tokio/latest/tokio/time/fn.pause.html#auto-advance
- To force a timeout reliably, `sleep(d).await` with auto-advance rather than `advance(d)`. `advance` completes all timers due in the jump at once, in any order, and a `sleep` may not be ready on its first poll after it. https://docs.rs/tokio/latest/tokio/time/fn.advance.html

```rust
#[tokio::test(start_paused = true)]
async fn test_timeout_reliable() {
    time::sleep(Duration::from_secs(5)).await;
}
```
https://docs.rs/tokio/latest/tokio/time/fn.advance.html

- `advance` panics when time is not paused or when the duration overflows. Avoid `Duration::MAX`. https://docs.rs/tokio/latest/tokio/time/fn.advance.html
- Real I/O under paused time: run it in `spawn_blocking`, the clock does not auto-advance while a blocking task runs. https://docs.rs/tokio/latest/tokio/time/fn.pause.html#preventing-auto-advance

```rust
#[tokio::test(start_paused = true)]
async fn test_with_io() {
    let start = Instant::now();
    let result = task::spawn_blocking(|| { std::thread::sleep(std::time::Duration::from_millis(10)); 42 }).await.unwrap();
    assert_eq!(start.elapsed(), Duration::ZERO);
}
```
https://docs.rs/tokio/latest/tokio/time/fn.pause.html#preventing-auto-advance

## Mocking I/O

- Make handlers generic over `AsyncRead + Unpin` and `AsyncWrite + Unpin` instead of taking a `TcpStream`, then test them with `tokio_test::io::Builder` (`.read(..)`, `.write(..)`, `.build()`). `tokio_test` is a separate crate. https://tokio.rs/tokio/topics/testing

```rust
async fn handle_connection<Reader, Writer>(reader: Reader, mut writer: Writer) -> std::io::Result<()>
where Reader: AsyncRead + Unpin, Writer: AsyncWrite + Unpin
```
https://tokio.rs/tokio/topics/testing

- `Vec<u8>` and `&[u8]` implement `AsyncRead` / `AsyncWrite`, so byte buffers can stand in for a stream. https://tokio.rs/tokio/tutorial/io
