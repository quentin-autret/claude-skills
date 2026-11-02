---
name: rust-std-docs
description: Gives current guidance from the official Rust standard library docs (std 1.99.1 stable) on pitfalls, panics, platform differences, discouraged APIs and newer stable replacements. Use when writing, reviewing or refactoring Rust code that uses std, especially collections (Vec, HashMap, BTreeMap, BinaryHeap, entry, extract_if, sort), Option/Result/error handling (expect, ?, Error, ExitCode, panic), iterators (collect, flatten, Peekable, try_fold), Box/Rc/RefCell/Cow/Pin/mem::take, Arc/Mutex/RwLock/OnceLock/LazyLock/thread::scope/mpsc/Condvar/atomics, String/str/fmt/format!/OsString/CString, io/fs/path (BufWriter, OpenOptions, PathBuf::join), process/env/net/time (Command, set_var, TcpStream, Instant, Duration), From/TryFrom/AsRef/Ord/Drop/Deref/Clone, and unsafe raw pointers, MaybeUninit, transmute, NonNull or integer overflow and float ordering.
metadata:
  version: "1.99.1"
  source: "https://doc.rust-lang.org/std/index.html"
  fetched: "2026-11-02"
---

# Rust standard library (std 1.99.1)

## Overview

This skill holds what the official std docs say that is easy to get wrong: panics, surprising defaults, platform notes, discouraged APIs and the newer stable APIs that replace old idioms. Basics every Rust developer knows are left out. Every claim in `references/` ends with the doc URL it comes from, so you can reopen the page. https://doc.rust-lang.org/std/index.html

## Version covered

Rust std 1.99.1, stable channel. "Since" versions come from the pages. An API marked nightly here is not usable on stable 1.99.1, so do not suggest it. Before using an API stabilized after the project's minimum Rust version, check that version.

## How to navigate

1. Find the domain of the code you write or review in the table below.
2. Open only that reference file. Each one stands alone, with a table of contents at the top.
3. Apply its default. Take the escape hatch only when the stated condition holds.
4. In a review, also check the "Old patterns" table at the end of the file, and suggest the replacement.

## Highest-value rules

- `Command::output()` and `status()` return `Ok` on a non-zero exit. Always check `status.success()`. https://doc.rust-lang.org/std/process/struct.Command.html
- `Path::join` or `PathBuf::push` with an absolute path replaces the whole base path. https://doc.rust-lang.org/std/path/struct.PathBuf.html#method.push
- `env::set_var` is `unsafe` and unsound once threads run (outside Windows). Configure children with `Command::env`. https://doc.rust-lang.org/std/env/fn.set_var.html
- Call `flush()` on a `BufWriter` before drop, which ignores flush errors. https://doc.rust-lang.org/std/io/struct.BufWriter.html
- `OpenOptions::write(true)` without `truncate(true)` leaves the old tail. Use `File::create_new` for atomic creation. https://doc.rust-lang.org/std/fs/struct.OpenOptions.html#method.write
- Prefer `try_exists()` over `exists()`, which hides errors as `false`. https://doc.rust-lang.org/std/path/struct.Path.html#method.try_exists
- Infallible one-time init uses `LazyLock` (1.80). A panicking `LazyLock` init poisons it forever. Fallible init uses `OnceLock` (1.70): load first, handle the error outside, then `get_or_init` or `set`, never `LazyLock<Result<..>>`. https://doc.rust-lang.org/std/sync/struct.LazyLock.html#poisoning https://doc.rust-lang.org/std/sync/struct.OnceLock.html#method.get_or_init
- `thread::scope` (1.63) borrows locals in threads without `Arc` or `'static`. https://doc.rust-lang.org/std/thread/fn.scope.html
- A `recv()` loop ends only when every `Sender` is dropped. Drop the original `tx`. https://doc.rust-lang.org/std/sync/mpsc/index.html
- Use `Arc::into_inner`, not `Arc::try_unwrap(..).ok()`, when the error arc is discarded. https://doc.rust-lang.org/std/sync/struct.Arc.html
- `unwrap_or(f())`, `ok_or(..)` and `map_or(..)` evaluate their argument eagerly. Use the `_else` forms for calls. https://doc.rust-lang.org/std/option/enum.Option.html#method.unwrap_or
- Word `expect` messages as what "should" hold. https://doc.rust-lang.org/std/error/index.html#common-message-styles
- `flatten()` over `Result` items drops errors silently. Collect into `Result<Vec<_>, _>`. https://doc.rust-lang.org/std/result/index.html#iterating-over-result
- Use `extract_if` (Vec 1.87, HashMap/HashSet 1.88, BTreeMap 1.91) to remove and keep matching items. If the iterator is dropped or short-circuits before it is exhausted, the unvisited elements stay in the collection and are never extracted. https://doc.rust-lang.org/std/vec/struct.Vec.html#method.extract_if
- Sort floats with `sort_unstable_by(|a, b| a.total_cmp(b))`, or `sort_by` when stability matters. `total_cmp` orders -NaN, -inf, negatives, -0.0, +0.0, positives, +inf, +NaN, so NaN goes to an end by its sign and -0.0 sorts before +0.0. https://doc.rust-lang.org/std/primitive.f64.html#method.total_cmp
- String slicing takes byte indices and panics off a char boundary. Use `get(..)` or `floor_char_boundary` (1.91). https://doc.rust-lang.org/std/string/struct.String.html
- `CString::new(..).unwrap().as_ptr()` in one expression dangles. Bind the `CString` first. https://doc.rust-lang.org/std/ffi/struct.CString.html
- Measure time with `Instant`. `SystemTime::elapsed` returns `Result` because the clock can go back. https://doc.rust-lang.org/std/time/struct.SystemTime.html
- Implement `From`, never `Into`. Use `TryFrom` instead of a lossy `as`. https://doc.rust-lang.org/std/convert/trait.From.html
- Pick overflow behavior explicitly: `checked_*`, `strict_*` (1.91), `saturating_*`, `wrapping_*`. https://doc.rust-lang.org/std/primitive.i32.html#method.strict_add
- Avoid `transmute`. Use `from_le_bytes`, `as_bytes`, `addr()`, casts. https://doc.rust-lang.org/std/mem/fn.transmute.html#alternatives

## References

| File | Covers |
| :-- | :-- |
| [references/collections.md](references/collections.md) | Vec, slices, arrays, VecDeque, BinaryHeap, HashMap/HashSet, entry API, BTreeMap, Hash |
| [references/option-result-error.md](references/option-result-error.md) | unwrap/expect, eager vs lazy combinators, newer Option/Result methods, Error impls, ExitCode, panics |
| [references/iterators.md](references/iterators.md) | Chains, non-termination and overflow, Peekable, from_fn, successors, implementing Iterator |
| [references/ownership-smart-pointers.md](references/ownership-smart-pointers.md) | Box, Rc, Weak, Cell, RefCell, OnceCell, LazyCell, Cow, mem::take, Pin |
| [references/concurrency.md](references/concurrency.md) | Arc, Mutex, RwLock, poisoning, OnceLock, LazyLock, Condvar, mpsc, threads, scope, atomics |
| [references/strings-formatting.md](references/strings-formatting.md) | String/str indexing, case, format syntax, Display/Debug impls, OsString, CString |
| [references/io-fs-path.md](references/io-fs-path.md) | BufReader/BufWriter, OpenOptions, TOCTOU, Read/Write/BufRead, io::Error, Path/PathBuf |
| [references/process-env-net-time.md](references/process-env-net-time.md) | Command, Child, pipes, Windows quirks, exit, env, TcpListener/TcpStream, Instant, SystemTime, Duration |
| [references/traits-conversions.md](references/traits-conversions.md) | From/Into/TryFrom, AsRef/Borrow, Clone/Copy/Default, PartialEq/Ord, Drop, Deref, Fn, Any, PhantomData |
| [references/unsafe-ptr-num.md](references/unsafe-ptr-num.md) | Pointer validity, provenance, NonNull, from_raw_parts, MaybeUninit, transmute, overflow, floats, hints |

All pages read are listed in [SOURCES.md](SOURCES.md).
