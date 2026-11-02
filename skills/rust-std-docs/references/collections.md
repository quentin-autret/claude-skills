# Collections, slices and arrays (std 1.99.1)

## Contents
- Choosing a collection
- Vec
- Slice methods
- Arrays
- VecDeque and BinaryHeap
- HashMap and HashSet
- BTreeMap
- Hash
- Old patterns

## Choosing a collection

- Default to `Vec` or `HashMap`. Reach for `VecDeque` (queue), `BTreeMap` (key order, ranges, min/max key) or `BinaryHeap` (priority queue) only for those needs. `LinkedList` only when "absolutely certain". https://doc.rust-lang.org/std/collections/index.html
- Collections never shrink on their own. Call `shrink_to_fit` or `shrink_to` when memory matters. https://doc.rust-lang.org/std/collections/index.html
- HashMap iteration order is arbitrary. Never depend on it. https://doc.rust-lang.org/std/collections/index.html

## Vec

- `Vec::with_capacity(n)`, `vec![x; n]` and `vec![a, b]` request an allocation of the exact size for n elements (no power-of-two rounding). Growth strategy and drop order are not guaranteed. https://doc.rust-lang.org/std/vec/struct.Vec.html#capacity-and-reallocation
- A Vec never erases removed data. Do not rely on `clear` or drop for zeroing secrets, the buffer may be reused and zeroing optimized away. https://doc.rust-lang.org/std/vec/struct.Vec.html#capacity-and-reallocation
- `remove(i)` is O(n) and panics out of bounds. Use `swap_remove` (O(1), breaks order) or `VecDeque::pop_front` for queue-like removal. `insert(i, x)` panics if `i > len`. https://doc.rust-lang.org/std/vec/struct.Vec.html#method.remove
- `reserve_exact` may still over-allocate. Prefer `reserve` when more pushes follow. `try_reserve` returns `Err` instead of aborting and keeps contents. https://doc.rust-lang.org/std/vec/struct.Vec.html#method.reserve_exact
- `extract_if(range, pred)` (stable 1.87.0) removes and yields matching elements. If the iterator is not exhausted (dropped without iterating, or short-circuited), the remaining elements are retained, so unvisited matches are not removed. To just delete, use `extract_if(..).for_each(drop)`, or `retain_mut` with a negated predicate when no range is needed. https://doc.rust-lang.org/std/vec/struct.Vec.html#method.extract_if

```rust
let mut v = vec![2, 4, 6, 7];
let first = v.extract_if(.., |x| *x % 2 == 0).next(); // Some(2)
// v is [4, 6, 7]: 4 and 6 were never visited, so they stay
```
- `pop_if(pred)` (1.86.0) pops the last element only when the predicate holds. https://doc.rust-lang.org/std/vec/struct.Vec.html#method.pop_if
- `drain(range)` drops the unconsumed removed elements when the iterator is dropped early. If the iterator is leaked (`mem::forget`), the vec may lose and leak elements arbitrarily. `splice(range, iter)` removes the range even if the `Splice` is not consumed before it is dropped. https://doc.rust-lang.org/std/vec/struct.Vec.html#method.drain
- `into_flattened` (1.80.0) turns `Vec<[T; N]>` into `Vec<T>`. https://doc.rust-lang.org/std/vec/struct.Vec.html#method.into_flattened
- Nightly only on 1.99.1: `try_remove`, `push_within_capacity`, `try_shrink_to`. https://doc.rust-lang.org/std/vec/struct.Vec.html

## Slice methods

- Default to `sort_unstable` (in place, no allocation, usually faster). Use `sort` when stability matters or input is already partly sorted. https://doc.rust-lang.org/std/primitive.slice.html#method.sort
- If `Ord` is not a total order, `sort` may panic and the order is unspecified. https://doc.rust-lang.org/std/primitive.slice.html#method.sort
- Floats are only `PartialOrd`. Sort them with `total_cmp`, unstable like the default: https://doc.rust-lang.org/std/primitive.f64.html#method.total_cmp

```rust
v.sort_unstable_by(|a, b| a.total_cmp(b));
```

- `total_cmp` orders: negative NaN, -inf, negatives, -0.0, +0.0, positives, +inf, positive NaN. NaN sorts to an end by its sign bit, and -0.0 before +0.0 (unlike `PartialEq`). https://doc.rust-lang.org/std/primitive.f64.html#method.total_cmp

- `sort_by_cached_key` calls an expensive key function at most once per element, in unspecified order. https://doc.rust-lang.org/std/primitive.slice.html#method.sort_by_cached_key
- `binary_search` on unsorted data is meaningless. With duplicates it returns any match. To insert into a sorted Vec or find a run of equal items, use `partition_point`: https://doc.rust-lang.org/std/primitive.slice.html#method.binary_search

```rust
let idx = s.partition_point(|&x| x < num);
s.insert(idx, num);
```

- `windows(0)` and `chunks(0)` panic. There is no `windows_mut`. For a constant chunk size use `as_chunks` / `as_rchunks`. `chunk_by` (1.77.0) groups by a predicate on neighbours. https://doc.rust-lang.org/std/primitive.slice.html#method.windows
- `get_disjoint_mut([i, j])` (1.86.0) returns several `&mut` at once, with an error on overlap or out of bounds. https://doc.rust-lang.org/std/primitive.slice.html#method.get_disjoint_mut
- `copy_from_slice` / `clone_from_slice` panic when lengths differ. `split_at(mid)` panics if `mid > len`. https://doc.rust-lang.org/std/primitive.slice.html#method.split_at

## Arrays

- Slice to array: `<[u8; 4]>::try_from(&s[4..8])` or `s.try_into()`. It fails unless lengths match. https://doc.rust-lang.org/std/primitive.array.html
- `[expr; N]` needs `expr: Copy` or a `const`. `[expr; 0]` still evaluates `expr`. https://doc.rust-lang.org/std/primitive.array.html
- Editions 2015/2018: `array.into_iter()` yields references. From 2021 it yields values. In old editions write `IntoIterator::into_iter(array)`. https://doc.rust-lang.org/std/primitive.array.html#editions
- `[T; N]::map` is eager and can use much stack. If you do not need an array result, use `arr.into_iter().map(f)`. https://doc.rust-lang.org/std/primitive.array.html#method.map

## VecDeque and BinaryHeap

- `VecDeque` is a ring buffer, not contiguous. `as_slices()` returns 2 slices. Call `make_contiguous()` before sorting or slicing it. https://doc.rust-lang.org/std/collections/struct.VecDeque.html
- `VecDeque::extract_if` is nightly only. `retain_back(len)` is stable since 1.99.0. https://doc.rust-lang.org/std/collections/struct.VecDeque.html
- `BinaryHeap` is a max-heap. For a min-heap push `std::cmp::Reverse(x)`. https://doc.rust-lang.org/std/collections/struct.BinaryHeap.html
- Iterating a heap is in arbitrary order. Use `pop` in a loop or `into_sorted_vec()` (ascending). https://doc.rust-lang.org/std/collections/struct.BinaryHeap.html#method.into_sorted_vec

## HashMap and HashSet

- Default hasher is randomly seeded and currently SipHash 1-3 (subject to change). Other hashers via `with_hasher` typically do not protect against HashDoS. Keep the default for untrusted keys. https://doc.rust-lang.org/std/collections/struct.HashMap.html
- `HashMap::new()` is not const. For a static map use `LazyLock` (keeps random seed). A fixed `BuildHasherDefault` with `with_hasher` (const since 1.85.0) is not DoS resistant. https://doc.rust-lang.org/std/collections/struct.HashMap.html#usage-in-const-and-static

```rust
static MAP: LazyLock<Mutex<HashMap<String, u32>>> =
    LazyLock::new(|| Mutex::new(HashMap::new()));
```

- Mutating a key's hash or equality while it is in the map (via `Cell`, `RefCell`) is a logic error: unspecified behavior confined to the map, never UB. https://doc.rust-lang.org/std/collections/struct.HashMap.html
- Query `HashMap<String, V>` with `&str` directly. The borrowed form must hash and compare like the key. https://doc.rust-lang.org/std/collections/struct.HashMap.html#method.get
- `insert` on an existing key replaces the value but keeps the old key. https://doc.rust-lang.org/std/collections/struct.HashMap.html#method.insert
- `HashSet::insert` on an existing value keeps the original and drops the new one. https://doc.rust-lang.org/std/collections/struct.HashSet.html#method.insert
- `retain` and iteration cost O(capacity), not O(len). https://doc.rust-lang.org/std/collections/struct.HashMap.html#method.retain
- `extract_if(pred)` is stable on HashMap and HashSet since 1.88.0. `get_disjoint_mut([&k1, &k2])` (1.86.0) panics on duplicate keys. `try_insert` is nightly. https://doc.rust-lang.org/std/collections/struct.HashMap.html#method.extract_if
- Entry API: `or_insert(v)` takes a value, `or_insert_with(f)` a closure, `or_default()`, `and_modify(f).or_insert(v)`. https://doc.rust-lang.org/std/collections/hash_map/enum.Entry.html

```rust
*counts.entry(word).or_insert(0) += 1;
stats.entry("mana").and_modify(|m| *m += 200).or_insert(100);
```

https://doc.rust-lang.org/std/collections/struct.HashMap.html#entry-api

## BTreeMap

- `range(a..b)` panics if `start > end`, or if `start == end` with both bounds excluded. https://doc.rust-lang.org/std/collections/struct.BTreeMap.html#method.range
- `first_key_value`, `pop_first`, `last_key_value`, `pop_last` (1.66.0) give min/max. Drain in order with `while let Some((k, v)) = map.pop_first()`. https://doc.rust-lang.org/std/collections/struct.BTreeMap.html#method.pop_first
- `extract_if(range, pred)` is stable since 1.91.0 and takes a range: `map.extract_if(.., |k, _v| k % 2 == 0)`. https://doc.rust-lang.org/std/collections/struct.BTreeMap.html#method.extract_if

## Hash

- Derive `PartialEq, Eq, Hash` together. If written by hand, `a == b` must imply equal hashes. https://doc.rust-lang.org/std/hash/trait.Hash.html#hash-and-eq
- Hash output is not portable across platforms or compiler versions. Do not hard-code hash values in tests or serialized formats. https://doc.rust-lang.org/std/hash/trait.Hash.html#portability

## Old patterns

| Old | Current |
| :-- | :-- |
| `std::hash::SipHasher` (deprecated, SipHash 2-4) | `std::hash::DefaultHasher` (the default `Hasher` of `RandomState`) https://doc.rust-lang.org/std/hash/index.html |
| Manual retain-and-collect loop | `extract_if` (Vec 1.87, HashMap/HashSet 1.88, BTreeMap 1.91) https://doc.rust-lang.org/std/vec/struct.Vec.html#method.extract_if |
| `array.into_iter()` in 2015/2018 for values | `IntoIterator::into_iter(array)` or edition 2021+ https://doc.rust-lang.org/std/primitive.array.html#editions |
