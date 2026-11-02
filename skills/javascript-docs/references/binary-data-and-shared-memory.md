# Binary data and shared memory

## Contents

- Typed arrays and DataView
- Resizable and transferred buffers
- Base64 and hex
- SharedArrayBuffer and Atomics
- Old patterns

## Typed arrays and DataView

- Use `DataView` whenever byte order matters: typed arrays use the platform order. WebAssembly memory is always little-endian. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/TypedArray , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/DataView
- `DataView` getters and setters default to big-endian. Pass `true` for little-endian. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/DataView

```js
const buffer = new ArrayBuffer(2);
new DataView(buffer).setInt16(0, 256, true /* littleEndian */);
new Int16Array(buffer)[0] === 256; // true on little-endian platforms
```
https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/DataView

- `subarray()` is a view on the same buffer, `slice()` a copy. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/TypedArray/subarray , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/TypedArray/slice
- A non-empty typed array cannot be frozen. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/TypedArray
- `byteOffset` must be a multiple of the element size, or the constructor throws `RangeError`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/TypedArray

```js
Object.freeze(Int8Array.of(1, 2, 3)); // TypeError
new Int32Array(new ArrayBuffer(4), 1); // RangeError: offset must be multiple of 4
```
https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/TypedArray

- Out-of-range writes do not throw: integer arrays keep the low bits, `Uint8ClampedArray` clamps then rounds half to even. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/TypedArray
- Indexed access never reads the prototype: `new Int8Array(8)[-1]` is `undefined`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/TypedArray
- A BigInt typed array and a number typed array cannot copy into each other. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/TypedArray
- Typed array `sort()` is numeric by default. Copying versions `toSorted()`, `toReversed()`, `with()` exist. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/TypedArray/sort
- `TypedArray.from()` reads the whole iterator before sizing and is always dense. `set(source, offset)` throws `RangeError` past the end. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/TypedArray/from , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/TypedArray/set
- `Float16Array` suits float canvases, WebGPU, WebGL and ML, and may be slow without hardware support. `Math.f16round()` uses the same rounding. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Float16Array , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/TypedArray
- Avoid relying on `[Symbol.species]` on typed arrays and buffers: engines are investigating its removal. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/TypedArray/Symbol.species , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/ArrayBuffer/Symbol.species

## Resizable and transferred buffers

- Grow a buffer in place with `new ArrayBuffer(n, { maxByteLength })` and `resize()`. Keep `maxByteLength` as small as the use case allows, never above 1073741824. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/ArrayBuffer/ArrayBuffer
- A typed array created without a `length` tracks the buffer size. One with an explicit `length` stays fixed and goes out of bounds (length 0) if the buffer shrinks below it. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/TypedArray

```js
const buffer = new ArrayBuffer(8, { maxByteLength: 16 });
const f = new Float32Array(buffer); // length-tracking
buffer.resize(12);
f.length; // 3
const g = new Float32Array(buffer, 0, 2); // fixed, stays 2
```
https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/TypedArray

- Move ownership with `transfer()`, which keeps resizability. Use `transferToFixedLength()` for a fixed result. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/ArrayBuffer/transfer
- After a transfer the old buffer is detached: `byteLength` reads 0 and methods throw `TypeError`. Check `buffer.detached`. `WebAssembly.Memory.buffer` cannot be transferred. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/ArrayBuffer , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/ArrayBuffer/detached , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/ArrayBuffer/transfer

## Base64 and hex

- Use `Uint8Array.fromBase64()` and `toBase64()` instead of `atob()` and `btoa()`, unless the data is ASCII text. Pass `{ alphabet: "base64url" }` for URL-safe input. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Uint8Array/fromBase64 , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Uint8Array/toBase64
- The default `lastChunkHandling: "loose"` accepts missing padding. Use `"strict"` for exact padding, and `"stop-before-partial"` with `setFromBase64()` for streams. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Uint8Array/fromBase64
- `setFromBase64()` fills an existing array and returns `{ read, written }`. It never splits a 4-character chunk, so the last bytes may stay unwritten. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Uint8Array/setFromBase64
- `Uint8Array.fromHex()` rejects odd lengths and whitespace. For one number use `parseInt(s, 16)`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Uint8Array/fromHex

```js
Uint8Array.fromHex("cafed00d"); // Uint8Array [ 202, 254, 208, 13 ]
```
https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Uint8Array/fromHex

## SharedArrayBuffer and Atomics

- `SharedArrayBuffer` needs a secure, cross-origin isolated page. Check `crossOriginIsolated` and fall back to `ArrayBuffer`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/SharedArrayBuffer

```js
if (crossOriginIsolated) {
  myWorker.postMessage(new SharedArrayBuffer(16));
} else {
  myWorker.postMessage(new ArrayBuffer(16));
}
```
https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/SharedArrayBuffer

- A `SharedArrayBuffer` cannot be transferred or shrunk, only grown. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/SharedArrayBuffer
- Update shared memory with `Atomics.add()` and friends: `i[0] += 1` is a data race. Use `Atomics.exchange()` when you need the old value. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Agent_communication_and_coordination#atomic_updates , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Atomics/store
- `Atomics.wait()` blocks, so it throws a `TypeError` where the thread cannot be blocked, such as the web main thread. Use `Atomics.waitAsync()` there. Both work on `Int32Array` or `BigInt64Array` over shared memory only. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Atomics/wait , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Atomics/waitAsync
- `wait()` returns `"ok"` on any notify, changed value or not: re-check the value. Set the ready flag right before `notify()`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Atomics/wait , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Agent_communication_and_coordination#waiting_and_notifying
- Release locks in `finally` and acquire several locks in a fixed order. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Agent_communication_and_coordination#avoiding_deadlocks
- Spin briefly with `compareExchange` plus `Atomics.pause()`, then fall back to `wait()`. Avoid spinning on the main thread: it freezes the page. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Atomics/pause

## Old patterns

| Old pattern | Replacement | Source |
| :-- | :-- | :-- |
| `atob()`, `btoa()` | `Uint8Array.fromBase64()`, `toBase64()` | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Uint8Array/fromBase64 |
| Copying into a bigger buffer by hand | resizable buffer or `transfer(newByteLength)` | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/ArrayBuffer , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/ArrayBuffer/transfer |
| Busy-wait loops | `Atomics.pause()`, then `wait()` or `waitAsync()` | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Agent_communication_and_coordination#waiting_and_notifying |
