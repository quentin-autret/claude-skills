# Option, Result, errors, panics and exit codes (std 1.99.1)

## Contents
- Result or panic
- unwrap and expect
- Eager and lazy combinators
- Newer Option and Result methods
- Writing an Error type
- main, ExitCode and panics
- Old patterns

## Result or panic

- Use `Result` for anticipated, recoverable failures. Use `panic!` for detected bugs. `panic!` suits assertions in tests and example code. https://doc.rust-lang.org/std/macro.panic.html#when-to-use-panic-vs-result
- `Result` is `#[must_use]`: ignoring one triggers the `unused_must_use` warning. Handle it with `?`, `expect`, or an explicit check. https://doc.rust-lang.org/std/result/index.html#results-must-be-used

## unwrap and expect

- `expect` is generally preferred over `unwrap`. `unwrap` stays a good fit where code trivially cannot panic, like parsing a literal, or in early prototyping. https://doc.rust-lang.org/std/error/index.html#converting-errors-into-panics
- Word an `expect` message as a precondition with "should". The panic output already appends the error, so do not repeat it. https://doc.rust-lang.org/std/error/index.html#common-message-styles

```rust
let path = std::env::var("IMPORTANT_PATH")
    .expect("env variable `IMPORTANT_PATH` should be set by `wrapper_script.sh`");
// panics with: env variable `IMPORTANT_PATH` should be set by `wrapper_script.sh`: NotPresent
let first = slice.get(0).expect("slice should not be empty");
```

- `Result::unwrap` / `expect` need `E: Debug`. `unwrap_err` / `expect_err` need `T: Debug`. https://doc.rust-lang.org/std/result/index.html#extracting-contained-values
- `unwrap_unchecked` on `None` or `Err` is undefined behavior. https://doc.rust-lang.org/std/option/enum.Option.html#method.unwrap_unchecked

## Eager and lazy combinators

- Arguments to `unwrap_or`, `map_or`, `ok_or`, `and`, `or` are evaluated even when unused. When the argument is a function call or allocates, use `unwrap_or_else`, `map_or_else`, `ok_or_else`, `and_then`, `or_else`. https://doc.rust-lang.org/std/option/enum.Option.html#method.unwrap_or

```rust
let v = opt.unwrap_or_else(|| expensive_default());
let r = opt.ok_or_else(|| format!("missing {key}"));
```

- Call `as_ref()` or `as_deref()` before `map` to avoid moving out of an `Option<String>`. https://doc.rust-lang.org/std/option/enum.Option.html#method.as_ref
- `Option::insert` drops the old value. `get_or_insert_with` keeps an existing one. https://doc.rust-lang.org/std/option/enum.Option.html#method.insert
- `flatten()` on an iterator of `Result` silently drops every `Err`. Collect into `Result<Vec<_>, _>` to stop at the first error instead: https://doc.rust-lang.org/std/result/index.html#iterating-over-result

```rust
let all: Result<Vec<u8>, _> = inputs.iter().map(|s| s.parse::<u8>()).collect();
let total: Option<i32> = [Some(1), Some(2)].into_iter().sum();
```

https://doc.rust-lang.org/std/result/index.html#collecting-into-result and https://doc.rust-lang.org/std/option/index.html#collecting-into-option

- To add one optional item to an `impl Iterator`, chain the `Option` itself: `(0..4).chain(extra).chain(4..8)`. Both branches then share one type. https://doc.rust-lang.org/std/option/index.html#iterating-over-option
- Ordering when `T` (and `E`) implement `PartialOrd`: `None < Some(_)` and `Ok(_) < Err(_)`. https://doc.rust-lang.org/std/option/index.html#comparison-operators and https://doc.rust-lang.org/std/result/index.html#comparison-operators

## Newer Option and Result methods

| Method | Since | Use |
| :-- | :-- | :-- |
| `is_some_and(f)`, `is_ok_and(f)`, `is_err_and(f)` | 1.70 | Predicate on the contents https://doc.rust-lang.org/std/option/enum.Option.html#method.is_some_and |
| `is_none_or(f)` | 1.82 | True if `None` or predicate holds https://doc.rust-lang.org/std/option/enum.Option.html#method.is_none_or |
| `inspect`, `inspect_err` | 1.76 | Log without consuming https://doc.rust-lang.org/std/result/enum.Result.html#method.inspect_err |
| `Option::take_if(f)` | 1.80 | Take the value when `f(&mut T)` is true https://doc.rust-lang.org/std/option/enum.Option.html#method.take_if |
| `get_or_insert_default` | 1.83 | Lazy default slot https://doc.rust-lang.org/std/option/enum.Option.html#method.get_or_insert_default |
| `Option::as_slice` | 1.75 | `&[T]` of length 0 or 1 https://doc.rust-lang.org/std/option/enum.Option.html#method.as_slice |
| `Result::flatten` | 1.89 | `Result<Result<T, E>, E>` to `Result<T, E>` https://doc.rust-lang.org/std/result/enum.Result.html#method.flatten |

Nightly only on 1.99.1, do not suggest: `Option::zip_with`, `get_or_try_insert_with`, `Result::into_ok`, `error::Report`, `Error::provide`, `dyn Error::sources`. https://doc.rust-lang.org/std/error/index.html

## Writing an Error type

- `Error` needs only `Debug` and `Display`. The `impl Error` body can be empty. https://doc.rust-lang.org/std/error/trait.Error.html#example
- Error messages are typically concise lowercase sentences without trailing punctuation, as in `"invalid digit found in string"`. https://doc.rust-lang.org/std/error/trait.Error.html
- A wrapper error exposes its cause through `source()` OR prints it in `Display`, never both. https://doc.rust-lang.org/std/error/trait.Error.html#error-source

```rust
#[derive(Debug)]
struct ConfigError { path: PathBuf, source: io::Error }
impl fmt::Display for ConfigError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "unable to read configuration at {}", self.path.display())
    }
}
impl Error for ConfigError {
    fn source(&self) -> Option<&(dyn Error + 'static)> { Some(&self.source) }
}
```

https://doc.rust-lang.org/std/error/trait.Error.html#error-source

- `Box<dyn Error>` converts from `&str` and `String` via `From`, so `Err("bad input".into())` works. https://doc.rust-lang.org/std/error/trait.Error.html#implementors
- Inspect causes with `source()` and `downcast_ref::<T>()` on `dyn Error`. https://doc.rust-lang.org/std/error/trait.Error.html#implementations

## main, ExitCode and panics

- For plain success or failure, return `Result<(), E>` from `main`. The error is printed and the exit code is failure. https://doc.rust-lang.org/std/process/struct.ExitCode.html
- For a specific code, return `ExitCode::from(42)` instead of calling `process::exit`, which skips destructors. https://doc.rust-lang.org/std/process/struct.ExitCode.html

```rust
fn main() -> ExitCode {
    if !check_foo() { return ExitCode::from(42); }
    ExitCode::SUCCESS
}
```

- `ExitCode` values are not portable (platforms mask bits). It has no `Eq` and no raw value access. https://doc.rust-lang.org/std/process/struct.ExitCode.html#portability
- A panic in the main thread ends the process with exit code 101. https://doc.rust-lang.org/std/macro.panic.html#current-implementation
- Edition 2021+: `panic!` always takes a format string. Use `std::panic::panic_any(x)` for a non-string payload. https://doc.rust-lang.org/std/macro.panic.html#editions
- A panic payload holds either `&str` or `String`, unspecified which. Check both when downcasting in a hook. https://doc.rust-lang.org/std/macro.panic.html

## Old patterns

| Old | Current |
| :-- | :-- |
| `Error::description()` (deprecated 1.42) | `Display` / `to_string()` https://doc.rust-lang.org/std/error/trait.Error.html#method.description |
| `Error::cause()` (deprecated 1.33) | `Error::source()` https://doc.rust-lang.org/std/error/trait.Error.html#method.cause |
| `std::panic::PanicInfo` in a hook | `PanicHookInfo` https://doc.rust-lang.org/std/panic/index.html |
| `match` and early `return Err` | `?`, also on `Option` in a function returning `Option` https://doc.rust-lang.org/std/option/index.html#the-question-mark-operator- |
| `process::exit(code)` at the end of `main` | return `ExitCode` https://doc.rust-lang.org/std/process/struct.ExitCode.html |
