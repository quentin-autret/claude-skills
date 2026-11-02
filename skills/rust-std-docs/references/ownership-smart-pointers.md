# Box, Rc, Weak, cells, Cow, mem and Pin (std 1.99.1)

## Contents
- Box
- Rc and Weak
- Cell, RefCell, OnceCell, LazyCell
- Cow and mem
- Pin
- Old patterns

## Box

- `Box<T>` for sized `T` is one pointer, ABI-compatible with C `T*`. A nullable C pointer maps to `Option<Box<T>>`. https://doc.rust-lang.org/std/boxed/index.html#memory-layout
- Only put pointers from the global allocator into a `Box`. Never wrap an arbitrary C pointer. For functions defined in C, keep raw pointers in the Rust signature, not `Box`. https://doc.rust-lang.org/std/boxed/index.html#memory-layout
- `Box::leak` is for data that lives for the rest of the program. If the memory must be freed later, prefer `Box::into_raw`. Rebuilding a `Box` from the leaked reference is a grey area, avoid it. https://doc.rust-lang.org/std/boxed/struct.Box.html#method.leak
- Box aliasing rules match `&mut T`: raw pointers derived from a box must not be used after the box is mutated through, moved or borrowed as `&mut T`. https://doc.rust-lang.org/std/boxed/index.html#considerations-for-unsafe-code
- Before edition 2024, `boxed_slice.into_iter()` yields references. Write `.iter()` or `IntoIterator::into_iter(b)` explicitly. https://doc.rust-lang.org/std/boxed/index.html#editions
- Uninitialized box: https://doc.rust-lang.org/std/boxed/struct.Box.html

```rust
let mut five = Box::<u32>::new_uninit(); // 1.82.0
five.write(5);
let five = unsafe { five.assume_init() };
```

## Rc and Weak

- Write `Rc::clone(&rc)`, not `rc.clone()`, to show that only the pointer is copied. https://doc.rust-lang.org/std/rc/index.html#cloning-references
- Rc methods are associated functions (`Rc::get_mut(&mut rc)`, `Rc::downgrade(&rc)`) to avoid clashes with `T`'s methods. https://doc.rust-lang.org/std/rc/struct.Rc.html
- `Rc` is not `Send`. Use `Arc` across threads. https://doc.rust-lang.org/std/rc/index.html
- `Rc` cycles leak. Point parent to child with `Rc` and child to parent with `Weak`. `upgrade()` returns `Option<Rc<T>>`. https://doc.rust-lang.org/std/rc/index.html#examples
- A self-referencing node: `Rc::new_cyclic(|me| Node { me: me.clone() })` (1.60.0). Inside the closure `upgrade()` is `None`. https://doc.rust-lang.org/std/rc/struct.Rc.html#method.new_cyclic
- `Rc::get_mut` is `None` if any other `Rc` OR `Weak` exists. `Rc::make_mut` clones only if other `Rc` exist. With only `Weak` left, it disassociates them (their `upgrade()` turns `None`). https://doc.rust-lang.org/std/rc/struct.Rc.html#method.make_mut

```rust
let mut data = Rc::new(5);
*Rc::make_mut(&mut data) += 1;          // no clone
let other = Rc::clone(&data);
*Rc::make_mut(&mut data) += 1;          // clones inner data
```

- `Rc::unwrap_or_clone(rc)` (1.76.0) takes the value without cloning when unique. `Rc::into_inner(rc)` (1.70.0) returns `Some` for exactly one of all clones. https://doc.rust-lang.org/std/rc/struct.Rc.html#method.unwrap_or_clone
- A `Weak` keeps the allocation alive (not the value). It is read only through `upgrade()`. `Weak::new()` always upgrades to `None`. https://doc.rust-lang.org/std/rc/struct.Weak.html

## Cell, RefCell, OnceCell, LazyCell

- Prefer inherited mutability. Interior mutability is a last resort for mutating inside shared pointers such as `Rc`, implementation details of logically-immutable methods (caches), and `Clone` impls. https://doc.rust-lang.org/std/cell/index.html#when-to-choose-interior-mutability
- `Cell<T>` suits simple types where copying or moving is cheap. `RefCell` has advantages for larger and non-`Copy` types. https://doc.rust-lang.org/std/cell/index.html#celltt
- `RefCell` checks borrows at runtime: `borrow()` panics while mutably borrowed, `borrow_mut()` panics while any borrow lives. Scope guards tightly. https://doc.rust-lang.org/std/cell/index.html#introducing-mutability-inside-of-something-immutable

```rust
{
    let mut map = shared.borrow_mut();
    map.insert("africa", 92388);
} // guard dropped before the next borrow()
let total: i32 = shared.borrow().values().sum();
```

- When a conflict is possible and a panic is not acceptable, use `try_borrow` / `try_borrow_mut`. https://doc.rust-lang.org/std/cell/struct.RefCell.html#method.try_borrow
- With `&mut RefCell<T>`, call `get_mut()` and skip the runtime check. https://doc.rust-lang.org/std/cell/struct.RefCell.html#method.get_mut
- `RefCell::replace`, `replace_with`, `take` panic while the cell is borrowed in any way. `clone`, `==` and `cmp` panic while it is mutably borrowed. https://doc.rust-lang.org/std/cell/struct.RefCell.html#method.replace
- `RefCell` is `Send` (if `T: Send`) but never `Sync`. https://doc.rust-lang.org/std/cell/struct.RefCell.html
- Cache in a `&self` method with `OnceCell`: `self.cache.get_or_init(|| self.compute())`. If the closure panics, the cell stays empty. Reentrant init panics. https://doc.rust-lang.org/std/cell/struct.OnceCell.html#method.get_or_init
- `LazyCell::new(|| ..)` (1.80.0) fixes one init closure. Use `OnceCell` when the init varies by call. https://doc.rust-lang.org/std/cell/index.html#lazycellt-f
- If a `LazyCell` init panics, the cell is poisoned for good. Every later access panics. https://doc.rust-lang.org/std/cell/struct.LazyCell.html#poisoning
- Thread-safe counterparts: `Mutex`/`RwLock`/atomics for `Cell`/`RefCell`, `OnceLock` for `OnceCell`, `LazyLock` for `LazyCell`. https://doc.rust-lang.org/std/cell/index.html

## Cow and mem

- `Cow::to_mut()` clones only if borrowed. `into_owned()` moves out without cloning when already owned. https://doc.rust-lang.org/std/borrow/enum.Cow.html
- For reference-counted pointers, `Rc::make_mut` / `Arc::make_mut` provide clone-on-write. https://doc.rust-lang.org/std/borrow/enum.Cow.html
- Move a field out of `&mut self` with `mem::take` (needs `Default`) or `mem::replace`. No clone needed. https://doc.rust-lang.org/std/mem/fn.take.html

```rust
fn get_and_reset(&mut self) -> Vec<T> { mem::take(&mut self.buf) }
```

## Pin

- Pin a `!Unpin` value on the heap with `Box::pin(v)`, an existing box with `Box::into_pin(b)`, on the stack with `std::pin::pin!`. https://doc.rust-lang.org/std/pin/struct.Pin.html#pinning-a-value-inside-a-box
- For an `Unpin` pointee, use the safe `Pin::new`, `Pin::get_mut`, `Pin::into_inner`, never the `unchecked` variants. https://doc.rust-lang.org/std/pin/struct.Pin.html#method.new_unchecked
- `Pin::new_unchecked` on `&mut T` breaks the pinning contract (the value can move after the `Pin` is dropped). It is also unsafe on `Rc<T>` (aliases escape the pinning) and on closure captures, unless the API documents that the closure is pinned. Pin with `pin!` in the outer function instead. https://doc.rust-lang.org/std/pin/struct.Pin.html#method.new_unchecked
- `Pin` pins the pointee, not the pointer. `Box<T>` is `Unpin`, which does not affect `Pin<Box<T>>`: only whether `T` is `Unpin` matters. A type that relies on pinning must include `PhantomPinned`. https://doc.rust-lang.org/std/pin/index.html#unpin
- For address-sensitive types, write `Drop` as if it took `Pin<&mut Self>`. `#[repr(packed)]` types cannot be pinned. https://doc.rust-lang.org/std/pin/index.html#implementing-drop-for-types-with-address-sensitive-states

## Old patterns

| Old | Current |
| :-- | :-- |
| `Rc::try_unwrap(x).ok()` (equivalent for `Rc` only) | `Rc::into_inner(x)` https://doc.rust-lang.org/std/rc/struct.Rc.html#method.into_inner |
| `if unique { unwrap } else { clone }` | `Rc::unwrap_or_clone` https://doc.rust-lang.org/std/rc/struct.Rc.html#method.unwrap_or_clone |
| `Pin::new_unchecked` on a capture | `pin!` in the caller https://doc.rust-lang.org/std/pin/struct.Pin.html#method.new_unchecked |
