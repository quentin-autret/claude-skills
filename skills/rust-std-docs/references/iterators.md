# Iterators (std 1.99.1)

## Contents
- Writing chains
- Panics and non-termination
- Peekable and builders
- Implementing an iterator
- Not stable yet
- Old patterns

## Writing chains

- Loop for side effects with `for`, not with `map`. A bare `map` does nothing and only warns. `for_each` reads well at the end of a long chain but cannot `break` or `continue`. https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.for_each
- Clone late: put `.cloned()` after `.filter(..)` so only kept items are cloned. https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.cloned

```rust
let ones: Vec<String> = a.iter().filter(|s| s.len() == 1).cloned().collect();
```

- Write `flat_map(f)` instead of `map(f).flatten()`. https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.flatten
- `flatten` on `Option`/`Result` items drops `None` and `Err` silently. To fail on the first error, `collect::<Result<Vec<_>, _>>()`. https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.collect
- `by_ref()` takes a prefix without consuming the iterator: https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.by_ref

```rust
let head: Vec<_> = words.by_ref().take(2).collect();
let rest: Vec<_> = words.collect();
```

- `try_fold` short-circuits and leaves the rest in the iterator: `iter.try_fold(0i8, |acc, x| acc.checked_add(x))`. https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.try_fold
- Use `zip(0u32..)` instead of `enumerate()` when the counter is not `usize`. https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.enumerate
- Use `is_sorted`, `is_sorted_by`, `is_sorted_by_key` (1.82.0) to check order. https://doc.rust-lang.org/std/iter/trait.Iterator.html
- `min`/`max` need `Item: Ord`. https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.max
- `max` returns the last of equal maxima, `min` the first of equal minima. https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.max
- `FromIterator::from_iter` reads better than a turbofish when you must name the container: `VecDeque::from_iter(0..10)`. https://doc.rust-lang.org/std/iter/trait.FromIterator.html

## Panics and non-termination

- On an infinite iterator, `min`, `fold` and similar methods that traverse the whole iterator might not terminate, even when the answer is obvious: `repeat(1).min()` loops forever. https://doc.rust-lang.org/std/iter/index.html#infinity
- `sum()` and `product()` into integers panic on overflow when overflow checks are enabled. https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.sum
- `count()` and `enumerate()` do not guard against `usize` overflow. https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.count
- `step_by(0)` panics. The first element is always yielded. https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.step_by
- After the first `None`, `next` may yield `Some` again. Call `fuse()` when you need `None` forever. https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.fuse
- `map_while` consumes the element that stopped it. https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.map_while
- If an adapter panics, the iterator is left in an unspecified but memory safe state, not guaranteed to stay the same across Rust versions. Do not rely on the values it returns afterwards. https://doc.rust-lang.org/std/iter/index.html#adapters
- `size_hint` and `ExactSizeIterator::len` are hints that a buggy safe impl can break. Unsafe code must not trust them. `Chain` is not `ExactSizeIterator` on purpose. https://doc.rust-lang.org/std/iter/trait.ExactSizeIterator.html
- `next_back()` can yield different elements than `last()`. https://doc.rust-lang.org/std/iter/trait.DoubleEndedIterator.html#remarks

## Peekable and builders

- `peek()` over an iterator of references gives `Option<&&T>`. https://doc.rust-lang.org/std/iter/struct.Peekable.html#method.peek
- Consume conditionally with `next_if` / `next_if_eq` (1.51.0) instead of peek-then-next. https://doc.rust-lang.org/std/iter/struct.Peekable.html#method.next_if

```rust
while iter.next_if(|&x| x < 10).is_some() {}
```

- `next_if_map` (1.94.0) returns `Ok(r)` to consume or `Err(item)` to put the item back. If its closure panics, the item is lost. https://doc.rust-lang.org/std/iter/struct.Peekable.html#method.next_if_map

```rust
while let Some(d) = it.next_if_map(|c: char| c.to_digit(10).ok_or(c)) { n = n * 10 + d; }
```

- `iter::from_fn(move || ..)` builds an iterator from a closure. It is not fused. https://doc.rust-lang.org/std/iter/fn.from_fn.html
- `iter::successors(first, f)` chains values and stops at `None`, with checked arithmetic as a natural end: https://doc.rust-lang.org/std/iter/fn.successors.html

```rust
let powers = std::iter::successors(Some(1_u16), |n| n.checked_mul(10));
```

- `iter::repeat_n`, `once_with`, `repeat_with`, `iter::zip`, `iter::chain` are free functions in `std::iter`. https://doc.rust-lang.org/std/iter/index.html#functions

## Implementing an iterator

- Only `next` is required. Override `nth` (not `skip`) when you can jump faster. Implement `try_fold` explicitly when it can beat the default loop, by delegating to the inner parts. https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.try_fold
- Take any collection with `T: IntoIterator` and bound `T::Item` in a `where` clause. https://doc.rust-lang.org/std/iter/trait.IntoIterator.html
- `Extend` on a map updates existing keys. https://doc.rust-lang.org/std/iter/trait.Extend.html

## Not stable yet

Nightly only on 1.99.1: `intersperse`, `map_windows`, `array_chunks`, `next_chunk`, `advance_by`, `try_collect`, `collect_into`, `try_find`, `try_reduce`, `is_partitioned`, `cmp_by`, `eq_by`. Write them with stable tools instead. https://doc.rust-lang.org/std/iter/trait.Iterator.html

## Old patterns

| Old | Current |
| :-- | :-- |
| `.map(|x| side_effect(x))` left unused | `for x in iter { .. }` https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.map |
| `map(f).flatten()` | `flat_map(f)` https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.flatten |
| `if let Some(x) = it.peek() { if cond { it.next(); } }` | `it.next_if(cond)` https://doc.rust-lang.org/std/iter/struct.Peekable.html#method.next_if |
