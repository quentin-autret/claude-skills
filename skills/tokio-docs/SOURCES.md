# Sources

tokio 1.54.1, docs read on 2026-11-02.

- Docs: https://docs.rs/tokio/latest/tokio/ (resolves to 1.54.1)
- Raw rustdoc sources: https://raw.githubusercontent.com/tokio-rs/tokio/tokio-1.54.1/
- Raw tutorial and topics pages: https://raw.githubusercontent.com/tokio-rs/website/master/

## crate-and-runtime

| Docs page | Raw source |
| :-- | :-- |
| https://docs.rs/tokio/latest/tokio/index.html | https://raw.githubusercontent.com/tokio-rs/tokio/tokio-1.54.1/tokio/src/lib.rs |
| https://docs.rs/tokio/latest/tokio/runtime/index.html | https://raw.githubusercontent.com/tokio-rs/tokio/tokio-1.54.1/tokio/src/runtime/mod.rs |
| https://docs.rs/tokio/latest/tokio/runtime/struct.Builder.html | https://raw.githubusercontent.com/tokio-rs/tokio/tokio-1.54.1/tokio/src/runtime/builder.rs |
| https://docs.rs/tokio/latest/tokio/runtime/struct.Runtime.html | https://raw.githubusercontent.com/tokio-rs/tokio/tokio-1.54.1/tokio/src/runtime/runtime.rs |
| https://docs.rs/tokio/latest/tokio/runtime/struct.Handle.html | https://raw.githubusercontent.com/tokio-rs/tokio/tokio-1.54.1/tokio/src/runtime/handle.rs |
| https://docs.rs/tokio/latest/tokio/runtime/struct.LocalRuntime.html | https://raw.githubusercontent.com/tokio-rs/tokio/tokio-1.54.1/tokio/src/runtime/local_runtime/runtime.rs |
| https://docs.rs/tokio/latest/tokio/attr.main.html | https://raw.githubusercontent.com/tokio-rs/tokio/tokio-1.54.1/tokio-macros/src/lib.rs |
| https://tokio.rs/tokio/topics/bridging | https://raw.githubusercontent.com/tokio-rs/website/master/content/tokio/topics/bridging.md |

## tasks

| Docs page | Raw source |
| :-- | :-- |
| https://docs.rs/tokio/latest/tokio/task/index.html | https://raw.githubusercontent.com/tokio-rs/tokio/tokio-1.54.1/tokio/src/task/mod.rs |
| https://docs.rs/tokio/latest/tokio/task/fn.spawn.html | https://raw.githubusercontent.com/tokio-rs/tokio/tokio-1.54.1/tokio/src/task/spawn.rs |
| https://docs.rs/tokio/latest/tokio/task/fn.spawn_blocking.html | https://raw.githubusercontent.com/tokio-rs/tokio/tokio-1.54.1/tokio/src/task/blocking.rs |
| https://docs.rs/tokio/latest/tokio/task/struct.JoinSet.html | https://raw.githubusercontent.com/tokio-rs/tokio/tokio-1.54.1/tokio/src/task/join_set.rs |
| https://docs.rs/tokio/latest/tokio/task/struct.JoinHandle.html | https://raw.githubusercontent.com/tokio-rs/tokio/tokio-1.54.1/tokio/src/runtime/task/join.rs |
| https://docs.rs/tokio/latest/tokio/task/struct.LocalSet.html | https://raw.githubusercontent.com/tokio-rs/tokio/tokio-1.54.1/tokio/src/task/local.rs |
| https://docs.rs/tokio/latest/tokio/task/struct.Builder.html | https://raw.githubusercontent.com/tokio-rs/tokio/tokio-1.54.1/tokio/src/task/builder.rs |
| https://docs.rs/tokio/latest/tokio/macro.task_local.html | https://raw.githubusercontent.com/tokio-rs/tokio/tokio-1.54.1/tokio/src/task/task_local.rs |
| https://docs.rs/tokio/latest/tokio/task/fn.yield_now.html | https://raw.githubusercontent.com/tokio-rs/tokio/tokio-1.54.1/tokio/src/task/yield_now.rs |
| https://docs.rs/tokio/latest/tokio/task/coop/index.html | https://raw.githubusercontent.com/tokio-rs/tokio/tokio-1.54.1/tokio/src/task/coop/mod.rs |
| https://tokio.rs/tokio/tutorial/spawning | https://raw.githubusercontent.com/tokio-rs/website/master/content/tokio/tutorial/spawning.md |

## sync-locks

| Docs page | Raw source |
| :-- | :-- |
| https://docs.rs/tokio/latest/tokio/sync/index.html | https://raw.githubusercontent.com/tokio-rs/tokio/tokio-1.54.1/tokio/src/sync/mod.rs |
| https://docs.rs/tokio/latest/tokio/sync/struct.Mutex.html | https://raw.githubusercontent.com/tokio-rs/tokio/tokio-1.54.1/tokio/src/sync/mutex.rs |
| https://docs.rs/tokio/latest/tokio/sync/struct.RwLock.html | https://raw.githubusercontent.com/tokio-rs/tokio/tokio-1.54.1/tokio/src/sync/rwlock.rs |
| https://docs.rs/tokio/latest/tokio/sync/struct.Semaphore.html | https://raw.githubusercontent.com/tokio-rs/tokio/tokio-1.54.1/tokio/src/sync/semaphore.rs |
| https://docs.rs/tokio/latest/tokio/sync/struct.Notify.html | https://raw.githubusercontent.com/tokio-rs/tokio/tokio-1.54.1/tokio/src/sync/notify.rs |
| https://docs.rs/tokio/latest/tokio/sync/struct.OnceCell.html | https://raw.githubusercontent.com/tokio-rs/tokio/tokio-1.54.1/tokio/src/sync/once_cell.rs |
| https://docs.rs/tokio/latest/tokio/sync/struct.SetOnce.html | https://raw.githubusercontent.com/tokio-rs/tokio/tokio-1.54.1/tokio/src/sync/set_once.rs |
| https://docs.rs/tokio/latest/tokio/sync/struct.Barrier.html | https://raw.githubusercontent.com/tokio-rs/tokio/tokio-1.54.1/tokio/src/sync/barrier.rs |
| https://tokio.rs/tokio/tutorial/shared-state | https://raw.githubusercontent.com/tokio-rs/website/master/content/tokio/tutorial/shared-state.md |

## sync-channels

| Docs page | Raw source |
| :-- | :-- |
| https://docs.rs/tokio/latest/tokio/sync/mpsc/index.html | https://raw.githubusercontent.com/tokio-rs/tokio/tokio-1.54.1/tokio/src/sync/mpsc/mod.rs |
| https://docs.rs/tokio/latest/tokio/sync/mpsc/struct.Sender.html | https://raw.githubusercontent.com/tokio-rs/tokio/tokio-1.54.1/tokio/src/sync/mpsc/bounded.rs |
| https://docs.rs/tokio/latest/tokio/sync/mpsc/struct.UnboundedSender.html | https://raw.githubusercontent.com/tokio-rs/tokio/tokio-1.54.1/tokio/src/sync/mpsc/unbounded.rs |
| https://docs.rs/tokio/latest/tokio/sync/oneshot/index.html | https://raw.githubusercontent.com/tokio-rs/tokio/tokio-1.54.1/tokio/src/sync/oneshot.rs |
| https://docs.rs/tokio/latest/tokio/sync/broadcast/index.html | https://raw.githubusercontent.com/tokio-rs/tokio/tokio-1.54.1/tokio/src/sync/broadcast.rs |
| https://docs.rs/tokio/latest/tokio/sync/watch/index.html | https://raw.githubusercontent.com/tokio-rs/tokio/tokio-1.54.1/tokio/src/sync/watch.rs |
| https://tokio.rs/tokio/tutorial/channels | https://raw.githubusercontent.com/tokio-rs/website/master/content/tokio/tutorial/channels.md |

## select-and-cancellation

| Docs page | Raw source |
| :-- | :-- |
| https://docs.rs/tokio/latest/tokio/macro.select.html | https://raw.githubusercontent.com/tokio-rs/tokio/tokio-1.54.1/tokio/src/macros/select.rs |
| https://docs.rs/tokio/latest/tokio/macro.join.html | https://raw.githubusercontent.com/tokio-rs/tokio/tokio-1.54.1/tokio/src/macros/join.rs |
| https://docs.rs/tokio/latest/tokio/macro.try_join.html | https://raw.githubusercontent.com/tokio-rs/tokio/tokio-1.54.1/tokio/src/macros/try_join.rs |
| https://tokio.rs/tokio/tutorial/select | https://raw.githubusercontent.com/tokio-rs/website/master/content/tokio/tutorial/select.md |
| https://tokio.rs/tokio/topics/shutdown | https://raw.githubusercontent.com/tokio-rs/website/master/content/tokio/topics/shutdown.md |
| https://tokio.rs/tokio/tutorial/async | https://raw.githubusercontent.com/tokio-rs/website/master/content/tokio/tutorial/async.md |

## time-and-testing

| Docs page | Raw source |
| :-- | :-- |
| https://docs.rs/tokio/latest/tokio/time/index.html | https://raw.githubusercontent.com/tokio-rs/tokio/tokio-1.54.1/tokio/src/time/mod.rs |
| https://docs.rs/tokio/latest/tokio/time/fn.sleep.html | https://raw.githubusercontent.com/tokio-rs/tokio/tokio-1.54.1/tokio/src/time/sleep.rs |
| https://docs.rs/tokio/latest/tokio/time/struct.Interval.html | https://raw.githubusercontent.com/tokio-rs/tokio/tokio-1.54.1/tokio/src/time/interval.rs |
| https://docs.rs/tokio/latest/tokio/time/fn.timeout.html | https://raw.githubusercontent.com/tokio-rs/tokio/tokio-1.54.1/tokio/src/time/timeout.rs |
| https://docs.rs/tokio/latest/tokio/time/fn.pause.html | https://raw.githubusercontent.com/tokio-rs/tokio/tokio-1.54.1/tokio/src/time/clock.rs |
| https://docs.rs/tokio/latest/tokio/time/struct.Instant.html | https://raw.githubusercontent.com/tokio-rs/tokio/tokio-1.54.1/tokio/src/time/instant.rs |
| https://tokio.rs/tokio/topics/testing | https://raw.githubusercontent.com/tokio-rs/website/master/content/tokio/topics/testing.md |

## io-and-net

| Docs page | Raw source |
| :-- | :-- |
| https://docs.rs/tokio/latest/tokio/io/index.html | https://raw.githubusercontent.com/tokio-rs/tokio/tokio-1.54.1/tokio/src/io/mod.rs |
| https://docs.rs/tokio/latest/tokio/io/fn.split.html | https://raw.githubusercontent.com/tokio-rs/tokio/tokio-1.54.1/tokio/src/io/split.rs |
| https://docs.rs/tokio/latest/tokio/io/fn.copy.html | https://raw.githubusercontent.com/tokio-rs/tokio/tokio-1.54.1/tokio/src/io/util/copy.rs |
| https://docs.rs/tokio/latest/tokio/net/index.html | https://raw.githubusercontent.com/tokio-rs/tokio/tokio-1.54.1/tokio/src/net/mod.rs |
| https://docs.rs/tokio/latest/tokio/net/struct.TcpListener.html | https://raw.githubusercontent.com/tokio-rs/tokio/tokio-1.54.1/tokio/src/net/tcp/listener.rs |
| https://docs.rs/tokio/latest/tokio/net/struct.TcpStream.html | https://raw.githubusercontent.com/tokio-rs/tokio/tokio-1.54.1/tokio/src/net/tcp/stream.rs |
| https://docs.rs/tokio/latest/tokio/net/struct.TcpSocket.html | https://raw.githubusercontent.com/tokio-rs/tokio/tokio-1.54.1/tokio/src/net/tcp/socket.rs |
| https://docs.rs/tokio/latest/tokio/net/struct.UdpSocket.html | https://raw.githubusercontent.com/tokio-rs/tokio/tokio-1.54.1/tokio/src/net/udp.rs |
| https://tokio.rs/tokio/tutorial/io | https://raw.githubusercontent.com/tokio-rs/website/master/content/tokio/tutorial/io.md |
| https://tokio.rs/tokio/tutorial/framing | https://raw.githubusercontent.com/tokio-rs/website/master/content/tokio/tutorial/framing.md |

## os-fs-process-signal

| Docs page | Raw source |
| :-- | :-- |
| https://docs.rs/tokio/latest/tokio/net/windows/named_pipe/index.html | https://raw.githubusercontent.com/tokio-rs/tokio/tokio-1.54.1/tokio/src/net/windows/named_pipe.rs |
| https://docs.rs/tokio/latest/tokio/fs/index.html | https://raw.githubusercontent.com/tokio-rs/tokio/tokio-1.54.1/tokio/src/fs/mod.rs |
| https://docs.rs/tokio/latest/tokio/fs/struct.File.html | https://raw.githubusercontent.com/tokio-rs/tokio/tokio-1.54.1/tokio/src/fs/file.rs |
| https://docs.rs/tokio/latest/tokio/process/index.html | https://raw.githubusercontent.com/tokio-rs/tokio/tokio-1.54.1/tokio/src/process/mod.rs |
| https://docs.rs/tokio/latest/tokio/signal/index.html | https://raw.githubusercontent.com/tokio-rs/tokio/tokio-1.54.1/tokio/src/signal/mod.rs |
| https://docs.rs/tokio/latest/tokio/signal/fn.ctrl_c.html | https://raw.githubusercontent.com/tokio-rs/tokio/tokio-1.54.1/tokio/src/signal/ctrl_c.rs |
| https://docs.rs/tokio/latest/tokio/signal/windows/index.html | https://raw.githubusercontent.com/tokio-rs/tokio/tokio-1.54.1/tokio/src/signal/windows.rs |

## Failed pages

None. Every page listed above was downloaded.
