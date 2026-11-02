# Web Streams and node:stream/iter

## Contents

- Web Streams
- Interop with Node streams
- node:stream/iter (experimental, added in v25.9.0)
- Old patterns

## Web Streams

- `ReadableStream`, `WritableStream`, `TransformStream` and friends are globals and live in `node:stream/web` (stable). https://nodejs.org/docs/latest/api/webstreams.html
- `for await` over a `ReadableStream` locks it, and leaving early cancels it. Keep the stream with `values({ preventCancel: true })`. https://nodejs.org/docs/latest/api/webstreams.html#async-iteration

```js
for await (const chunk of stream.values({ preventCancel: true })) console.log(Buffer.from(chunk).toString());
```

- Never pass a pooled Buffer (from `Buffer.allocUnsafe()`, `Buffer.from()`, many fs callbacks) to `ReadableStreamBYOBReader.read(view)`: its ArrayBuffer is detached, invalidating every view on the shared pool. Allocate a dedicated `Uint8Array` or `Buffer.alloc()`. https://nodejs.org/docs/latest/api/webstreams.html#readablestreambyobreaderreadview-options
- BYOB needs `new ReadableStream({ type: 'bytes' })`. `pull` is not called again until its previous promise settles. https://nodejs.org/docs/latest/api/webstreams.html#new-readablestreamunderlyingsource--strategy
- Build from any iterable with `ReadableStream.from()`. https://nodejs.org/docs/latest/api/webstreams.html#readablestreamfromiterable
- Transfer a stream between threads with `port.postMessage(stream, [stream])`. https://nodejs.org/docs/latest/api/webstreams.html#transferring-with-postmessage
- `CompressionStream` formats: `'deflate'`, `'deflate-raw'`, `'gzip'`, and `'brotli'` (added v24.7.0). https://nodejs.org/docs/latest/api/webstreams.html#new-compressionstreamformat
- Decode text with `TextDecoderStream`. It strips the BOM by default. https://nodejs.org/docs/latest/api/webstreams.html#class-textdecoderstream
- `ReadableStreamTee(stream, cloneForBranch2)` (added in v26.5.0, experimental) clones chunks into the second branch so one branch's mutations are not seen by the other. `stream.tee()` never clones. https://nodejs.org/docs/latest/api/webstreams.html#readablestreamteestream-cloneforbranch2

## Interop with Node streams

- Convert with `Readable.toWeb/fromWeb`, `Writable.toWeb/fromWeb`, `Duplex.toWeb/fromWeb`, stable since v24.0.0. `pipeline()` and `compose()` accept Web Streams directly. https://nodejs.org/docs/latest/api/webstreams.html#nodejs-streams-interoperability
- For a byte stream from `Duplex.toWeb()` pass `readableType: 'bytes'` (the `type` option is deprecated). https://nodejs.org/docs/latest/api/stream.html#streamduplextowebstreamduplex-options

## node:stream/iter (experimental, added in v25.9.0)

Not on the v24 LTS line. Needs `--experimental-stream-iter`. https://nodejs.org/docs/latest/api/stream_iter.html

- Streams are plain iterables of batches (`Uint8Array[]` per step) to amortize promise cost. All data is bytes, strings are UTF-8 encoded. A transform is a function called per batch (receiving `null` as the flush signal) or a `{ transform(source, options) }` generator for stateful work. https://nodejs.org/docs/latest/api/stream_iter.html#concepts
- Core functions: `from()`, `pull(source, ...transforms)` (lazy, natural backpressure), `pipeTo(source, ...transforms, writer)`, `push()`, `duplex()`, and consumers `text`, `bytes`, `array`, `arrayBuffer`. Compression transforms live in `node:zlib/iter`. https://nodejs.org/docs/latest/api/stream_iter.html#pipelines

```js
import { from, pull, text } from 'node:stream/iter';
import { compressGzip, decompressGzip } from 'node:zlib/iter';
const compressed = pull(from('Hello, world!'), compressGzip());
console.log(await text(pull(compressed, decompressGzip())));
```
https://nodejs.org/docs/latest/api/stream_iter.html#pipelines

- With `push()`, always `await writer.write()` and run the consumer concurrently, otherwise the first write that fills the buffer waits forever. Fast path: `if (!writer.writeSync(chunk)) await writer.write(chunk);`. https://nodejs.org/docs/latest/api/stream_iter.html#writer-interface

```js
const { writer, readable } = push({ budget: 16384 });
const consuming = text(readable);
for (const item of dataset) await writer.write(item);
await writer.end();
```
https://nodejs.org/docs/latest/api/stream_iter.html#backpressure

- Backpressure defaults to `'strict'`: unawaited writes throw "Backpressure violation". `'unbounded'` (the classic and Web Streams behavior) leaks memory silently on unawaited writes. `'drop-oldest'` / `'drop-newest'` suit live feeds. https://nodejs.org/docs/latest/api/stream_iter.html#backpressure
- `broadcast()`: create consumers with `bc.push()` before writing. `share(source)`: when all consumers detach the source stays open, so call `share.cancel()`. Only `'drop-oldest'` lets fast consumers proceed past a stalled one. https://nodejs.org/docs/latest/api/stream_iter.html#sharesource-options , https://nodejs.org/docs/latest/api/stream_iter.html#broadcastoptions
- `dump(source, { limit })` (added in v26.12.0) drains unneeded data with memory capped to one batch. https://nodejs.org/docs/latest/api/stream_iter.html#dumpsource-options
- `fromReadable()`, `toReadable()`, `fromWritable()`, `toWritable()` (added in v26.1.0) bridge classic streams. They are Node extensions, not part of the draft standard. https://nodejs.org/docs/latest/api/stream_iter.html#classic-stream-interop

## Old patterns

| Old pattern | Replacement | Source |
| :-- | :-- | :-- |
| `Duplex.toWeb(d, { type: 'bytes' })` | `Duplex.toWeb(d, { readableType: 'bytes' })` | https://nodejs.org/docs/latest/api/deprecations.html#DEP0201 |
| `stream.tee()` when branches mutate chunks | `ReadableStreamTee(stream, true)` (v26.5.0+) | https://nodejs.org/docs/latest/api/webstreams.html#readablestreamteestream-cloneforbranch2 |
