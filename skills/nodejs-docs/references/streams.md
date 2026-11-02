# node:stream

## Contents

- Composing with pipeline
- Backpressure and highWaterMark
- Consuming a Readable
- Ending and destroying a Writable
- Implementing streams
- Experimental and version-flagged
- Old patterns

## Composing with pipeline

- Default to `pipeline()` from `node:stream/promises` over `.pipe()`. With `.pipe()`, an erroring Readable does not close the destination, so every stream must be closed by hand or it leaks. https://nodejs.org/docs/latest/api/stream.html#readablepipedestination-options
- `pipeline()` takes async generators as source, transform or destination, and Web Streams. A generator transform receives `{ signal }`. https://nodejs.org/docs/latest/api/stream.html#streampipelinesource-transforms-destination-options

```js
await pipeline(
  fs.createReadStream('lowercase.txt'),
  async function* (source, { signal }) {
    source.setEncoding('utf8');
    for await (const chunk of source) yield await processChunk(chunk, { signal });
  },
  fs.createWriteStream('uppercase.txt'),
);
```
https://nodejs.org/docs/latest/api/stream.html#streampipelinesource-transforms-destination-options

- An async generator used as the pipeline source must honor `signal`, otherwise the pipeline never completes on abort. https://nodejs.org/docs/latest/api/stream.html#streampipelinesource-transforms-destination-options
- `end: false` keeps the destination open (Transform streams are always ended). The callback form leaves listeners attached after it runs. https://nodejs.org/docs/latest/api/stream.html#streampipelinesource-transforms-destination-options , https://nodejs.org/docs/latest/api/stream.html#streampipelinestreams-callback
- On error `pipeline()` destroys the streams, including an HTTP response's socket, so a fallback `res.end('error')` in its callback is never delivered. Validate (file exists, status) before piping. https://nodejs.org/docs/latest/api/stream.html#streampipelinestreams-callback
- Detect premature destruction (an aborted request) with `finished()`. It leaves listeners on purpose; call the returned cleanup if unwanted. https://nodejs.org/docs/latest/api/stream.html#streamfinishedstream-options-callback

```js
const cleanup = finished(rs, (err) => { cleanup(); /* ... */ });
```

- `stream.compose(...streams)` returns one Duplex (write to the first, read from the last) that can be piped further. Marked stable in v26.2.0. https://nodejs.org/docs/latest/api/stream.html#streamcomposestreams
- `stream.duplexPair()` gives 2 connected Duplex ends, like a network connection. https://nodejs.org/docs/latest/api/stream.html#streamduplexpairoptions

## Backpressure and highWaterMark

- When `write()` returns `false`, stop until `'drain'`. Ignoring it buffers until memory runs out, and on a TCP socket whose peer stops reading it can be "a remotely exploitable vulnerability". https://nodejs.org/docs/latest/api/stream.html#writablewritechunk-encoding-callback

```js
function write(data, cb) {
  if (!stream.write(data)) stream.once('drain', cb);
  else process.nextTick(cb);
}
```

- Data producible on demand belongs in a `Readable`, piped. https://nodejs.org/docs/latest/api/stream.html#writablewritechunk-encoding-callback
- `highWaterMark` is a threshold, not a memory limit. Defaults since v22.0.0: 64 KiB for byte streams (16 KiB on Windows), 16 objects in object mode. Read or change them with `getDefaultHighWaterMark()` / `setDefaultHighWaterMark()`. https://nodejs.org/docs/latest/api/stream.html#buffering , https://nodejs.org/docs/latest/api/stream.html#streamgetdefaulthighwatermarkobjectmode
- After `setEncoding()` the read-side `highWaterMark` counts characters, not bytes. https://nodejs.org/docs/latest/api/stream.html#highwatermark-discrepancy-after-calling-readablesetencoding
- Set object mode in the constructor only, switching later is not safe. `null` is reserved for end of stream. https://nodejs.org/docs/latest/api/stream.html#object-mode

## Consuming a Readable

- Choose one style per stream: `for await`, `'data'`, `'readable'` or `pipe()`. Mixing them is unpredictable. https://nodejs.org/docs/latest/api/stream.html#choose-one-api-style
- Data is lost when a stream flows with no consumer (`resume()` without a `'data'` listener). Removing `'data'` listeners does not pause. https://nodejs.org/docs/latest/api/stream.html#two-reading-modes
- Once `readableFlowing` is `false` (after `pause()` or `unpipe()`), adding a `'data'` listener does not restart flow: call `resume()`. https://nodejs.org/docs/latest/api/stream.html#three-states
- A socket with only an `'end'` listener never ends because it stays paused. Call `socket.resume()` to discard the data. https://nodejs.org/docs/latest/api/stream.html#compatibility-with-older-nodejs-versions
- With `'readable'`, call `read()` in a loop until `null`. It overrides `'data'`. https://nodejs.org/docs/latest/api/stream.html#event-readable
- `break`, `return` or `throw` out of `for await` destroys the stream. Keep it with `readable.iterator({ destroyOnReturn: false })`. https://nodejs.org/docs/latest/api/stream.html#readableiteratoroptions
- Decode text with `readable.setEncoding('utf8')` or a `StringDecoder`, never `chunk.toString()` per chunk, which splits multi-byte characters. https://nodejs.org/docs/latest/api/stream.html#readablesetencodingencoding , https://nodejs.org/docs/latest/api/string_decoder.html
- `StringDecoder.end()` turns a trailing incomplete character into substitution characters. https://nodejs.org/docs/latest/api/string_decoder.html#stringdecoderendbuffer
- `Readable.from(iterable)` defaults to object mode. A string or Buffer is emitted as one chunk, not iterated. An iterable of promises can cause unhandled rejections. https://nodejs.org/docs/latest/api/stream.html#streamreadablefromiterable-options
- `node:stream/consumers` (`text`, `json`, `buffer`...) each function returns a promise of the whole stream content, so use them only for bounded input. https://nodejs.org/docs/latest/api/webstreams.html#utility-consumers

## Ending and destroying a Writable

- `end()` flushes then closes. `destroy()` is immediate, and pending writes may fail with `ERR_STREAM_DESTROYED`. https://nodejs.org/docs/latest/api/stream.html#writabledestroyerror
- Duplex `allowHalfOpen` defaults to `true`. Set it `false` to end the writable side with the readable side. https://nodejs.org/docs/latest/api/stream.html#duplexallowhalfopen
- Dispose with `await using`: `[Symbol.asyncDispose]()` destroys with an `AbortError` and waits. Stable on Readable since v24.2.0. https://nodejs.org/docs/latest/api/stream.html#readablesymbolasyncdispose

## Implementing streams

- Report errors from `_write`, `_writev` and `_final` through the callback, and from `_read` with `this.destroy(err)`. Throwing or emitting `'error'` yourself is undefined behavior. https://nodejs.org/docs/latest/api/stream.html#errors-while-writing , https://nodejs.org/docs/latest/api/stream.html#errors-while-reading

```js
new Writable({ write(chunk, enc, cb) { cb(bad(chunk) ? new Error('invalid') : null); } });
new Readable({ read() { const err = check(); if (err) this.destroy(err); } });
```

- Stream methods are callback-based: avoid `async` implementation methods. Thenable support was removed. https://nodejs.org/docs/latest/api/deprecations.html#DEP0157
- Avoid overriding `write`, `end`, `cork`, `uncork`, `read` or `destroy`, and emitting `'error'`/`'data'`/`'end'`/`'finish'`/`'close'` through `.emit()`. Implement the underscore methods or pass `read`/`write`/`transform`/`final`/`construct` options. https://nodejs.org/docs/latest/api/stream.html#api-for-stream-implementers
- Do not let callers override `autoDestroy` or `emitClose` if your code assumes them: forward options explicitly. https://nodejs.org/docs/latest/api/stream.html#api-for-stream-implementers
- The `_write` callback signals a chunk is done, sync or later. Implement `_writev` for several chunks. https://nodejs.org/docs/latest/api/stream.html#writable_writechunk-encoding-callback
- In `_read`, stop pushing when `push()` returns `false` and resume on the next `_read`. Push `null` for EOF. https://nodejs.org/docs/latest/api/stream.html#readable_readsize
- In `_transform`, `callback(null, data)` equals `push(data)` plus `callback()`. Emit trailing output in `_flush`. https://nodejs.org/docs/latest/api/stream.html#transform_transformchunk-encoding-callback , https://nodejs.org/docs/latest/api/stream.html#transform_flushcallback
- Avoid `unshift()` inside a `_read()` implementation. If needed, follow it with `push('')`. https://nodejs.org/docs/latest/api/stream.html#readableunshiftchunk-encoding

## Experimental and version-flagged

- Readable operators (`map`, `filter`, `flatMap`, `take`, `reduce`, `toArray`...) are still experimental in v26. `map({ concurrency })` defaults to 1. https://nodejs.org/docs/latest/api/stream.html#readablemapfn-options
- `readable[Symbol.for('Stream.toAsyncStreamable')]()` (added in v26.1.0, experimental) needs `--experimental-stream-iter`. https://nodejs.org/docs/latest/api/stream.html#readablesymbolforstreamtoasyncstreamable

## Old patterns

| Old pattern | Replacement | Source |
| :-- | :-- | :-- |
| `a.pipe(b).pipe(c)` | `await pipeline(a, b, c)` | https://nodejs.org/docs/latest/api/stream.html#streampipelinestreams-options |
| `addAbortSignal(signal, stream)` to dispose a stream | `await using stream = ...` | https://nodejs.org/docs/latest/api/deprecations.html#DEP0209 |
| `Duplex.toWeb(d, { type })` | `{ readableType: 'bytes' }` | https://nodejs.org/docs/latest/api/deprecations.html#DEP0201 |
| `require('node:_stream_*')`, `node:_stream_wrap` | `node:stream` | https://nodejs.org/docs/latest/api/deprecations.html#DEP0193 |
| `async _final()` returning a promise | call the callback | https://nodejs.org/docs/latest/api/deprecations.html#DEP0157 |
| `readable.wrap(oldStream)` | pre-0.10 streams only | https://nodejs.org/docs/latest/api/stream.html#readablewrapstream |
