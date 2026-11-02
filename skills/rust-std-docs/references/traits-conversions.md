# Conversions, Clone/Copy/Default, comparison, Drop, Deref, Any (std 1.99.1)

## Contents
- From, Into, TryFrom
- AsRef and Borrow
- Clone, Copy, Default
- PartialEq, PartialOrd, Ord
- Drop
- Deref and operators
- Any and PhantomData
- Old patterns

## From, Into, TryFrom

- Implement `From` (or `TryFrom`), never `Into` directly. The blanket impl gives `Into`. In generic bounds, ask for `Into<T>`. https://doc.rust-lang.org/std/convert/trait.From.html
- A `From` impl must be infallible, lossless, value-preserving and the one obvious conversion. A `From` that can panic belongs in `TryFrom`. Otherwise write a named method. https://doc.rust-lang.org/std/convert/trait.From.html#when-to-implement-from
- `From` impls on your error enum let `?` convert errors: https://doc.rust-lang.org/std/convert/trait.From.html#examples

```rust
impl From<io::Error> for CliError {
    fn from(e: io::Error) -> Self { CliError::Io(e) }
}
```

- `as` truncates silently: `1_000_000_000_000i64 as i32` is `-727379968`. Use `i32::try_from(v)` when loss must be detected. https://doc.rust-lang.org/std/convert/trait.TryFrom.html#examples
- `TryFrom`, `TryInto` and `FromIterator` are in the prelude only from edition 2021. Import them in 2015/2018. https://doc.rust-lang.org/std/prelude/index.html#prelude-contents

## AsRef and Borrow

- `AsRef` is a cheap, infallible reference conversion. Accept `impl AsRef<str>` or `impl AsRef<Path>` to take owned and borrowed forms. https://doc.rust-lang.org/std/convert/trait.AsRef.html
- Do not call `as_ref()` just to deref a `Box` or smart pointer. Use Deref coercion (`&x`). https://doc.rust-lang.org/std/convert/trait.AsRef.html#generic-implementations
- `AsRef` has no blanket reflexive impl (`AsRef<T> for T`), because it would overlap the auto-dereferencing blanket impl for `&T`. https://doc.rust-lang.org/std/convert/trait.AsRef.html#reflexivity
- `Borrow` additionally requires `Hash`, `Eq` and `Ord` to match the owned form. To expose one field, implement `AsRef`, not `Borrow`. https://doc.rust-lang.org/std/convert/trait.AsRef.html#relation-to-borrow

## Clone, Copy, Default

- `#[derive(Clone)]` on a struct holding `Arc<Mutex<T>>` shares the state with the original. https://doc.rust-lang.org/std/clone/trait.Clone.html
- Derives add bounds on every type parameter. `#[derive(Clone, Copy)] struct Generate<T>(fn() -> T)` then gets unnecessary `T: Clone` and `T: Copy` bounds, so `Generate(f).clone()` fails when `T` is not `Clone`. Implement by hand instead: https://doc.rust-lang.org/std/clone/trait.Clone.html#how-can-i-implement-clone

```rust
impl<T> Copy for Generate<T> {}
impl<T> Clone for Generate<T> { fn clone(&self) -> Self { *self } }
```

- `Copy` is public API. Removing it later is breaking, so omit it if the type may grow. A type with `Drop` cannot be `Copy`. https://doc.rust-lang.org/std/marker/trait.Copy.html#when-should-my-type-be-copy
- `a.clone_from(&b)` can reuse `a`'s allocation. https://doc.rust-lang.org/std/clone/trait.Clone.html#method.clone_from
- `#[derive(Default)]` on an enum needs `#[default]` on a unit variant (1.62.0). https://doc.rust-lang.org/std/default/trait.Default.html#enums

```rust
#[derive(Default)]
enum Kind { #[default] A, B }
let opts = Options { verbose: true, ..Default::default() };
```

## PartialEq, PartialOrd, Ord

- Derive all of `PartialEq, Eq, PartialOrd, Ord` or implement all by hand. Mixing them makes them disagree. https://doc.rust-lang.org/std/cmp/trait.Ord.html#how-can-i-implement-ord
- Put the logic in `Ord::cmp` and have `partial_cmp` return `Some(self.cmp(other))`. https://doc.rust-lang.org/std/cmp/trait.PartialOrd.html

```rust
impl Ord for Character {
    fn cmp(&self, o: &Self) -> Ordering {
        self.experience.cmp(&o.experience).then(self.health.cmp(&o.health))
    }
}
impl PartialOrd for Character {
    fn partial_cmp(&self, o: &Self) -> Option<Ordering> { Some(self.cmp(o)) }
}
```

- `PartialEq` and `PartialOrd` must compare the same fields: `a == b` iff `partial_cmp == Some(Equal)`. https://doc.rust-lang.org/std/cmp/trait.PartialOrd.html#examples-of-incorrect-partialord-implementations
- Never implement `Ord` with `<`/`>` on floats. Use `f64::total_cmp`. For sorting by a field, `sort_by_key` is simpler than an `Ord` impl. https://doc.rust-lang.org/std/cmp/trait.Ord.html#examples-of-incorrect-ord-implementations
- Derived ordering is field order for structs, and discriminant first for enums. https://doc.rust-lang.org/std/cmp/trait.Ord.html#derivable
- `Ord::max` returns the second argument on ties, `min` the first. https://doc.rust-lang.org/std/cmp/trait.Ord.html#method.max
- Implement `PartialEq<Foreign> for Local`, never `PartialEq<Local> for Foreign`. https://doc.rust-lang.org/std/cmp/trait.PartialEq.html#cross-crate-considerations
- With `Clone` and `PartialEq`, `x == x` must imply `x.clone() == x`. `HashMap`, `HashSet`, `BTreeMap`, `BTreeSet` and `BinaryHeap` rely on it for their keys. https://doc.rust-lang.org/std/clone/trait.Clone.html#clone-and-partialeqeq

## Drop

- Call `drop(x)` (prelude `mem::drop`), never `x.drop()` (E0040). https://doc.rust-lang.org/std/ops/trait.Drop.html#you-cannot-call-dropdrop-yourself
- Struct fields drop in declaration order. Locals drop in reverse declaration order. https://doc.rust-lang.org/std/ops/trait.Drop.html#drop-order
- Do not panic in `drop`. It may run during unwinding, and a double panic aborts. https://doc.rust-lang.org/std/ops/trait.Drop.html#panics
- An `impl Drop` on `S<'a>` forces `'a` to be live at drop, which can reject code that compiled before. https://doc.rust-lang.org/std/ops/trait.Drop.html#drop-check

## Deref and operators

- Implement `Deref` only for smart-pointer-like wrappers: cheap, infallible, unsurprising. Not when the deref could fail, when methods may collide with the target's, or when deref coercion should not be public API. https://doc.rust-lang.org/std/ops/trait.Deref.html#when-to-implement-deref-or-derefmut
- A generic `Deref` wrapper (like `Box<T>`) should provide few or no methods, since any could collide with the target's. https://doc.rust-lang.org/std/ops/trait.Deref.html#when-to-implement-deref-or-derefmut
- Bound on `Fn` only when the closure is called repeatedly without mutating state (e.g. concurrently). Otherwise use `FnMut` or `FnOnce`. `Fn` the trait is not `fn` the pointer type. https://doc.rust-lang.org/std/ops/trait.Fn.html
- For operator traits, implement both `Add<T>` and `Add<&T>` so callers avoid clones. `&&` and `||` are currently not overloadable. https://doc.rust-lang.org/std/ops/index.html
- `AsyncFn`, `AsyncFnMut`, `AsyncFnOnce` sit beside `Fn*` in `std::ops` and the prelude. https://doc.rust-lang.org/std/ops/index.html#traits

## Any and PhantomData

- `dyn Any` tests a concrete type only, not a trait. https://doc.rust-lang.org/std/any/index.html
- `boxed.type_id()` on a `Box<dyn Any>` gives the box's id. Use `(&*boxed).type_id()`. https://doc.rust-lang.org/std/any/index.html#smart-pointers-and-dyn-any
- `PhantomData<T>` is zero-sized. Use `PhantomData<&'a T>` for a struct holding raw pointers tied to `'a`. https://doc.rust-lang.org/std/marker/struct.PhantomData.html#unused-lifetime-parameters

## Old patterns

| Old | Current |
| :-- | :-- |
| `impl Into<External> for Local` | `impl From<Local> for External` (since 1.41) https://doc.rust-lang.org/std/convert/trait.Into.html |
| `x as i32` where loss matters | `i32::try_from(x)` https://doc.rust-lang.org/std/convert/trait.TryFrom.html#examples |
| `boxed.as_ref()` to deref | `&boxed` https://doc.rust-lang.org/std/convert/trait.AsRef.html#generic-implementations |
| Float `Ord` with `<` | `total_cmp` https://doc.rust-lang.org/std/cmp/trait.Ord.html#examples-of-incorrect-ord-implementations |
