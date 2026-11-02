# process, env, net and time (std 1.99.1)

## Contents
- Command
- Child and pipes
- Windows specifics
- Exit
- env
- net
- time
- Old patterns

## Command

- `Command::new` takes only the program. `Command::new("ls -l")` looks for a program named `ls -l`. One `.arg` is one argument: `.arg("-C").arg(path)`, never `.arg("-C /path")`. https://doc.rust-lang.org/std/process/struct.Command.html
- No shell is involved. Quotes, globs, pipes and `$VAR` have no effect. https://doc.rust-lang.org/std/process/struct.Command.html
- `output()` and `status()` return `Ok` even when the child exits non-zero or is killed. Always check `status.success()`. `Err` means only spawn, read or wait failed. https://doc.rust-lang.org/std/process/struct.Command.html

```rust
let out = Command::new("ls").args(["-l", "/tmp"]).output()?;
if !out.status.success() {
    return Err(io::Error::other(String::from_utf8_lossy(&out.stderr).into_owned()));
}
```

- Defaults differ: `spawn` and `status` inherit all 3 streams. `output` pipes stdout and stderr, and gives stdin a closed stream. https://doc.rust-lang.org/std/process/struct.Command.html
- Set a child's environment with `.env`, `.envs`, `.env_remove`, `.env_clear`, never with `env::set_var`. https://doc.rust-lang.org/std/env/fn.set_var.html
- After `env_clear()` on Unix, `PATH` lookup falls back to an OS default like `/bin:/usr/bin`, not the parent's `PATH`. Use an absolute program path or set `PATH` explicitly. https://doc.rust-lang.org/std/process/struct.Command.html
- A relative program combined with `current_dir` resolves in a platform-specific way. Canonicalize the program path. https://doc.rust-lang.org/std/process/struct.Command.html
- Env var names are case-insensitive on Windows, case-sensitive elsewhere. https://doc.rust-lang.org/std/process/struct.Command.html
- `Command`'s `Debug` output looks like a shell line but is not safely escaped. https://doc.rust-lang.org/std/process/struct.Command.html

## Child and pipes

- Dropping a `Child` neither waits nor kills it. Without `wait`, it may stay a zombie and exhaust resources such as process IDs, so dropping unwaited handles is not recommended in long-running programs. https://doc.rust-lang.org/std/process/struct.Child.html
- `wait_with_output` captures only streams set to `Stdio::piped()`. https://doc.rust-lang.org/std/process/struct.Child.html
- Writing a large stdin while not reading stdout can deadlock on the pipe buffer. Take stdin, write it from another thread, then `wait_with_output`: https://doc.rust-lang.org/std/process/index.html#handling-io

```rust
let mut child = Command::new("sort").stdin(Stdio::piped()).stdout(Stdio::piped()).spawn()?;
let mut stdin = child.stdin.take().expect("stdin should be piped");
std::thread::spawn(move || stdin.write_all(&big_input));
let output = child.wait_with_output()?;
```

- `wait` closes the child's stdin first. `try_wait` does not. https://doc.rust-lang.org/std/process/struct.Child.html
- `kill` is SIGKILL on Unix and returns `Ok` if the child already exited. https://doc.rust-lang.org/std/process/struct.Child.html
- Redirect child output to the parent's with `Stdio::from(io::stderr())` (1.74.0). https://doc.rust-lang.org/std/process/struct.Stdio.html

## Windows specifics

- Rust resolves the program itself: the child's `PATH` if set, the exe directory, system dirs, then the parent's `PATH`, even after `env_clear`. https://doc.rust-lang.org/std/process/struct.Command.html
- Windows passes one command-line string that the child parses. `cmd.exe` and `.bat` parse differently, and an untrusted argument to them can run commands. Validate input or build the line with `raw_arg`. https://doc.rust-lang.org/std/process/index.html#windows-argument-splitting
- Spawning a `.bat` directly is historical behavior and may be removed. Some characters make `spawn` fail. https://doc.rust-lang.org/std/process/index.html#batch-file-special-handling

## Exit

- Return `ExitCode` or `Result` from `main`. `process::exit` runs no destructors, so buffered writers are not flushed. https://doc.rust-lang.org/std/process/fn.exit.html
- On most Unix-like platforms only the low 8 bits of the exit code are considered: `exit(256)` gives 0 on Linux and 256 on Windows. https://doc.rust-lang.org/std/process/fn.exit.html

## env

- `env::set_var` and `remove_var` are `unsafe`. They are sound only single-threaded or on Windows, because std itself (DNS lookups) reads the environment. https://doc.rust-lang.org/std/env/fn.set_var.html

```rust
unsafe { env::set_var("KEY", "VALUE"); } // only before any thread starts
```

- `env::var` returns `Err(NotUnicode)` on non-UTF-8 values. Use `var_os` when that can happen. Use `env!` / `option_env!` for build-time values. https://doc.rust-lang.org/std/env/fn.var.html
- `env::args()` panics during iteration on a non-Unicode argument. Use `args_os()`. The first item is not reliable for security. https://doc.rust-lang.org/std/env/fn.args.html
- Unix shells expand globs, Windows passes them as-is. https://doc.rust-lang.org/std/env/fn.args.html

## net

- Bind port 0 to let the OS pick, then read `local_addr()`. https://doc.rust-lang.org/std/net/struct.TcpListener.html
- In an `accept` loop, treat per-connection errors and fd exhaustion as recoverable. Log and keep accepting. https://doc.rust-lang.org/std/net/struct.TcpListener.html
- `incoming()` never returns `None` and does not give the peer address. Use `accept()` for the address. https://doc.rust-lang.org/std/net/struct.TcpListener.html
- `set_read_timeout(Some(Duration::ZERO))` is an error. A timeout gives `WouldBlock` on Unix, but Windows may give `TimedOut`, so match both. https://doc.rust-lang.org/std/net/struct.TcpStream.html

```rust
Err(e) if matches!(e.kind(), ErrorKind::WouldBlock | ErrorKind::TimedOut) => { /* timeout */ }
```

- `connect_timeout` needs one resolved `SocketAddr`, not a host string. `connect` to a name tries each address and returns the last error. https://doc.rust-lang.org/std/net/struct.TcpStream.html
- Unix writes use `MSG_NOSIGNAL`, so a closed peer gives an error, not SIGPIPE. https://doc.rust-lang.org/std/net/struct.TcpStream.html
- `set_nodelay` is stable. `set_linger` and `set_keepalive` are nightly. https://doc.rust-lang.org/std/net/struct.TcpStream.html

## time

- Measure elapsed time with `Instant`, never `SystemTime`, which can go backwards. https://doc.rust-lang.org/std/time/struct.SystemTime.html
- `SystemTime::elapsed` and `duration_since` return `Result`. Handle the `Err` (clock moved back) in code that must survive clock changes. https://doc.rust-lang.org/std/time/struct.SystemTime.html
- Unix timestamp: `SystemTime::now().duration_since(UNIX_EPOCH)?.as_secs()`. https://doc.rust-lang.org/std/time/struct.SystemTime.html
- `Instant::duration_since` and `elapsed` saturate to zero, but future versions may reintroduce panics. Use `checked_duration_since` to detect swapped operands. https://doc.rust-lang.org/std/time/struct.Instant.html#monotonicity
- `Instant + Duration` can panic when the result is not representable. A 1000-year duration panics on macOS but not on Linux. Durations up to about 100 years are portable. Use `checked_add`. https://doc.rust-lang.org/std/time/struct.Instant.html#os-specific-behaviors
- Whether a system suspend counts as elapsed time for `Instant` is unspecified and varies by platform and Rust version. https://doc.rust-lang.org/std/time/struct.Instant.html
- `Duration::from_secs_f64` panics when the value is negative, overflows or is not finite. For untrusted input use `try_from_secs_f64` (1.66.0). https://doc.rust-lang.org/std/time/struct.Duration.html
- A `Duration` result cannot be negative. `checked_sub` returns `None` and `saturating_sub` returns zero then. `abs_diff` gives the absolute difference. https://doc.rust-lang.org/std/time/struct.Duration.html
- `Duration` has no `Display`. `Debug` prints a `µs` suffix that is not ASCII. https://doc.rust-lang.org/std/time/struct.Duration.html
- `Duration::from_mins` and `from_hours` are stable since 1.91.0. `from_days` and `from_weeks` are nightly. https://doc.rust-lang.org/std/time/struct.Duration.html

## Old patterns

| Old | Current |
| :-- | :-- |
| `env::set_var` to configure a child | `Command::env` https://doc.rust-lang.org/std/env/fn.set_var.html |
| `process::exit` at the end of `main` | return `ExitCode` https://doc.rust-lang.org/std/process/fn.exit.html |
| `SystemTime` to time code | `Instant` https://doc.rust-lang.org/std/time/struct.SystemTime.html |
| `Duration::from_secs(n * 60)` | `Duration::from_mins(n)` https://doc.rust-lang.org/std/time/struct.Duration.html |
| `CommandExt::before_exec` (deprecated) | `unsafe` `pre_exec` https://doc.rust-lang.org/std/process/struct.Command.html |
