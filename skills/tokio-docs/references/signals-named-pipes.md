# Signals and Windows named pipes

## Contents
- ctrl_c and Unix signals
- Windows console events
- Named pipe server
- Named pipe client
- Pipe options and readiness

## ctrl_c and Unix signals

- Signal handling is "a very tricky topic" in the docs. Use it with care. https://docs.rs/tokio/latest/tokio/signal/index.html
- Default portable handler: `tokio::signal::ctrl_c()`. It registers on first poll, so it catches only the Ctrl-C after that poll, not after the call. Create and await it early, for example in a spawned task. https://docs.rs/tokio/latest/tokio/signal/fn.ctrl_c.html

```rust
tokio::spawn(async move { tokio::signal::ctrl_c().await.unwrap(); /* handler */ });
```
https://docs.rs/tokio/latest/tokio/signal/fn.ctrl_c.html

- On Unix, the first listener replaces the default handler for the whole process, forever. After the listener is dropped, SIGINT no longer terminates the process, so the app must exit by itself. https://docs.rs/tokio/latest/tokio/signal/fn.ctrl_c.html
- Other Unix signals: `signal(SignalKind::hangup())` returns a stream, loop on `recv().await`. `Signal::recv` is cancel safe. https://docs.rs/tokio/latest/tokio/signal/index.html and https://docs.rs/tokio/latest/tokio/signal/unix/struct.Signal.html#method.recv

## Windows console events

- `tokio::signal::windows` exists only on Windows. `ctrl_c()`, `ctrl_break()`, `ctrl_close()` and the logoff and shutdown constructors each return a listener with `recv().await`. https://docs.rs/tokio/latest/tokio/signal/windows/index.html
- Every event reaches all listeners. Events not processed fast enough coalesce: 2 back-to-back events may give one item. https://docs.rs/tokio/latest/tokio/signal/windows/struct.CtrlC.html
- `recv` returns `Option<()>` but never `None`. https://docs.rs/tokio/latest/tokio/signal/windows/struct.CtrlC.html#method.recv

```rust
let mut signal = ctrl_close()?;
signal.recv().await;
println!("got CTRL-CLOSE. Cleaning up before exiting");
```
https://docs.rs/tokio/latest/tokio/signal/windows/fn.ctrl_close.html

## Named pipe server

- Keep at least one server instance available at all times, or clients fail with `ErrorKind::NotFound`. In the accept loop, create the next instance before handing the connected one to a task. https://docs.rs/tokio/latest/tokio/net/windows/named_pipe/struct.NamedPipeServer.html

```rust
let mut server = ServerOptions::new().first_pipe_instance(true).create(PIPE_NAME)?;
loop {
    server.connect().await?;
    let connected_client = server;
    server = ServerOptions::new().create(PIPE_NAME)?;
    tokio::spawn(async move { /* use the connected client */ });
}
```
https://docs.rs/tokio/latest/tokio/net/windows/named_pipe/struct.NamedPipeServer.html

- Set `first_pipe_instance(true)` on the first instance to be sure you are the only listener. If more than one instance of the pipe exists while the flag is set, `create` fails with `PermissionDenied`: only the first instance should set it. https://docs.rs/tokio/latest/tokio/net/windows/named_pipe/struct.ServerOptions.html#method.first_pipe_instance
- `connect()` is cancel safe. `disconnect()` forces the client off, and its next write fails with `ERROR_PIPE_NOT_CONNECTED`. https://docs.rs/tokio/latest/tokio/net/windows/named_pipe/struct.NamedPipeServer.html#method.connect and https://docs.rs/tokio/latest/tokio/net/windows/named_pipe/struct.NamedPipeServer.html#method.disconnect
- `create` and `open` fail outside a runtime or on one without I/O enabled. https://docs.rs/tokio/latest/tokio/net/windows/named_pipe/struct.ServerOptions.html#method.create

## Named pipe client

- `NotFound` on open means no server. `ERROR_PIPE_BUSY` means a server exists but is busy: sleep and retry. https://docs.rs/tokio/latest/tokio/net/windows/named_pipe/struct.NamedPipeClient.html

```rust
let client = loop {
    match ClientOptions::new().open(PIPE_NAME) {
        Ok(client) => break client,
        Err(e) if e.raw_os_error() == Some(ERROR_PIPE_BUSY as i32) => (),
        Err(e) => return Err(e),
    }
    time::sleep(Duration::from_millis(50)).await;
};
```
https://docs.rs/tokio/latest/tokio/net/windows/named_pipe/struct.NamedPipeClient.html

- `security_qos_flags` defaults to `SECURITY_IDENTIFICATION`, which stops a malicious pipe from taking the privileges of a privileged process that opens a user-supplied path. Setting it replaces the default entirely. https://docs.rs/tokio/latest/tokio/net/windows/named_pipe/struct.ClientOptions.html#method.security_qos_flags

## Pipe options and readiness

- Server defaults: byte mode, inbound and outbound access, `reject_remote_clients` true, unlimited instances, 65536-byte buffers. https://docs.rs/tokio/latest/tokio/net/windows/named_pipe/struct.ServerOptions.html#method.reject_remote_clients
- `max_instances` accepts 1 to 254 and panics above. Every server instance of a pipe must use the same value, or creation might fail (the docs example shows `ERROR_PIPE_BUSY`). https://docs.rs/tokio/latest/tokio/net/windows/named_pipe/struct.ServerOptions.html#method.max_instances
- With `access_inbound(false)` or `access_outbound(false)` on the server, the client must disable the matching `write(false)` / `read(false)`, or `open` fails with `PermissionDenied`. https://docs.rs/tokio/latest/tokio/net/windows/named_pipe/struct.ServerOptions.html#method.access_inbound
- Set `write_dac(true)` to change the pipe DACL later. https://docs.rs/tokio/latest/tokio/net/windows/named_pipe/struct.ServerOptions.html#method.write_dac
- `ready(interest)` can return a false positive or an empty `Ready`. Check it, and wait again when `try_read` / `try_write` return `WouldBlock`. https://docs.rs/tokio/latest/tokio/net/windows/named_pipe/struct.NamedPipeServer.html#method.ready
- Prefer `readable()` over `poll_read_ready`, which wakes only the last waker per direction. Use one interest per `try_io` closure. https://docs.rs/tokio/latest/tokio/net/windows/named_pipe/struct.NamedPipeServer.html#method.poll_read_ready and https://docs.rs/tokio/latest/tokio/net/windows/named_pipe/struct.NamedPipeServer.html#method.try_io
- `flush` on a pipe does nothing. https://docs.rs/tokio/latest/tokio/net/windows/named_pipe/struct.NamedPipeClient.html
