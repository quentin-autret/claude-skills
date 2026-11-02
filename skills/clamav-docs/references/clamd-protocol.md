# clamd socket protocol

## Contents

- Sockets and paths
- Framing
- Commands
- INSTREAM
- FILDES
- Replies
- IDSESSION
- Limits
- Shell checks

## Sockets and paths

- Unix socket: path commands, `INSTREAM`, `FILDES`. TCP: path commands (if valid on the daemon host) and `INSTREAM`, no `FILDES`, no encryption, no authentication. https://docs.clamav.net/manual/Usage/ClamdProtocol.html#socket-types
- clamd resolves paths, not the client. For a daemon in another container or host, send content with `INSTREAM`. https://docs.clamav.net/manual/Usage/ClamdProtocol.html#socket-types
- Send `VERSIONCOMMANDS` at startup: a known command may still be disabled in clamd.conf or not compiled in. https://docs.clamav.net/manual/Usage/ClamdProtocol.html#overview

## Framing

- The first byte selects the terminator. Use `zCOMMAND\0`, NUL terminated, safe with newlines in arguments. `nCOMMAND\n` also works. Replies use the same terminator. https://docs.clamav.net/manual/Usage/ClamdProtocol.html#command-framing
- Unprefixed legacy commands still work outside a session for some commands, are rejected inside `IDSESSION`, and new clients must not use them. https://docs.clamav.net/manual/Usage/ClamdProtocol.html#command-framing

```text
zPING\0
zSCAN /srv/uploads/sample.bin\0
```

## Commands

| Command | Behavior | Gate in clamd.conf |
| ------- | -------- | ------------------ |
| `SCAN path` | File or recursive directory, stops at the first virus in a directory, full path | |
| `CONTSCAN path` | Like `SCAN`, continues after detections | |
| `MULTISCAN path` | Directory scan on several worker threads | |
| `ALLMATCHSCAN path` | Keeps matching within a file after a hit, n or z framing | `AllowAllMatchScan` |
| `INSTREAM` | Streamed content, n or z framing | `StreamMaxLength` caps size |
| `FILDES` | Open file descriptor, Unix socket only | |
| `PING` | Replies `PONG` | |
| `VERSION` | On by default | `EnableVersionCommand` |
| `VERSIONCOMMANDS` | Version then `COMMANDS:` list | |
| `RELOAD` | Off unless enabled | `EnableReloadCommand` |
| `SELFCHECK` | Off unless enabled | `EnableSelfCheckCommand` |
| `SHUTDOWN` | Off unless enabled | `EnableShutdownCommand` |
| `STATS` | On by default, unstable text format, read until `END` | `EnableStatsCommand` |

A disabled `VERSION`, `RELOAD`, `SELFCHECK`, `SHUTDOWN` or `STATS` replies `COMMAND UNAVAILABLE` (a disabled `ALLMATCHSCAN` returns an error). https://docs.clamav.net/manual/Usage/ClamdProtocol.html

- The legacy `STREAM` command is no longer supported. Use `INSTREAM`. https://docs.clamav.net/manual/Usage/ClamdProtocol.html#legacy-stream

## INSTREAM

- After `zINSTREAM\0`, send chunks as a 4-byte unsigned big-endian length followed by the data. End with a zero-length chunk, `00 00 00 00`. https://docs.clamav.net/manual/Usage/ClamdProtocol.html#instream
- A total over `StreamMaxLength` fails with `INSTREAM size limit exceeded`. https://docs.clamav.net/manual/Usage/ClamdProtocol.html#instream

```python
import socket, struct

with socket.socket(socket.AF_UNIX, socket.SOCK_STREAM) as sock:
    sock.connect("/run/clamav/clamd.sock")
    sock.sendall(b"zINSTREAM\0")
    with open(path, "rb") as f:
        while chunk := f.read(1024 * 1024):
            sock.sendall(struct.pack(">I", len(chunk)) + chunk)
    sock.sendall(struct.pack(">I", 0))
    reply = b""
    while data := sock.recv(4096):
        reply += data
print(reply.split(b"\0")[0].decode())  # "stream: OK" or "stream: <Name> FOUND"
```

https://docs.clamav.net/manual/Usage/ClamdProtocol.html#instream-example

## FILDES

- Pass the descriptor as Unix socket ancillary data with at least one dummy byte, in the same packet as the command or later. https://docs.clamav.net/manual/Usage/ClamdProtocol.html#fildes

```python
sock.sendall(b"zFILDES\0")
sock.sendmsg([b"x"], [(socket.SOL_SOCKET, socket.SCM_RIGHTS, array.array("i", [fd]))])
```

https://docs.clamav.net/manual/Usage/ClamdProtocol.html#fildes-example

## Replies

- Outside a session, read until clamd closes the connection. A directory scan returns several records. https://docs.clamav.net/manual/Usage/ClamdProtocol.html#reading-replies

```text
/path/to/file: OK
/path/to/file: Eicar-Signature FOUND
/path/to/file: Access denied ERROR
stream: OK
stream: Eicar-Signature FOUND
```

- Parse the suffix: `OK` clean, `FOUND` detection with the signature name before it, `ERROR` failure. https://docs.clamav.net/manual/Usage/ClamdProtocol.html#reading-replies

## IDSESSION

- `zIDSESSION\0`, then framed commands, then `zEND\0`. Accepted inside: `SCAN`, `INSTREAM`, `FILDES`, `VERSION`, `VERSIONCOMMANDS`, `PING`, `STATS`, `END`. Rejected: `CONTSCAN`, `MULTISCAN`, `ALLMATCHSCAN`, `RELOAD`, `SHUTDOWN`, `SELFCHECK`. https://docs.clamav.net/manual/Usage/ClamdProtocol.html#commands-accepted-inside-idsession
- Replies carry a request ID starting at 1 (`IDSESSION` takes none) and may arrive out of order. Match them by ID. Avoid `STATS` beside other in-flight work. https://docs.clamav.net/manual/Usage/ClamdProtocol.html#idsession-replies

```text
1: /path/a: OK
2: /path/b: Eicar-Signature FOUND
3: PONG
```

- Keep reading while sending, with a nonblocking socket and `select()` or `poll()`. Alternating blocking sends and receives can deadlock, and clamd may close the connection. https://docs.clamav.net/manual/Usage/ClamdProtocol.html#avoiding-idsession-deadlocks
- `IDSESSION` is the client-driven alternative to `MULTISCAN` for parallel scanning. https://docs.clamav.net/manual/Usage/ClamdProtocol.html#using-idsession-for-parallel-scanning

## Limits

- A framed path command fits in `PATH_MAX + 8` bytes. Use `INSTREAM` or `FILDES` for long paths. https://docs.clamav.net/manual/Usage/ClamdProtocol.html#protocol-limits
- `INSTREAM` content does not count toward that limit. Chunk lengths are 32-bit, the total is bounded by `StreamMaxLength`. https://docs.clamav.net/manual/Usage/ClamdProtocol.html#protocol-limits
- A session has no request limit, but IDs are finite: start a new session for very large batches. Concurrency is bounded by `MaxThreads`, `MaxQueue`, descriptors, buffers, memory, `CommandReadTimeout` and `ReadTimeout`. Cap in-flight requests. https://docs.clamav.net/manual/Usage/ClamdProtocol.html#protocol-limits

## Shell checks

```bash
printf 'zPING\0' | nc -U /run/clamav/clamd.sock | tr '\0' '\n'
printf 'zVERSIONCOMMANDS\0' | nc -U /run/clamav/clamd.sock | tr '\0' '\n'
printf 'zPING\0' | nc 127.0.0.1 3310 | tr '\0' '\n'
```

https://docs.clamav.net/manual/Usage/ClamdProtocol.html#basic-checks
