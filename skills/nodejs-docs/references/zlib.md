# node:zlib

## Contents

- Usage and threadpool
- Memory tuning
- Decompressing untrusted data
- Flush and reset
- Zstd (experimental)
- ZIP archives (added in v26.8.0)
- Old patterns

## Usage and threadpool

- Compress files and streams with `pipeline()`, which handles errors and cleanup. https://nodejs.org/docs/latest/api/zlib.html

```js
pipeline(createReadStream('input.txt'), createGzip(), createWriteStream('input.txt.gz'), (err) => {
  if (err) { console.error('An error occurred:', err); process.exitCode = 1; }
});
```

- Supported: Gzip, Deflate/Inflate (zlib and raw), Brotli, Zstd. HTTP encodings `gzip`, `deflate`, `br`, `zstd`. https://nodejs.org/docs/latest/api/zlib.html#compressing-http-requests-and-responses
- The `*Sync` convenience methods block the event loop: avoid them in servers. https://nodejs.org/docs/latest/api/zlib.html#convenience-methods
- Every async zlib call uses the libuv threadpool. Creating many zlib objects at once fragments memory. Cache compressed results: the docs mark 30,000 concurrent `zlib.deflate()` calls in a loop as "DO NOT DO THIS". https://nodejs.org/docs/latest/api/zlib.html#threadpool-usage-and-performance-considerations
- Compression is expensive: cache compressed HTTP responses. https://nodejs.org/docs/latest/api/zlib.html#compressing-http-requests-and-responses
- Once a response pipeline fails, headers are already sent: only `response.end()` and logging remain. https://nodejs.org/docs/latest/api/zlib.html#flushing
- `zlib.crc32(data, value)` (added v22.2.0) is a non-cryptographic checksum. Strings are hashed as UTF-8, unlike browser libraries that hash `charCodeAt()` values. https://nodejs.org/docs/latest/api/zlib.html#zlibcrc32data-value

## Memory tuning

- Deflate needs `(1 << (windowBits + 2)) + (1 << (memLevel + 9))` bytes, 256 KiB by default. `{ windowBits: 14, memLevel: 7 }` halves it with worse compression. Inflate needs `1 << windowBits` (32 KiB), plus one `chunkSize` slab (16 KiB). `level` dominates speed. https://nodejs.org/docs/latest/api/zlib.html#for-zlib-based-streams
- Equivalents: zlib `level` is Brotli `BROTLI_PARAM_QUALITY` and Zstd `ZSTD_c_compressionLevel`. `windowBits` is `BROTLI_PARAM_LGWIN` and `ZSTD_c_windowLog`. https://nodejs.org/docs/latest/api/zlib.html#for-zlib-based-streams

```js
zlib.createBrotliCompress({
  params: {
    [zlib.constants.BROTLI_PARAM_MODE]: zlib.constants.BROTLI_MODE_TEXT,
    [zlib.constants.BROTLI_PARAM_QUALITY]: 4,
    [zlib.constants.BROTLI_PARAM_SIZE_HINT]: fs.statSync(inputFile).size,
  },
});
```
https://nodejs.org/docs/latest/api/zlib.html#class-brotlioptions

## Decompressing untrusted data

- Set `maxOutputLength` explicitly on convenience methods: the default is `buffer.kMaxLength`, so output is effectively unbounded by default. https://nodejs.org/docs/latest/api/zlib.html#class-options
- `rejectGarbageAfterEnd: true` (added in v26.5.0, default `false`) fails on trailing input after the end of the stream, including extra gzip members. https://nodejs.org/docs/latest/api/zlib.html#class-options
- Truncated input throws by default. To inspect a partial file, decompress with `finishFlush: zlib.constants.Z_SYNC_FLUSH` (Brotli `BROTLI_OPERATION_FLUSH`, Zstd `ZSTD_e_flush`), then validate the result yourself: truncation and a missing integrity check look the same. https://nodejs.org/docs/latest/api/zlib.html#compressing-http-requests-and-responses

```js
zlib.unzip(buffer, { finishFlush: zlib.constants.Z_SYNC_FLUSH }, (err, buffer) => { /* ... */ });
```

## Flush and reset

- Call `.flush()` only for streaming partial responses (for example in the `write` callback of a server-sent stream). Premature flushes hurt compression, and the flush is queued like a write. https://nodejs.org/docs/latest/api/zlib.html#zlibflushkind-callback
- `params(level, strategy, callback)` works only on deflate-based streams. https://nodejs.org/docs/latest/api/zlib.html#zlibparamslevel-strategy-callback
- Resetting a gzip or Zstd stream after it emitted output for an incomplete member throws `ERR_ZLIB_INCOMPLETE_FRAME`: `end()` it or create a new stream. `reset()` during a write throws. Brotli keeps params and dictionary across `reset()` since v26.11.0. https://nodejs.org/docs/latest/api/zlib.html#zlibreset

## Zstd (experimental)

- `createZstdCompress()` / `createZstdDecompress()` are experimental (added v23.8.0, backported to v22.15.0). https://nodejs.org/docs/latest/api/zlib.html#class-zstdoptions
- `pledgedSrcSize` must equal the real input size at finish, else compression fails with `ZSTD_error_srcSize_wrong`. https://nodejs.org/docs/latest/api/zlib.html#pledged-source-size
- `ZstdDecompress` handles concatenated and skippable frames. https://nodejs.org/docs/latest/api/zlib.html#class-zlibzstddecompress

```js
zlib.createZstdCompress({
  params: { [zlib.constants.ZSTD_c_compressionLevel]: 10, [zlib.constants.ZSTD_c_checksumFlag]: 1 },
});
```

## ZIP archives (added in v26.8.0)

Early development (Stability 1.0). The first use emits an experimental warning. https://nodejs.org/docs/latest/api/zlib.html#class-zlibzipbuffer

- `ZipBuffer` is a zero-copy view: do not modify or reuse the source memory while it or any entry is alive. Pass `Buffer.from(source)` if it may change. Edits are serialized by `toBuffer()`. https://nodejs.org/docs/latest/api/zlib.html#class-zlibzipbuffer

```js
const zip = new ZipBuffer(readFileSync('archive.zip'));
for (const [name, entry] of zip) console.log(name, entry.size);
await zip.add('hello.txt', Buffer.from('Hello, world!'));
writeFileSync('archive.zip', await zip.toBuffer());
```

- `entry.content({ maxSize })` verifies CRC-32 and rejects entries declared above `maxSize` (default 256 MiB, `setMaxZipContentSize()`) before allocating. Streaming reads (`contentIterator()`, `zipFile.stream()`) are not capped by that default (no limit unless you pass `maxSize`). https://nodejs.org/docs/latest/api/zlib.html#zlibsetmaxzipcontentsizesize

## Old patterns

| Old pattern | Replacement | Source |
| :-- | :-- | :-- |
| `zlib.Gzip()` without `new` | `new zlib.Gzip()` or `zlib.createGzip()` | https://nodejs.org/docs/latest/api/deprecations.html#DEP0184 |
| `zlib.gunzipSync(untrusted)` with defaults | `{ maxOutputLength }` and the async form | https://nodejs.org/docs/latest/api/zlib.html#class-options |
