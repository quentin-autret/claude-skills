# Arrays and keyed collections

## Contents

- Copying and indexing
- Searching and iterating
- sort and reduce
- Sparse arrays and construction
- Map and Set
- WeakMap and WeakSet
- Old patterns

## Copying and indexing

- Prefer copying methods: `toReversed`, `toSorted`, `toSpliced`, `with`. They never return sparse arrays. To keep a mutating method, copy first with spread or `slice()`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array

```js
const values = [1, 10, 21, 2];
const sortedValues = values.toSorted((a, b) => a - b); // [1, 2, 10, 21]
console.log(values); // [1, 10, 21, 2]
```
https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/toSorted

- Read from the end with `at(-1)`: `arr[-1]` reads the property `"-1"`. Write with `with(-1, v)`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/at
- Every copy is shallow. `fill(obj)` puts one shared reference in every slot. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/fill
- Use `Array.isArray()`: it works across realms and rejects fakes. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/isArray
- `push(...more)` fails beyond the engine's argument limit. Use `concat()` or a loop. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/push

## Searching and iterating

- `includes()` finds `NaN`, `indexOf()` never does. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/includes
- Search from the end with `findLast()` / `findLastIndex()`, not `find()` on a reversed copy. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/findLast
- `forEach` cannot stop and does not await async callbacks. For early exit use `for...of`, `some`, `every`, `find`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/forEach
- Calling `map()` without using the result is an anti-pattern. To drop items while mapping, return `[]` from `flatMap()`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/map
- `["1","2","3"].map(parseInt)` gives `[1, NaN, NaN]`: the index becomes the radix. Use `map(Number)`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/map
- Mutating an array inside an iterative method gives hard-to-follow results. Avoid it. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array
- Array iterators are single use. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/values
- Format a list for people with `Intl.ListFormat`, not `toLocaleString()`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/toLocaleString

## sort and reduce

- Default `sort()` compares strings: `[1, 30, 4, 21, 100000].sort()` is `[1, 100000, 21, 30, 4]`. Pass `(a, b) => a - b`, which breaks with NaN. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/sort
- A comparator must be consistent and anti-symmetric: returning only `1` and `0` cannot sort reliably. `sort()` mutates and returns the same array. `undefined` goes last. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/sort
- Before `reduce()`, look for a dedicated tool: `flat()`, `Object.groupBy()`, `flatMap()`, `new Set()`, `find`, `some`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/reduce
- Pass an initial value to `reduce()` (an empty array throws without one). Return the accumulator each time. If you mutate it, a plain loop is clearer. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/reduce

## Sparse arrays and construction

- `new Array(3)` makes 3 empty slots, `new Array("2")` makes `["2"]`, and a negative or fractional length throws `RangeError`. Build a sequence with `Array.from({ length: n }, (_, i) => i)`, never sparse. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/Array , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/from
- Empty slots are not `undefined`. `forEach`, `map`, `filter`, `reduce`, `some`, `every`, `indexOf` skip them. `find`, `includes`, `join`, `values`, `toReversed`, `toSorted`, `toSpliced` and `with` treat them as `undefined`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array
- `arr[3.4] = "x"` and `arr["02"]` create plain properties, not elements. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Indexed_collections , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array
- Array methods are generic: `Array.prototype.join.call({ 0: "a", 1: "b", length: 2 }, "+")` is `"a+b"`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array

## Map and Set

- Use a `Map` for keys known only at run time, user input, non-string keys, or when order and `size` matter. Use an object for a fixed shape with string keys. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Keyed_collections#object_and_map_compared , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Map#objects_vs._maps
- `map["key"] = v` sets a plain property that `has`, `get` and `size` ignore. Always call `set()`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Map#setting_object_properties
- Keys compare by SameValueZero: objects by identity. `Map.groupBy()` results are read with the original key objects. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Map#key_equality , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Map/groupBy
- Insertion order counts the first insertion only. A Set value deleted and re-added goes last. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Map#description , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Set#using_the_set_object
- `Map` has no native JSON serialization. Build it with a replacer and reviver. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Map#objects_vs._maps
- Use `getOrInsertComputed(key, callback)` for an expensive default: `getOrInsert(key, value)` evaluates its default every call. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Map/getOrInsertComputed
- Use `Set` for membership: `has` is on average faster than `includes` on an array of the same length. Use `union`, `intersection`, `difference`, `symmetricDifference`, `isSubsetOf`, `isSupersetOf`, `isDisjointFrom`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Set#performance , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Set#set_composition
- Set methods accept set-like arguments (`size`, `has()`, `keys()`), so a `Map` works. Arrays do not. `isSubsetOf` is not proper. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Set#set-like_objects , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Set/isSubsetOf

## WeakMap and WeakSet

- Attach metadata to objects you do not own with a `WeakMap`: entries do not keep the key alive. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WeakMap#associating_metadata

```js
function withCache(fn) {
  const cache = new WeakMap();
  return (param) => cache.getOrInsertComputed(param, fn);
}
```
https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WeakMap/getOrInsertComputed

- Keys are objects or non-registered symbols. Keys cannot be enumerated: use `Map` to list them. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WeakMap#description
- Track visited objects (cycle detection) with a `WeakSet`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WeakSet#detecting_circular_references

## Old patterns

| Old pattern | Replacement | Source |
| :-- | :-- | :-- |
| `arr.reverse()` / `sort()` / `splice()` on shared data | `toReversed()` / `toSorted()` / `toSpliced()` | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array |
| `x instanceof Array` | `Array.isArray(x)` | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/isArray |
| `reduce` to group | `Object.groupBy()` / `Map.groupBy()` | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/reduce |
| `arr.splice(arr.indexOf(v), 1)` | `set.delete(v)` | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Keyed_collections#array_and_set_compared |
| `new Set([...a].filter(x => b.has(x)))` | `a.intersection(b)` | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Set#set_composition |
| `element.clicked = false` on DOM nodes | a `WeakMap` keyed by the element | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WeakMap#associating_metadata |
| Early-browser `Array.prototype.group` / `groupToMap` | `Object.groupBy()`, `Map.groupBy()` | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Map/groupBy , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/groupBy |
| Relying on `Array[Symbol.species]` | avoid, removal under investigation | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/Symbol.species |
