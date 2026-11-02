# Raw pointers, MaybeUninit, transmute and numbers (std 1.99.1)

## Contents
- Pointer validity
- Provenance
- NonNull and from_raw_parts
- MaybeUninit
- transmute and its replacements
- Integer overflow
- Floats
- Hints and benchmarks
- Old patterns

## Pointer validity

- Ask "is this pointer valid for this access", not "is this pointer valid". Null is never valid for reads or writes. https://doc.rust-lang.org/std/ptr/index.html#safety
- Alignment is required even for zero-size accesses. Use `NonNull::dangling()` as an aligned placeholder. Only `read_unaligned` and `write_unaligned` skip it. https://doc.rust-lang.org/std/ptr/index.html#alignment
- `&*ptr` needs aligned, non-null, dereferenceable memory holding a valid `T`, even if the reference is never used. https://doc.rust-lang.org/std/ptr/index.html#pointer-to-reference-conversion
- Once you derive a raw pointer from a reference, do not access the memory through references while the pointer is in use. https://doc.rust-lang.org/std/ptr/index.html
- `ptr` accesses are non-atomic. Concurrent access is UB unless all reads. Writing to a `const` or a non-interior-mutable `static` is UB. https://doc.rust-lang.org/std/ptr/index.html#allocation
- Every stack variable is its own allocation. `offset` across allocations and `offset_from` between allocations are UB. https://doc.rust-lang.org/std/ptr/index.html#provenance
- Prefer `offset`/`add` over `wrapping_offset` in hot code. A wrapping offset keeps its original allocation, so it never reaches another one. https://doc.rust-lang.org/std/primitive.pointer.html#method.wrapping_offset

## Provenance

- Keep a real pointer and edit its address with `addr()`, `with_addr()`, `map_addr()`. A pointer rebuilt from `ptr.addr()` has no provenance and is UB to deref. https://doc.rust-lang.org/std/primitive.pointer.html#method.addr

```rust
let tagged = ptr.map_addr(|a| a | 0x1);
let data = unsafe { *tagged.map_addr(|a| a & !0x1) };
```

https://doc.rust-lang.org/std/ptr/index.html#using-strict-provenance

- Use `expose_provenance` / `with_exposed_provenance` only when an integer address is unavoidable (MMIO, legacy APIs). https://doc.rust-lang.org/std/ptr/index.html#exposed-provenance
- Store pointers in `AtomicPtr`, not `AtomicUsize`. Store "pointer or bare integer" as a pointer, using `without_provenance` for the integer. https://doc.rust-lang.org/std/ptr/index.html#using-strict-provenance

## NonNull and from_raw_parts

- Use `NonNull<T>` only for non-null plus covariance. "If you are not sure, just use `*mut T`." Add `PhantomData<Cell<T>>` when covariance would be unsound. https://doc.rust-lang.org/std/ptr/struct.NonNull.html
- A `NonNull` built `From<&T>` does not allow mutation. https://doc.rust-lang.org/std/ptr/struct.NonNull.html
- `NonNull::dangling()` is not a "not initialized" sentinel. Its address can equal a real one. https://doc.rust-lang.org/std/ptr/struct.NonNull.html#method.dangling
- `slice::from_raw_parts(ptr, len)`: `len` counts elements. The pointer must be non-null and aligned even for `len == 0`. All memory must be one initialized allocation. https://doc.rust-lang.org/std/slice/fn.from_raw_parts.html#safety
- Never glue 2 adjacent slices with `from_raw_parts`. They may be different allocations. https://doc.rust-lang.org/std/slice/fn.from_raw_parts.html#incorrect-usage
- From FFI, check null first: https://doc.rust-lang.org/std/slice/fn.from_raw_parts.html#ffi-handling-null-pointers

```rust
let data: &[u8] = if ptr.is_null() { &[] } else { unsafe { slice::from_raw_parts(ptr, len) } };
```

- Tie the returned lifetime to a safe source through a helper signature. https://doc.rust-lang.org/std/slice/fn.from_raw_parts.html#caveat

## MaybeUninit

- `mem::zeroed()` on a reference, or `assume_init` on uninitialized `bool` or even `i32`, is instant UB. https://doc.rust-lang.org/std/mem/union.MaybeUninit.html#initialization-invariant
- Dropping a `MaybeUninit<T>` never drops `T`. Call `assume_init` or `assume_init_drop`. https://doc.rust-lang.org/std/mem/union.MaybeUninit.html#method.new
- Prefer `assume_init` over `assume_init_read`, which copies bitwise. https://doc.rust-lang.org/std/mem/union.MaybeUninit.html#method.assume_init_read
- Init a struct field by field through `&raw mut`, never through `&mut` to uninit data: https://doc.rust-lang.org/std/mem/union.MaybeUninit.html#initializing-a-struct-field-by-field

```rust
let mut u = MaybeUninit::<Foo>::uninit();
let p = u.as_mut_ptr();
unsafe { (&raw mut (*p).name).write("Bob".to_string()); }
unsafe { (&raw mut (*p).list).write(vec![0, 1, 2]); }
let foo = unsafe { u.assume_init() };
```

- Uninitialized array: `[const { MaybeUninit::<T>::uninit() }; N]`, then `write` each element. https://doc.rust-lang.org/std/mem/union.MaybeUninit.html#initializing-an-array-element-by-element
- `Option<MaybeUninit<T>>` loses the niche and can be larger than `Option<T>`. https://doc.rust-lang.org/std/mem/union.MaybeUninit.html#layout

## transmute and its replacements

`transmute` is "the absolute last resort". Use these instead. https://doc.rust-lang.org/std/mem/fn.transmute.html#alternatives

| Instead of transmuting | Write |
| :-- | :-- |
| `[u8; 4]` to `u32` | `u32::from_le_bytes` / `from_be_bytes` / `from_ne_bytes` |
| pointer to `usize` | `ptr.addr()` |
| `*mut T` to `&mut T` | `&mut *ptr` |
| `&mut T` to `&mut U` | `&mut *(r as *mut T as *mut U)` |
| `&str` to `&[u8]` | `s.as_bytes()` |
| `Vec<&T>` to `Vec<Option<&T>>` | `v.into_iter().map(Some).collect()` |

- Integer to pointer by `transmute` is likely not equivalent to an `as` cast, and non-zero-sized accesses through such a pointer are UB. Transmuting `Vec<T>` relies on unspecified layout. https://doc.rust-lang.org/std/mem/fn.transmute.html#transmutation-between-pointers-and-integers

## Integer overflow

- Plain `+` can panic in debug mode. Say what you mean: `checked_*` (Option), `strict_*` (always panics, 1.91.0), `saturating_*`, `wrapping_*`, `overflowing_*`. https://doc.rust-lang.org/std/primitive.i32.html#method.strict_add

```rust
let total = a.checked_add(b).ok_or(Error::Overflow)?;
let hash = h.wrapping_mul(31).wrapping_add(c);
```

- Use `Wrapping<T>` when modular arithmetic is the intent. https://doc.rust-lang.org/std/num/struct.Wrapping.html
- `Option<NonZero<u32>>` is the size of `u32`. Build with `NonZero::new(n)` (returns `Option`). `new_unchecked(0)` is UB. https://doc.rust-lang.org/std/num/struct.NonZero.html
- `i32::cast_unsigned()` reinterprets bits explicitly. `checked_cast_unsigned` (returns `None` on negatives) and `strict_cast_unsigned` are nightly-only. https://doc.rust-lang.org/std/primitive.i32.html#method.cast_unsigned
- Parsing and conversion errors live in `std::num`: `ParseIntError`, `TryFromIntError`, `IntErrorKind`. https://doc.rust-lang.org/std/num/index.html

## Floats

- Sort or order floats with `total_cmp`. https://doc.rust-lang.org/std/primitive.f64.html#method.total_cmp
- `f64::max`/`min` return the non-NaN operand. With `+0.0` and `-0.0`, either may come back. https://doc.rust-lang.org/std/primitive.f64.html#method.max
- `f64::clamp` panics if `min > max` or a bound is NaN. https://doc.rust-lang.org/std/primitive.f64.html#method.clamp
- `powf`, `ln` and other transcendental functions have non-deterministic precision across platforms and Rust versions. Never compare their results exactly. `powf(NaN, 0.0)` is `1.0`. https://doc.rust-lang.org/std/primitive.f64.html#method.powf

## Hints and benchmarks

- In benchmarks wrap inputs and the result in `std::hint::black_box`. It is best effort, never for correctness or constant-time crypto. https://doc.rust-lang.org/std/hint/fn.black_box.html

```rust
let _ = black_box(increment(black_box(5)));
```

- Stable hints: `assert_unchecked`, `cold_path`, `select_unpredictable`, `spin_loop`, `unreachable_unchecked`. `likely`/`unlikely` are nightly. https://doc.rust-lang.org/std/hint/index.html
- `debug_assert!` runs only in non-optimized builds unless `-C debug-assertions` is set. Use it for checks too expensive for release builds. https://doc.rust-lang.org/std/macro.debug_assert.html

## Old patterns

| Old | Current |
| :-- | :-- |
| `mem::uninitialized()` / `mem::zeroed()` where zeroed or uninitialized bits are invalid for the type | `MaybeUninit` https://doc.rust-lang.org/std/mem/union.MaybeUninit.html#initialization-invariant |
| `ptr as usize` then back `as *const T` | `addr` / `with_addr` / `map_addr` https://doc.rust-lang.org/std/ptr/index.html#strict-provenance |
| `AtomicUsize` holding a pointer | `AtomicPtr` https://doc.rust-lang.org/std/ptr/index.html#using-strict-provenance |
| `MaybeUninit::uninit().assume_init()` array trick | `[const { MaybeUninit::uninit() }; N]` https://doc.rust-lang.org/std/mem/union.MaybeUninit.html#initializing-an-array-element-by-element |
