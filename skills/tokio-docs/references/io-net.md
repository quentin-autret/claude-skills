# I/O and networking

## Contents
- Reading and writing
- Buffering
- Splitting a stream
- io::copy
- Framing
- TcpListener
- TcpStream and TcpSocket
- UdpSocket
- Old patterns

## Reading and writing

- Import `AsyncReadExt` / `AsyncWriteExt` (feature `io-util`) for `read`, `write_all` and the other methods. The feature "io-util" enables these `Ext` traits. https://docs.rs/tokio/latest/tokio/index.html
- `read()` returning `Ok(0)` means the stream is closed, and every later call returns `Ok(0)` at once. A read loop that does not exit on `Ok(0)` spins at 100 % CPU. https://tokio.rs/tokio/tutorial/io
- `write()` may write only part of the buffer. Default: `write_all`. https://tokio.rs/tokio/tutorial/io
- Allocate a per-connection buffer on the heap (`vec![0; 1024]`). A stack array held across `.await` is stored inside the task and makes it large. https://tokio.rs/tokio/tutorial/io
- `AsyncRead` / `AsyncWrite` implementations must never block the thread. https://docs.rs/tokio/latest/tokio/io/index.html
- Async stdin, stdout and stderr panic outside a runtime. https://docs.rs/tokio/latest/tokio/io/index.html

## Buffering

- `BufWriter` discards buffered data on drop unless flushed or shut down. Always `flush().await` it. https://docs.rs/tokio/latest/tokio/io/index.html

```rust
let mut writer = BufWriter::new(f);
writer.write(&[42u8]).await?;
writer.flush().await?;
```
https://docs.rs/tokio/latest/tokio/io/index.html

- Wrap a `TcpStream` in `BufWriter` before many small writes such as `write_u8`. https://tokio.rs/tokio/tutorial/framing
- `BufReader` with `AsyncBufReadExt` adds `read_line` and cuts system calls. https://docs.rs/tokio/latest/tokio/io/index.html
- Stream conversions live in tokio-util: `ReaderStream` / `StreamReader`, and `codec` (`Decoder`, `Encoder`) for structured frames. https://docs.rs/tokio/latest/tokio/io/index.html

## Splitting a stream

- `io::copy(&mut socket, &mut socket)` does not compile. Split the stream. https://tokio.rs/tokio/tutorial/io
- Default for `TcpStream`: `split()` (borrowed, zero cost, both halves on the same task) or `into_split()` (owned halves for different tasks, one `Arc`). https://tokio.rs/tokio/tutorial/io
- Escape hatch for any `AsyncRead + AsyncWrite`: `io::split`, which costs an `Arc` and a `Mutex`. https://docs.rs/tokio/latest/tokio/io/fn.split.html
- Dropping the halves of `io::split` does not close the connection. Call `shutdown()` on the write half to send EOF. https://tokio.rs/tokio/tutorial/io

```rust
let (mut rd, mut wr) = io::split(socket);
tokio::spawn(async move {
    wr.write_all(b"hello\r\n").await?;
    wr.shutdown().await?;
    Ok::<_, io::Error>(())
});
```
https://tokio.rs/tokio/tutorial/io

- `unsplit` panics when the halves come from different streams. Check with `is_pair_of`. https://docs.rs/tokio/latest/tokio/io/struct.ReadHalf.html#method.unsplit

## io::copy

- `io::copy` copies until EOF with an 8 KB heap buffer and returns the byte count. For another size, use `copy_buf` with a `BufReader` of that capacity. https://docs.rs/tokio/latest/tokio/io/fn.copy.html
- `copy` retries `ErrorKind::Interrupted` and returns any other error immediately. https://docs.rs/tokio/latest/tokio/io/fn.copy.html
- Prefer async alternatives to wrapping a reader in `SyncIoBridge` to feed a sync consumer (hasher, compressor). https://docs.rs/tokio/latest/tokio/io/fn.copy.html

## Framing

- One `read()` can return a whole frame, part of one or several. Buffer the bytes, parse once a frame is complete, keep the rest. https://tokio.rs/tokio/tutorial/framing
- Default buffer: `BytesMut` with `read_buf`, which advances the cursor itself and skips the zero-fill of `vec![0; 4096]`. https://tokio.rs/tokio/tutorial/framing
- When `read_buf` returns 0: an empty buffer is a clean close (`Ok(None)`), a non-empty one is a reset mid-frame (error). https://tokio.rs/tokio/tutorial/framing
- Parse in 2 steps: check that a full frame is buffered, then parse and `buffer.advance(len)`. https://tokio.rs/tokio/tutorial/framing
- Handle the result of `flush().await` (the tutorial's `write_frame` ignores it). https://tokio.rs/tokio/tutorial/framing

## TcpListener

- Accept errors are not always fatal (too many open files, peer gone while queued). Handle them instead of ending the accept loop on `?`. https://docs.rs/tokio/latest/tokio/net/struct.TcpListener.html
- `accept()` is cancel safe in `select!`. https://docs.rs/tokio/latest/tokio/net/struct.TcpListener.html#method.accept
- `bind` sets `SO_REUSEADDR` on Unix. Port 0 lets the OS choose, read it with `local_addr()`. https://docs.rs/tokio/latest/tokio/net/struct.TcpListener.html#method.bind
- `from_std` requires `set_nonblocking(true)` first. A blocking listener is always wrong. It panics outside a runtime with I/O enabled. https://docs.rs/tokio/latest/tokio/net/struct.TcpListener.html#method.from_std
- `TcpListenerStream` (tokio-stream) makes the listener a stream. https://docs.rs/tokio/latest/tokio/net/struct.TcpListener.html

## TcpStream and TcpSocket

- `connect` tries each resolved address and returns the last error. On a host without IPv6 an `AddrNotAvailable` can hide an IPv4 "connection refused". https://docs.rs/tokio/latest/tokio/net/struct.TcpStream.html#method.connect
- `shutdown()` closes only the write direction. https://docs.rs/tokio/latest/tokio/net/struct.TcpStream.html
- `readable()` / `writable()` / `ready()` can report false positives. After them, `try_read` / `try_write` may return `WouldBlock`, and you must wait again. https://docs.rs/tokio/latest/tokio/net/struct.TcpStream.html#method.readable
- In `try_io`, return `WouldBlock` only when the system call returned it, use one interest, and do not call Tokio stream methods inside the closure. https://docs.rs/tokio/latest/tokio/net/struct.TcpStream.html#method.try_io
- `from_std` has the same non-blocking requirement as the listener. https://docs.rs/tokio/latest/tokio/net/struct.TcpStream.html#method.from_std
- Use `TcpSocket` only when the defaults of `connect` / `bind` do not fit. For options it does not expose, use `socket2` on the raw socket. https://docs.rs/tokio/latest/tokio/net/struct.TcpSocket.html
- On Windows `SO_REUSEADDR` allows hijacking a socket in use, so the documented bind equivalent does not set it there. `set_reuseport` exists only on Unix (not Solaris, illumos, Cygwin or NuttX). https://docs.rs/tokio/latest/tokio/net/struct.TcpSocket.html and https://docs.rs/tokio/latest/tokio/net/struct.TcpSocket.html#method.set_reuseport
- Buffer sizes read back may differ from those set (the OS clamps, Linux doubles). https://docs.rs/tokio/latest/tokio/net/struct.TcpSocket.html#method.send_buffer_size

## UdpSocket

- `bind` with `send_to` / `recv_from` talks to many peers. `connect` with `send` / `recv` talks to one, and `send` / `recv` fail on an unconnected socket. https://docs.rs/tokio/latest/tokio/net/struct.UdpSocket.html#method.send
- Share it as `Arc<UdpSocket>` with no `Mutex`, every method takes `&self`. https://docs.rs/tokio/latest/tokio/net/struct.UdpSocket.html
- The source address of a datagram can be spoofed. Do not trust it in the protocol. https://docs.rs/tokio/latest/tokio/net/struct.UdpSocket.html#method.recv_from
- On Windows, `peek` with a buffer smaller than the datagram loses the excess and fails with `WSAEMSGSIZE`. Use a 65536-byte buffer. https://docs.rs/tokio/latest/tokio/net/struct.UdpSocket.html#method.peek
- `send`, `send_to`, `recv` and `recv_from` are cancel safe. https://docs.rs/tokio/latest/tokio/net/struct.UdpSocket.html#method.recv_from
- For resources `tokio::net` does not cover, use `AsyncFd` (`tokio::io::unix::AsyncFd`). https://docs.rs/tokio/latest/tokio/net/index.html

## Old patterns

| Old | Replacement |
| :-- | :-- |
| `set_linger` (deprecated, blocks the thread on close) | `shutdown()` then read until EOF. `set_zero_linger` does not block but drops unsent data. https://docs.rs/tokio/latest/tokio/net/struct.TcpStream.html#method.set_linger and https://docs.rs/tokio/latest/tokio/net/struct.TcpSocket.html#method.set_zero_linger |
| `TcpSocket::tos` / `set_tos` (deprecated) | `tos_v4` / `set_tos_v4`. https://docs.rs/tokio/latest/tokio/net/struct.TcpSocket.html#method.tos_v4 |
