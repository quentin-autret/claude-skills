# io, fs and path (std 1.99.1)

## Contents
- Whole files and buffering
- OpenOptions
- Existence and TOCTOU
- Read, Write, BufRead
- io::Error and ErrorKind
- Path and PathBuf
- Old patterns

## Whole files and buffering

- Read a whole file with `fs::read_to_string` or `fs::read`, write one with `fs::write` (truncates, fails on some platforms if the parent dir is missing). https://doc.rust-lang.org/std/fs/fn.write.html
- `File` is unbuffered. Wrap it in `BufReader` / `BufWriter` for many small operations. No gain for a few large calls or in-memory data. https://doc.rust-lang.org/std/io/struct.BufWriter.html
- Call `flush()` on a `BufWriter` before it drops. Drop flushes but ignores errors. https://doc.rust-lang.org/std/io/struct.BufWriter.html

```rust
let mut out = BufWriter::new(File::create("out.txt")?);
for line in lines { writeln!(out, "{line}")?; }
out.flush()?;
```

- `File` drop ignores close errors. Call `sync_all()` (or the cheaper `sync_data()`) when durability matters. https://doc.rust-lang.org/std/fs/struct.File.html
- Two `BufReader`s on one stream, or using the inner reader after `into_inner`, can lose buffered data. https://doc.rust-lang.org/std/io/struct.BufReader.html
- `File::open_buffered` is nightly-only on 1.99.1. https://doc.rust-lang.org/std/fs/struct.File.html

## OpenOptions

- Create a file only if absent with `File::create_new` or `.write(true).create_new(true)`. It is atomic and avoids TOCTOU. https://doc.rust-lang.org/std/fs/struct.OpenOptions.html#method.create_new
- `.write(true)` on an existing file overwrites from the start and leaves the old tail. Add `.truncate(true)`. https://doc.rust-lang.org/std/fs/struct.OpenOptions.html#method.write
- `.create(true)` without write or append access fails with `InvalidInput`. https://doc.rust-lang.org/std/fs/struct.OpenOptions.html#method.create
- Append with `.append(true).create(true)`, not `seek(End)` then write, which races. Write each record in one `write` call. https://doc.rust-lang.org/std/fs/struct.OpenOptions.html#method.append

```rust
let log = OpenOptions::new().append(true).create(true).open("app.log")?;
```

- File locks (`lock`, `lock_shared`, `try_lock`) use `flock` on Unix and `LockFileEx` on Windows. Read each method's platform notes. https://doc.rust-lang.org/std/fs/struct.File.html

## Existence and TOCTOU

- `Path::exists()` returns `false` on permission errors and broken symlinks. Prefer `path.try_exists()?` (1.63.0) or `fs::exists`. https://doc.rust-lang.org/std/path/struct.Path.html#method.try_exists
- Checking then acting is a race. Open atomically instead, and keep the file open for the operation. https://doc.rust-lang.org/std/fs/index.html
- `Path::is_symlink()` returns `false` on errors (for example a permission error). Use `fs::symlink_metadata` to see them. https://doc.rust-lang.org/std/path/struct.Path.html#method.is_symlink
- `fs::rename` replaces an existing destination (on Unix, a directory only if empty). https://doc.rust-lang.org/std/fs/fn.rename.html
- `fs::copy` also copies the permission bits. https://doc.rust-lang.org/std/fs/index.html

## Read, Write, BufRead

- `read` may return fewer bytes than asked. `write` may write fewer. Use `read_exact`, `read_to_end` and `write_all`. https://doc.rust-lang.org/std/io/trait.Write.html
- `read_line` appends to the buffer. `clear()` it each loop, and `Ok(0)` means EOF. https://doc.rust-lang.org/std/io/trait.BufRead.html#method.read_line

```rust
let mut line = String::new();
while reader.read_line(&mut line)? != 0 { handle(&line); line.clear(); }
```

- `read_line`, `read_until`, `read_to_end` and `read_to_string` read until delimiter or EOF. Bound untrusted input with `.take(limit)`. https://doc.rust-lang.org/std/io/trait.BufRead.html#method.read_line
- `read_to_end` and `read_to_string` on a continuous stream such as an interactive stdin block until EOF. https://doc.rust-lang.org/std/io/trait.Read.html#method.read_to_end
- `ErrorKind::Interrupted` means retry. `read_to_end`, `read_to_string` and `write_all` retry it for you. https://doc.rust-lang.org/std/io/trait.Read.html#tymethod.read
- Call `write!` / `writeln!` rather than `write_fmt` directly. https://doc.rust-lang.org/std/io/trait.Write.html#method.write_fmt

## io::Error and ErrorKind

- Wrap any error with `io::Error::other(e)` (1.74.0). For a bare kind use `io::Error::from(ErrorKind::NotFound)`, which does not allocate. https://doc.rust-lang.org/std/io/struct.Error.html#method.other
- `ErrorKind` is `#[non_exhaustive]` and grows. Match the kinds you need plus `_`. https://doc.rust-lang.org/std/io/enum.ErrorKind.html
- Call `io::Error::last_os_error()` right after the failing platform call. https://doc.rust-lang.org/std/io/struct.Error.html#method.last_os_error
- Never close or use a file descriptor you do not own or borrow. https://doc.rust-lang.org/std/io/index.html

## Path and PathBuf

- `join` or `push` with an absolute path REPLACES the base: `Path::new("/tmp").join("/etc")` is `/etc`. Never join untrusted input and assume it stays under the base. https://doc.rust-lang.org/std/path/struct.PathBuf.html#method.push
- To check a user-supplied name, walk `Path::new(name).components()` and accept only `Component::Normal(_)`. Reject `ParentDir` (`..`), `RootDir`, `Prefix` (Windows `C:` or `\\server\share`) and `CurDir`. `components()` drops repeated separators and inner `.` but leaves `..` unresolved, since `a/b/..` may differ from `a` with symlinks. https://doc.rust-lang.org/std/path/struct.Path.html#method.components https://doc.rust-lang.org/std/path/enum.Component.html
- A name with a root or prefix is absolute or partly so. `is_absolute` alone misses Windows `\temp` and `c:temp`, so check components. `join` with an absolute path replaces the base. `canonicalize` resolves symlinks and `..` against the filesystem. https://doc.rust-lang.org/std/path/struct.Path.html#method.join https://doc.rust-lang.org/std/path/struct.Path.html#method.canonicalize
- `push` and `set_file_name` do not sanitize `..` or separators. https://doc.rust-lang.org/std/path/struct.PathBuf.html#method.set_file_name
- On Windows, joining `\windows` keeps only the drive prefix, and joining `C:foo` replaces everything. https://doc.rust-lang.org/std/path/struct.PathBuf.html#method.push
- `starts_with` and `ends_with` compare whole components: `Path::new("file.conf").ends_with("conf")` is false. Test `extension()` instead. https://doc.rust-lang.org/std/path/struct.Path.html#method.ends_with
- `extension()` of a dotfile like `.hidden` is `None` and `file_stem()` is the whole name. `file_name()` of a path ending in `..` is `None`. https://doc.rust-lang.org/std/path/struct.Path.html#method.extension
- `set_extension` panics if the extension contains a separator. `""` removes the extension. To append (`a.tar` to `a.tar.gz`) use `with_added_extension` / `add_extension` (1.91.0). https://doc.rust-lang.org/std/path/struct.PathBuf.html#method.set_extension
- Comparison ignores repeated separators, non-leading `.` components and trailing separators. `..` is never resolved, so `a/b/../c` differs from `a/c`. Comparisons are case sensitive, even on Windows (except drive letters). https://doc.rust-lang.org/std/path/index.html
- `canonicalize` hits the filesystem and fails if the path does not exist. `std::path::absolute` does not touch the filesystem and keeps `..`. https://doc.rust-lang.org/std/path/struct.Path.html#method.canonicalize
- On Windows, `c:\x` is absolute but `\x` and `c:x` are not. https://doc.rust-lang.org/std/path/struct.Path.html#method.is_absolute
- Print with `path.display()` (lossy) or `{:?}`. `to_str()` returns `None` on non-UTF-8. https://doc.rust-lang.org/std/path/struct.Path.html#method.display
- Build from known parts with `collect()`. `push` suits parts not known up front. https://doc.rust-lang.org/std/path/index.html

```rust
let p: PathBuf = [base, "logs", "today.txt"].iter().collect();
```

- `read_dir` yields `io::Result<DirEntry>`. Errors can appear mid-iteration. https://doc.rust-lang.org/std/path/struct.Path.html#method.read_dir

## Old patterns

| Old | Current |
| :-- | :-- |
| `File::open` + `read_to_string` | `fs::read_to_string` https://doc.rust-lang.org/std/fs/struct.File.html#method.open |
| `if !p.exists() { File::create(p) }` | `File::create_new(p)` https://doc.rust-lang.org/std/fs/struct.OpenOptions.html#method.create_new |
| `p.exists()` | `p.try_exists()?` https://doc.rust-lang.org/std/path/struct.Path.html#method.exists |
| `io::Error::new(ErrorKind::Other, e)` | `io::Error::other(e)` https://doc.rust-lang.org/std/io/struct.Error.html#method.other |
| `fs::soft_link` (deprecated) | do not use (deprecated) https://doc.rust-lang.org/std/fs/index.html |
| `err.description()` (deprecated since 1.42.0) | `err.to_string()` https://doc.rust-lang.org/std/io/struct.Error.html |
