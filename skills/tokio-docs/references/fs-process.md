# Files and child processes

## Contents
- How tokio::fs works
- Batching file operations
- tokio::fs::File
- Command
- Child: wait, kill, pipes

## How tokio::fs works

- `tokio::fs` runs ordinary blocking file calls on the `spawn_blocking` pool, on every platform. On Linux with `tokio_unstable` it also uses io_uring for file operations. https://docs.rs/tokio/latest/tokio/fs/index.html
- Use it for ordinary files only. A named pipe on Linux can hang runtime shutdown. For special files use `tokio::net::unix::pipe` or `AsyncFd`. https://docs.rs/tokio/latest/tokio/fs/index.html
- Dropping a fs future stops the wait, not the operation. A cancelled `write()` that was polled still reaches the disk. https://docs.rs/tokio/latest/tokio/fs/index.html

## Batching file operations

- Each operation costs one `spawn_blocking` call, with "serious performance consequences". Batch into as few calls as possible. https://docs.rs/tokio/latest/tokio/fs/index.html
- Default: `tokio::fs::read` / `read_to_string` for a whole file, and build the content in memory before one `tokio::fs::write`. Read in chunks only for files too big for memory. https://docs.rs/tokio/latest/tokio/fs/index.html
- For many small writes, wrap the file in `BufWriter` and flush: https://docs.rs/tokio/latest/tokio/fs/index.html

```rust
let mut file = BufWriter::new(File::create("my_file.txt").await?);
file.write_all(b"First line.\n").await?;
file.flush().await?;
```
https://docs.rs/tokio/latest/tokio/fs/index.html

- Escape hatch for a long sequence of file work: run `std::fs` code inside one `spawn_blocking` closure. https://docs.rs/tokio/latest/tokio/fs/index.html
- `File::set_max_buf_size` sets the maximum buffer size of each underlying read or write (a large `write_all` goes out in chunks up to that size). 0 panics. https://docs.rs/tokio/latest/tokio/fs/struct.File.html#method.set_max_buf_size

## tokio::fs::File

- `write` returns before the data is written. Always call `flush().await` after writing, unlike `std::fs::File`. https://docs.rs/tokio/latest/tokio/fs/struct.File.html
- A `File` with pending operations is not closed on drop. `flush` before dropping closes it at once, but the OS may still hold the data in memory. `sync_all` asks the OS to push all in-core data to the filesystem. `sync_data` is similar but may skip metadata. https://docs.rs/tokio/latest/tokio/fs/struct.File.html and https://docs.rs/tokio/latest/tokio/fs/struct.File.html#method.sync_data
- `File::create_new` fails if the file exists, atomically, which avoids a check-then-create race. https://docs.rs/tokio/latest/tokio/fs/struct.File.html#method.create_new
- `File::options()` equals `OpenOptions::new()` without an import. https://docs.rs/tokio/latest/tokio/fs/struct.File.html#method.options
- Opening a `std::fs::File` to pass to `from_std` can block and is not recommended on the runtime. https://docs.rs/tokio/latest/tokio/fs/struct.File.html#method.from_std
- `into_std` waits for in-flight operations. `try_into_std` fails, returning the file, while one is in flight. https://docs.rs/tokio/latest/tokio/fs/struct.File.html#method.into_std
- `try_clone` shares one handle: reads, writes and seeks affect both. https://docs.rs/tokio/latest/tokio/fs/struct.File.html#method.try_clone

## Command

- A child keeps running after its `Child` is dropped. Set `kill_on_drop(true)` to kill it on drop (default false). https://docs.rs/tokio/latest/tokio/process/index.html
- `kill_on_drop` does not wait. On Unix the killed child stays a zombie until reaped, best effort, and zombies count against system limits. For strict cleanup, `child.wait().await` or `child.kill().await` instead of dropping. https://docs.rs/tokio/latest/tokio/process/struct.Command.html#method.kill_on_drop
- `arg` takes one argument per call: `.arg("-C").arg("/path")`, never `.arg("-C /path")`. https://docs.rs/tokio/latest/tokio/process/struct.Command.html#method.arg
- `output()` forces stdout and stderr to pipes, even if set before (unlike std). To keep your stdio config, `spawn` then `wait_with_output`. https://docs.rs/tokio/latest/tokio/process/struct.Command.html#method.output
- stdin inherits by default. stdout and stderr inherit for `spawn` / `status` and are piped for `output`. https://docs.rs/tokio/latest/tokio/process/struct.Command.html#method.stdout
- `current_dir` with a relative program path is platform specific. Use `tokio::fs::canonicalize` to pass an absolute program path. https://docs.rs/tokio/latest/tokio/process/struct.Command.html#method.current_dir
- Env var names are case-insensitive on Windows. https://docs.rs/tokio/latest/tokio/process/struct.Command.html#method.env
- The program is searched on `PATH` if not absolute. Setting `PATH` on the `Command` to steer that search has implementation limits on Windows. https://docs.rs/tokio/latest/tokio/process/struct.Command.html#method.new
- On Windows, `raw_arg` appends text with no quoting, for `cmd.exe /c`. `creation_flags` is always ORed with `CREATE_UNICODE_ENVIRONMENT`. https://docs.rs/tokio/latest/tokio/process/struct.Command.html
- On Unix, `spawn` fails with `ErrorKind::WouldBlock` at the system process limit. https://docs.rs/tokio/latest/tokio/process/struct.Command.html#method.spawn
- On Unix, a child with its own `process_group` does not get the terminal's Ctrl-C. https://docs.rs/tokio/latest/tokio/process/struct.Command.html#method.process_group
- `pre_exec` runs after `fork` in the child, where allocating or locking a mutex may not work. https://docs.rs/tokio/latest/tokio/process/struct.Command.html#method.pre_exec
- `into_std` drops Tokio options, `kill_on_drop` included. https://docs.rs/tokio/latest/tokio/process/struct.Command.html

## Child: wait, kill, pipes

- Take pipes out with `child.stdout.take()` so `child` is not partially moved and stays usable (e.g. `child.wait()`). They exist only with `Stdio::piped()`. https://docs.rs/tokio/latest/tokio/process/struct.Child.html and https://docs.rs/tokio/latest/tokio/process/struct.Child.html#method.wait_with_output
- Write to the child's stdin in a separate task from the one that awaits its exit or output, then drop stdin to send EOF. Otherwise both sides can block on full pipes. https://docs.rs/tokio/latest/tokio/process/index.html
- `wait()` closes the child's stdin first. Take stdin out before waiting to control when it closes. `wait()` is cancel safe. https://docs.rs/tokio/latest/tokio/process/struct.Child.html#method.wait
- `kill()` sends SIGKILL on Unix and then waits. `start_kill()` only sends it, like std's `kill`. After `start_kill` on Unix, `wait().await` (or a successful `try_wait()`) to avoid a zombie. https://docs.rs/tokio/latest/tokio/process/struct.Child.html#method.start_kill

```rust
tokio::select! {
    _ = child.wait() => {}
    _ = recv => child.kill().await.expect("kill failed"),
}
```
https://docs.rs/tokio/latest/tokio/process/struct.Child.html#method.kill

- `id()` returns `None` once the child was awaited, to avoid PID reuse confusion. https://docs.rs/tokio/latest/tokio/process/struct.Child.html#method.id
- To pipe one child into another, convert `stdout.take().unwrap()` into `Stdio` with `try_into()`, then `join!` both waits. https://docs.rs/tokio/latest/tokio/process/index.html
