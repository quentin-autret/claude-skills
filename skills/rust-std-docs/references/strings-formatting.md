# String, str, char, fmt and FFI strings (std 1.99.1)

## Contents
- String and str
- Case and comparison
- Format strings
- Display and Debug impls
- OsString and CString
- Old patterns

## String and str

- `len()` counts bytes, not chars. `&s[i..j]` takes byte indices and panics off a char boundary. Use `s.get(i..j)` for an `Option`, `chars().nth(n)` for the nth char, `char_indices()` for char plus byte offset. https://doc.rust-lang.org/std/string/struct.String.html
- `find` and `rfind` return byte indices. https://doc.rust-lang.org/std/primitive.str.html#method.find
- `truncate`, `split_off`, `split_at`, `drain`, `replace_range`, `insert`, `remove` panic off a char boundary. `split_at_checked` does not panic. https://doc.rust-lang.org/std/string/struct.String.html
- To cut at a byte budget, use `s.floor_char_boundary(max)` (1.91.0) and slice there. It can still split an emoji sequence. https://doc.rust-lang.org/std/primitive.str.html

```rust
let cut = &s[..s.floor_char_boundary(max_bytes)];
```

- `String::insert`, `insert_str` and `remove` shift the tail. Calling them in a loop is quadratic. https://doc.rust-lang.org/std/string/struct.String.html#method.remove
- `chars()` yields Unicode scalar values, not grapheme clusters. std has no grapheme iteration. https://doc.rust-lang.org/std/primitive.str.html#method.chars
- `split(' ')` on repeated spaces yields empty strings. Use `split_whitespace()` (Unicode) or `split_ascii_whitespace()`. https://doc.rust-lang.org/std/primitive.str.html#method.split
- `lines()` does not treat a lone `\r` as a line break. https://doc.rust-lang.org/std/primitive.str.html#method.lines
- `String::from_utf8_lossy` returns `Cow<str>`, borrowed when input is valid. https://doc.rust-lang.org/std/string/struct.String.html#method.from_utf8_lossy
- `s + &t` consumes `s` and reuses its buffer. `a.clone_from(&b)` can reuse `a`'s allocation. https://doc.rust-lang.org/std/string/struct.String.html
- Creating a non-UTF-8 `str` through `*_unchecked` is not immediate UB, but any later function that relies on the `str` invariant can lead to UB. https://doc.rust-lang.org/std/primitive.str.html#invariant

## Case and comparison

- Prefer `str::to_lowercase` over mapping `char::to_lowercase`. The `str` version handles Greek final sigma. Neither is locale aware. https://doc.rust-lang.org/std/primitive.char.html#notes-on-context-and-locale
- `eq_ignore_ascii_case` ignores only ASCII case: `"Ferrös"` and `"FERRÖS"` are not equal. https://doc.rust-lang.org/std/primitive.str.html
- `str` ordering is by bytes (code point order), not alphabetical. https://doc.rust-lang.org/std/primitive.str.html
- `char::to_digit(radix)` panics if radix is below 2 or above 36. `encode_utf8` panics if the buffer is too small for the char (4 bytes always suffice). https://doc.rust-lang.org/std/primitive.char.html

## Format strings

- Capture variables inline: `format!("{x} + {y}")`. The name must be a variable in scope. https://doc.rust-lang.org/std/fmt/index.html#named-parameters
- Explicit positions do not advance the implicit counter: `format!("{1} {} {0} {}", 1, 2)` is `"2 1 1 2"`. https://doc.rust-lang.org/std/fmt/index.html#positional-parameters
- `{:.*}` takes 2 arguments, precision first. Precision is decimals for floats, a max length (truncation) for strings, and ignored for integers. https://doc.rust-lang.org/std/fmt/index.html#precision
- Width or precision by name: `{:width$}`, `{:.prec$}`. The value must be `usize`. https://doc.rust-lang.org/std/fmt/index.html#width
- The `0` flag is sign aware and the `#` prefix counts in the width: `format!("{:#010x}", 27)` is `"0x0000001b"`, `format!("{:05}", -5)` is `"-0005"`. https://doc.rust-lang.org/std/fmt/index.html#signz0
- When precision truncates digits, rounding is half to even. https://doc.rust-lang.org/std/fmt/index.html#precision
- Padding and alignment are generally not applied to `Debug`. Format to a `String` first: https://doc.rust-lang.org/std/fmt/index.html#fillalignment

```rust
println!("[{:^15}]", format!("{:?}", Some("hi")));
```

- Write into an existing buffer with `write!` instead of building a temporary `String` with `format!`. With both traits in scope, import them as `_`: https://doc.rust-lang.org/std/macro.write.html

```rust
use std::fmt::Write as _;
use std::io::Write as _;
let mut s = String::new();
write!(&mut s, "{} {}", "abc", 123)?;
```

- For one value, `x.to_string()` is enough. For literals, `concat!` gives a `&'static str`. https://doc.rust-lang.org/std/macro.format.html
- Accept `fmt::Arguments` in logging helpers and call them with `format_args!`, with no heap allocation. https://doc.rust-lang.org/std/fmt/index.html#format_args

## Display and Debug impls

- Prefer implementing `Display` over `ToString`. `to_string()` comes for free. https://doc.rust-lang.org/std/fmt/trait.Display.html
- Implement `Debug` for all public types, usually by derive. Implement `Display` only when there is one obvious user-facing form. For alternatives, expose adapter types like `Path::display`. https://doc.rust-lang.org/std/fmt/trait.Display.html
- An ad hoc adapter: `fmt::from_fn(|f| write!(f, ..))` (`Display` for `FromFn` since 1.93.0). https://doc.rust-lang.org/std/fmt/trait.Display.html
- If `Display` is machine-parseable, implement `FromStr` for the same format. https://doc.rust-lang.org/std/fmt/trait.Display.html
- A manual `Debug` uses `f.debug_struct("P").field("x", &self.x).finish()`, which also gives `{:#?}` for free. https://doc.rust-lang.org/std/fmt/trait.Debug.html
- `Debug` output of std types and derives is not stable across Rust versions. Do not parse it or compare snapshots of it. https://doc.rust-lang.org/std/fmt/trait.Debug.html#stability
- Return `Err` from `fmt` only when the `Formatter` failed, and propagate it with `?`. `format!` panics if an impl returns an error. https://doc.rust-lang.org/std/fmt/index.html#formatting-traits
- `write!(f, ..)` in an impl ignores width and flags. To honor them, use `f.pad(s)` for text and `f.pad_integral(..)` for numbers. https://doc.rust-lang.org/std/fmt/index.html#formatting-traits

## OsString and CString

- `OsString` and `Path` may be non-UTF-8. `to_str()` returns `Option`, `to_string_lossy()` returns `Cow<str>`. Print with `os_str.display()` (1.87.0). https://doc.rust-lang.org/std/ffi/struct.OsString.html
- `as_encoded_bytes` is an unspecified encoding. Bytes sent over a network or stored in a file must not be rebuilt with `from_encoded_bytes_unchecked`. On Windows convert with `encode_wide`, on Unix with `as_bytes`. https://doc.rust-lang.org/std/ffi/struct.OsString.html
- `CString::new(..)` fails on an interior NUL and appends the trailing NUL itself. For a literal, use `CString::from(c"text")`. https://doc.rust-lang.org/std/ffi/struct.CString.html#method.new
- `CString::new(..).unwrap().as_ptr()` in one expression dangles because the temporary drops at the end of the statement. Bind the `CString` first. https://doc.rust-lang.org/std/ffi/struct.CString.html

```rust
let c = CString::new(name).expect("name should not contain NUL");
unsafe { ffi_call(c.as_ptr()) };
```

- The `as_ptr` buffer is read-only, even for C. `into_raw` must be freed by `CString::from_raw`, never C `free()`, and C must not change the length. Borrow foreign strings as `CStr`. https://doc.rust-lang.org/std/ffi/struct.CString.html#method.from_raw
- Never assume C strings are UTF-8. https://doc.rust-lang.org/std/ffi/index.html

## Old patterns

| Old | Current |
| :-- | :-- |
| `trim_left`, `trim_right` (deprecated 1.33) | `trim_start`, `trim_end` https://doc.rust-lang.org/std/primitive.str.html |
| `lines_any` (deprecated) | `lines` https://doc.rust-lang.org/std/primitive.str.html |
| `slice_unchecked` (deprecated) | `get_unchecked(a..b)` https://doc.rust-lang.org/std/primitive.str.html |
| `AsciiExt` trait (deprecated) | inherent `to_ascii_uppercase`, `eq_ignore_ascii_case` https://doc.rust-lang.org/std/primitive.str.html |
| `Formatter::flags()` (deprecated) | `sign_plus`, `alternate`, `sign_aware_zero_pad` https://doc.rust-lang.org/std/fmt/struct.Formatter.html |
| `impl ToString` | `impl Display` https://doc.rust-lang.org/std/fmt/trait.Display.html |
| `format!("{}", x)` with positional args | `format!("{x}")` https://doc.rust-lang.org/std/fmt/index.html#named-parameters |
