# node:buffer

## Contents

- Allocation
- The shared pool
- Encodings
- Buffer versus TypedArray
- Blob and File
- Version-flagged additions
- Old patterns

## Allocation

- Import it explicitly even though it is global: `import { Buffer } from 'node:buffer'`. https://nodejs.org/docs/latest/api/buffer.html
- Default to `Buffer.alloc(size)` (zero-filled, never pooled). Use `Buffer.allocUnsafe()` only when performance matters, and overwrite it completely before reading or sending: it may contain old sensitive data. https://nodejs.org/docs/latest/api/buffer.html#what-makes-bufferallocunsafe-and-bufferallocunsafeslow-unsafe
- Never use `new Buffer()` / `Buffer()`. Behavior depends on the argument type, so a number injected where a string was expected allocates that many bytes (DoS). https://nodejs.org/docs/latest/api/buffer.html#bufferfrom-bufferalloc-and-bufferallocunsafe
- `Buffer.from([257, 257.5, -255, '1'])` is `[1, 1, 1, 1]`: values are truncated with `& 255`. https://nodejs.org/docs/latest/api/buffer.html
- `--zero-fill-buffers` zero-fills unsafe allocations at a measurable cost. https://nodejs.org/docs/latest/api/buffer.html#the---zero-fill-buffers-command-line-option

## The shared pool

- `allocUnsafe`, `from(array)`, `from(string)` and `concat()` may use a shared pool when size is below `Buffer.poolSize >>> 1`. So `buf.byteOffset` is often not 0 and `buf.buffer` is larger than the Buffer: always pass `byteOffset` and length when reaching the ArrayBuffer. https://nodejs.org/docs/latest/api/buffer.html#bufferpoolsize
- `Buffer.poolSize` defaults to 65536 since v26.3.0 (8192 on v24). https://nodejs.org/docs/latest/api/buffer.html#bufferpoolsize
- `Buffer.alloc(size, fill)` never uses the pool, `Buffer.allocUnsafe(size).fill(fill)` can. https://nodejs.org/docs/latest/api/buffer.html#static-method-bufferallocsize-fill-encoding
- Pooled Buffers cannot be transferred to workers (the whole pool is cloned) and must never be passed to a BYOB reader `read()`. Use `Buffer.alloc()` or `allocUnsafeSlow()` there. https://nodejs.org/docs/latest/api/worker_threads.html#considerations-when-transferring-typedarrays-and-buffers , https://nodejs.org/docs/latest/api/webstreams.html#readablestreambyobreaderreadview-options

## Encodings

- Default is UTF-8. Invalid UTF-8 decodes to U+FFFD, losing data. https://nodejs.org/docs/latest/api/buffer.html#buffers-and-character-encodings
- `latin1` truncates characters above U+00FF, and WHATWG treats latin1 as windows-1252, so decoding an HTTP `charset=latin1` body with Node `latin1` can be wrong. `binary` is an alias of `latin1` and `ascii` clears the high bit: prefer `utf8`. https://nodejs.org/docs/latest/api/buffer.html#buffers-and-character-encodings
- `hex` decoding stops silently at the first non-hex character or a trailing odd digit. https://nodejs.org/docs/latest/api/buffer.html#buffers-and-character-encodings

```js
Buffer.from('1ag123', 'hex'); // <Buffer 1a>
Buffer.from('1a7', 'hex');    // <Buffer 1a>
```

- `base64` decoding accepts the URL-safe alphabet and ignores whitespace. `base64url` encodes without padding. https://nodejs.org/docs/latest/api/buffer.html#buffers-and-character-encodings
- Validate bytes with `buffer.isUtf8()` / `buffer.isAscii()`. https://nodejs.org/docs/latest/api/buffer.html#bufferisutf8input
- `buffer.transcode()` supports only ascii, utf8, utf16le, latin1. Unrepresentable characters become `?`. https://nodejs.org/docs/latest/api/buffer.html#buffertranscodesource-fromenc-toenc
- `buffer.constants.MAX_LENGTH` is 2^53 - 1 on 64-bit, but decoding beyond `MAX_STRING_LENGTH` fails. https://nodejs.org/docs/latest/api/buffer.html#bufferconstantsmax_length

## Buffer versus TypedArray

- `buf.subarray()` and the deprecated `buf.slice()` return views over the same memory, unlike `TypedArray.prototype.slice()` which copies. Use `subarray()` for a view and `Buffer.from(buf)` for a copy. https://nodejs.org/docs/latest/api/buffer.html#bufslicestart-end
- `new Uint32Array(buf)` copies one element per byte. To reinterpret bytes, pass the ArrayBuffer with offset and length: https://nodejs.org/docs/latest/api/buffer.html#buffers-and-typedarrays

```js
const u16 = new Uint16Array(buf.buffer, buf.byteOffset, buf.length / Uint16Array.BYTES_PER_ELEMENT);
```

- `Buffer.from(arrayBuffer, offset, length)` shares memory. `Buffer.from(array | buffer | string)` copies. https://nodejs.org/docs/latest/api/buffer.html#buffers-and-typedarrays
- Buffer methods work on a plain `Uint8Array` through `call`: `Buffer.prototype.toString.call(u8, 'utf8')`. https://nodejs.org/docs/latest/api/buffer.html#buffer-methods-are-callable-with-uint8array-instances

## Blob and File

- Blob sources (Buffer, ArrayBuffer, TypedArray) are copied, so mutating them afterwards is safe. Strings are UTF-8 encoded with lone surrogates replaced. `blob.type` is not validated. https://nodejs.org/docs/latest/api/buffer.html#new-bufferblobsources-options
- `blob.bytes()` returns a `Promise<Uint8Array>` (added v22.3.0). `blob.textStream()` is added in v26.5.0. https://nodejs.org/docs/latest/api/buffer.html#blobbytes , https://nodejs.org/docs/latest/api/buffer.html#blobtextstream

## Version-flagged additions

| API | Added | Source |
| :-- | :-- | :-- |
| `Buffer.allocUnsafe(size, alignment)` / `allocUnsafeSlow(size, alignment)` for O_DIRECT I/O | v26.8.0 | https://nodejs.org/docs/latest/api/buffer.html#aligned-allocations |
| `Buffer.stringLength(input, encoding)`: decoded length without decoding | v26.11.0 | https://nodejs.org/docs/latest/api/buffer.html#static-method-bufferstringlengthinput-encoding |
| `buffer.isLatin1(string)` (Node latin1, not windows-1252) | v26.11.0 | https://nodejs.org/docs/latest/api/buffer.html#bufferislatin1input |
| `end` parameter on `indexOf`, `lastIndexOf`, `includes` | v26.1.0 | https://nodejs.org/docs/latest/api/buffer.html |

- Alignment is not inherited by `subarray()` or `structuredClone()` and is lost in a startup snapshot. Measure before aligning for cache lines. https://nodejs.org/docs/latest/api/buffer.html#aligned-allocations
- Check `Buffer.stringLength(buf) <= constants.MAX_STRING_LENGTH` before decoding a large buffer. https://nodejs.org/docs/latest/api/buffer.html#static-method-bufferstringlengthinput-encoding

## Old patterns

| Old pattern | Replacement | Source |
| :-- | :-- | :-- |
| `new Buffer(size)` | `Buffer.alloc(size)` | https://nodejs.org/docs/latest/api/buffer.html#new-buffersize |
| `new Buffer(string, enc)`, `new Buffer(array)` | `Buffer.from(string, enc)`, `Buffer.from(array)` | https://nodejs.org/docs/latest/api/deprecations.html#DEP0005 |
| `buf.slice()` | `buf.subarray()` | https://nodejs.org/docs/latest/api/deprecations.html#DEP0158 |
| `buf.parent` | `buf.buffer` | https://nodejs.org/docs/latest/api/buffer.html#bufparent |
| `atob()` / `btoa()` | `Buffer.from(data, 'base64')` / `buf.toString('base64')` | https://nodejs.org/docs/latest/api/buffer.html#bufferatobdata |
