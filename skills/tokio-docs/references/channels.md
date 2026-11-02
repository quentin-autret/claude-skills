# Channels: mpsc, oneshot, broadcast, watch

## Contents
- Choosing a channel
- mpsc: capacity and backpressure
- mpsc: send, permits and cancel safety
- mpsc: closing and sync code
- oneshot
- broadcast
- watch

## Choosing a channel

- mpsc: many producers, one consumer, many values. oneshot: one value. broadcast: every receiver sees every value. watch: receivers see only the latest value (configuration, state). https://tokio.rs/tokio/tutorial/channels
- For many consumers where each message goes to only one of them, the tutorial points to the `async-channel` crate. https://tokio.rs/tokio/tutorial/channels
- `std::sync::mpsc` and `crossbeam::channel` block the thread to wait, which is not allowed in async code. https://tokio.rs/tokio/tutorial/channels

## mpsc: capacity and backpressure

- Default: the bounded `mpsc::channel(n)`. Once `n` messages are buffered, `send(..).await` waits for the receiver, which gives backpressure. https://docs.rs/tokio/latest/tokio/sync/mpsc/index.html
- Unbounded queues eventually fill memory and fail unpredictably. Pick a capacity and also bound total concurrency, for example open sockets in an accept loop. https://tokio.rs/tokio/tutorial/channels
- Escape hatch: `unbounded_channel` when the sender must not wait or must send from sync code (`UnboundedSender::send` is not async). Memory is the only bound and running out aborts the process. https://docs.rs/tokio/latest/tokio/sync/mpsc/fn.unbounded_channel.html
- `mpsc::channel(0)` panics. Capacity must be at least 1. https://docs.rs/tokio/latest/tokio/sync/mpsc/fn.channel.html
- For a single message use oneshot, not mpsc. https://docs.rs/tokio/latest/tokio/sync/mpsc/index.html

```rust
let (tx, mut rx) = mpsc::channel(32);
let tx2 = tx.clone();
while let Some(message) = rx.recv().await { println!("GOT = {}", message); }
```
https://tokio.rs/tokio/tutorial/channels

## mpsc: send, permits and cancel safety

- `recv` and `recv_many` are cancel safe in `select!`. https://docs.rs/tokio/latest/tokio/sync/mpsc/struct.Receiver.html#method.recv
- `send` in a `select!` branch that loses drops the message, which is lost. Default in `select!`: `reserve()` first, then send through the `Permit`. https://docs.rs/tokio/latest/tokio/sync/mpsc/struct.Sender.html#method.send
- Dropping a `Permit` unused releases its slot. `reserve_owned` returns an `OwnedPermit` that owns the `Sender`. https://docs.rs/tokio/latest/tokio/sync/mpsc/struct.Sender.html#method.reserve_owned
- `send` and `reserve` complete in request order. Cancelling one loses the place in the queue. https://docs.rs/tokio/latest/tokio/sync/mpsc/struct.Sender.html#method.reserve
- `Err` from `send` means the value will never be received and returns it. `Ok` does not guarantee it will be received. https://docs.rs/tokio/latest/tokio/sync/mpsc/struct.Sender.html#method.send
- `try_send` fails when the buffer is full or the receiver is closed. https://docs.rs/tokio/latest/tokio/sync/mpsc/struct.Sender.html#method.try_send
- `send_timeout` panics outside a runtime with time enabled. https://docs.rs/tokio/latest/tokio/sync/mpsc/struct.Sender.html#method.send_timeout

## mpsc: closing and sync code

- `recv` returns `None` once every `Sender` is dropped and the buffer is drained. Drop the original `tx` too, or the loop never ends. https://docs.rs/tokio/latest/tokio/sync/mpsc/index.html
- Clean shutdown: the receiver calls `close()`, then keeps calling `recv()` until `None`. If permits from before `close()` are outstanding, `recv` does not report the channel closed until they are released. https://docs.rs/tokio/latest/tokio/sync/mpsc/struct.Receiver.html#method.close
- `WeakSender` does not keep the channel open. https://docs.rs/tokio/latest/tokio/sync/mpsc/struct.WeakSender.html
- From sync code on a bounded channel, call `blocking_send` / `blocking_recv`. They panic in an async context. https://docs.rs/tokio/latest/tokio/sync/mpsc/struct.Receiver.html#method.blocking_recv
- Unbounded and sync to async: Tokio unbounded mpsc. Unbounded and async to sync: std or crossbeam unbounded channel. https://docs.rs/tokio/latest/tokio/sync/mpsc/index.html
- A channel method not marked `async` can be called anywhere, outside the runtime included. https://docs.rs/tokio/latest/tokio/sync/mpsc/index.html
- `ReceiverStream` (tokio-stream) turns a `Receiver` into a `Stream`. https://docs.rs/tokio/latest/tokio/sync/mpsc/struct.Receiver.html

## oneshot

- `Sender::send` is not async, consumes the sender and returns the value back in `Err` if the receiver is gone. `Ok` does not mean it will be received. https://docs.rs/tokio/latest/tokio/sync/oneshot/struct.Sender.html#method.send
- There is no `recv`. Await the `Receiver`, which yields `Result<T, RecvError>`. A sender dropped without sending gives `RecvError`. https://docs.rs/tokio/latest/tokio/sync/oneshot/struct.Receiver.html
- In a `select!` loop await `&mut recv`, which is cancel safe. https://docs.rs/tokio/latest/tokio/sync/oneshot/struct.Receiver.html

```rust
let (send, mut recv) = oneshot::channel();
loop {
    tokio::select! {
        _ = interval.tick() => println!("Another 100ms"),
        msg = &mut recv => { println!("Got message: {}", msg.unwrap()); break; }
    }
}
```
https://docs.rs/tokio/latest/tokio/sync/oneshot/index.html

- `Sender::closed().await` completes when the receiver closes or drops. Select on it to stop work nobody awaits. https://docs.rs/tokio/latest/tokio/sync/oneshot/struct.Sender.html#method.closed
- To send from a destructor, store the sender in an `Option` and `take()` it. https://docs.rs/tokio/latest/tokio/sync/oneshot/index.html
- `try_recv` returns `Empty` or `Closed` and does not register for wakeup. https://docs.rs/tokio/latest/tokio/sync/oneshot/struct.Receiver.html#method.try_recv

## broadcast

- `T: Clone` is required. Capacity is rounded up to the next power of two (`channel(3)` holds 4), and 0 panics. https://docs.rs/tokio/latest/tokio/sync/broadcast/index.html#lagging and https://docs.rs/tokio/latest/tokio/sync/broadcast/fn.channel.html
- A receiver from `Sender::subscribe` sees only values sent after the call. https://docs.rs/tokio/latest/tokio/sync/broadcast/struct.Sender.html#method.subscribe
- When full, the oldest value is overwritten. A slow receiver then gets `RecvError::Lagged(n)`. It stays open and the next `recv` returns the oldest retained value. Handle `Lagged` explicitly: resume or stop the task. https://docs.rs/tokio/latest/tokio/sync/broadcast/struct.Receiver.html#method.recv

```rust
loop {
    match rx.recv().await {
        Ok(v) => handle(v),
        Err(RecvError::Lagged(n)) => { /* n values missed, keep going */ }
        Err(RecvError::Closed) => break,
    }
}
```
Sketch of the documented behavior, not a docs example. https://docs.rs/tokio/latest/tokio/sync/broadcast/struct.Receiver.html#method.recv

- `recv` returns `RecvError::Closed` once all senders are dropped and retained values are read. https://docs.rs/tokio/latest/tokio/sync/broadcast/index.html#closing
- `Sender::send` returns `Ok(receiver count)`, not a delivery guarantee. `Err` (no receivers) does not mean later sends fail. https://docs.rs/tokio/latest/tokio/sync/broadcast/struct.Sender.html#method.send
- `recv` is cancel safe. https://docs.rs/tokio/latest/tokio/sync/broadcast/struct.Receiver.html#method.recv

## watch

- The initial value counts as seen. `changed()` returns immediately if the current value is unseen, otherwise waits, and errors only when the channel is closed and the value is seen. https://docs.rs/tokio/latest/tokio/sync/watch/index.html#change-notifications
- In a `changed()` loop use `borrow_and_update()`, not `borrow()`, or a value sent between the two calls makes the loop run twice on the same value. https://docs.rs/tokio/latest/tokio/sync/watch/index.html

```rust
loop {
    println!("{}! ", *rx.borrow_and_update());
    if rx.changed().await.is_err() { break; }
}
```
https://docs.rs/tokio/latest/tokio/sync/watch/index.html

- A `Ref` from `borrow` holds a read lock. Keep it short, do not hold it across `.await` (deadlock risk with `!Send` futures), and avoid 2 borrows at once in one task (may deadlock when a producer waits on the lock). https://docs.rs/tokio/latest/tokio/sync/watch/struct.Receiver.html#method.borrow
- `wait_for(|v| pred)` checks the current value first and may skip intermediate values. It is cancel safe. https://docs.rs/tokio/latest/tokio/sync/watch/struct.Receiver.html#method.wait_for
- `Sender::send` fails when no receiver exists and then does not store the value. To always store it, use `send_replace`, `send_modify` or `send_if_modified`. Prefer `send_if_modified` for conditional changes. https://docs.rs/tokio/latest/tokio/sync/watch/struct.Sender.html#method.send and https://docs.rs/tokio/latest/tokio/sync/watch/struct.Sender.html#method.send_modify
