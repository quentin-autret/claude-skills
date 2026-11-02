# node:fs

## Contents

- Choosing an API form
- Do not check, then use
- File handles and descriptors
- Reading and writing
- Directories, temp dirs, removal
- Glob and copy
- Watching
- Old patterns

Paths and `file:` URLs accepted by fs are covered in the path and URL reference.

## Choosing an API form

- Default to `node:fs/promises`. Switch to the callback API when maximal performance (time and memory) matters: the docs rank it above the promise API. https://nodejs.org/docs/latest/api/fs.html#callback-example
- Promise and callback operations run on the libuv threadpool and are not synchronized: concurrent modifications of the same file can corrupt it. Never call `filehandle.write()`, `writeFile()` or `writev()` again on the same file before the previous promise settles. https://nodejs.org/docs/latest/api/fs.html#promises-api , https://nodejs.org/docs/latest/api/fs.html#fspromiseswritefilefile-data-options
- Callback operations are not sequenced: to `stat` after a `rename`, call it inside the rename callback or `await` both. https://nodejs.org/docs/latest/api/fs.html#ordering-of-callback-and-promise-based-operations
- On Linux, positional writes are ignored when the file is opened in append mode: the kernel appends. https://nodejs.org/docs/latest/api/fs.html#filehandlewritebuffer-offset-length-position

## Do not check, then use

- Never call `access()`, `exists()` or `stat()` before `open`/`readFile`/`writeFile`: the state can change in between (TOCTOU). Open directly and handle the error code. Use `access()` only when the file is not used afterwards. https://nodejs.org/docs/latest/api/fs.html#fsaccesspath-mode-callback , https://nodejs.org/docs/latest/api/fs.html#fsstatpath-options-callback

```js
try {
  await writeFile('myfile', data, { flag: 'wx' }); // exclusive create
} catch (err) {
  if (err.code === 'EEXIST') console.error('myfile already exists');
  else throw err;
}
```
https://nodejs.org/docs/latest/api/fs.html#fsaccesspath-mode-callback

- For reads, open with `'r'` and test `err.code === 'ENOENT'`. https://nodejs.org/docs/latest/api/fs.html#fsaccesspath-mode-callback
- `'x'` (O_EXCL) may not work on network file systems, and on POSIX it fails on a symlink even when the target is missing. https://nodejs.org/docs/latest/api/fs.html#file-system-flags
- On Windows, directory ACLs may limit access that `access()` does not report. https://nodejs.org/docs/latest/api/fs.html#fsaccesspath-mode-callback

## File handles and descriptors

- Close every `FileHandle` explicitly. Auto-close on GC is unreliable, and a `FileHandle` left to GC throws. Prefer `await using`, else `try/finally`. https://nodejs.org/docs/latest/api/fs.html#class-filehandle , https://nodejs.org/docs/latest/api/deprecations.html#DEP0137

```js
let file;
try {
  file = await open('/open/some/file.txt', 'r');
  const stat = await file.stat();
} finally {
  await file?.close();
}
```
https://nodejs.org/docs/latest/api/fs.html#class-filehandle

- Close `fs.Dir` explicitly too (`dir.close()`, `await using`, or a full `for await` loop). Relying on GC may throw in future versions. https://nodejs.org/docs/latest/api/deprecations.html#DEP0200
- Leaked descriptors eventually crash the process. Never close an fd while an operation on it is pending: behavior is undefined. https://nodejs.org/docs/latest/api/fs.html#file-descriptors , https://nodejs.org/docs/latest/api/fs.html#fsclosefd-callback

## Reading and writing

- `readFile` buffers the whole file and reads it chunk by chunk. For large or performance-sensitive files use `createReadStream`. `writeFile` likewise performs several writes: use `createWriteStream` for hot paths. https://nodejs.org/docs/latest/api/fs.html#fsreadfilepath-options-callback
- Aborting `writeFile` with `signal` is best effort: some data may already be written. https://nodejs.org/docs/latest/api/fs.html#fspromiseswritefilefile-data-options
- `writeFile` defaults: flag `'w'` (truncates), mode `0o666` (only for new files). Pass `flush: true` to `fsync` after writing, otherwise data may stay in the OS cache. https://nodejs.org/docs/latest/api/fs.html#fspromiseswritefilefile-data-options
- `readFile` accepts `buffer` (a preallocated Buffer or `(size) => Buffer`). The result is a view over it, and a buffer that is too small rejects. https://nodejs.org/docs/latest/api/fs.html#fspromisesreadfilepath-options
- Read relative to the module, not to `process.cwd()`: https://nodejs.org/docs/latest/api/fs.html#fspromisesreadfilepath-options

```js
const contents = await readFile(new URL('./package.json', import.meta.url), 'utf8');
```

- `createReadStream({ start, end })`: both bounds are inclusive. With `fd`, `path` is ignored and no `'open'` fires. A blocking character device can keep the process alive. https://nodejs.org/docs/latest/api/fs.html#fscreatereadstreampath-options
- Experimental or version-flagged: `fs.Utf8Stream` (v24.6.0), `filehandle.pull()`/`writer()` for `node:stream/iter` (added in v25.9.0), `fs.openAsBlobSync()` (added in v26.10.0). https://nodejs.org/docs/latest/api/fs.html#class-fsutf8stream , https://nodejs.org/docs/latest/api/fs.html#filehandlepulltransforms-options

## Directories, temp dirs, removal

- `mkdir({ recursive: true })` does not fail on an existing directory and returns the first created path or `undefined`. On Windows, `mode` is ignored and `mkdir` on the root fails even with recursion. https://nodejs.org/docs/latest/api/fs.html#fspromisesmkdirpath-options , https://nodejs.org/docs/latest/api/fs.html#fsmkdirpath-options-callback
- `mkdtemp` appends 6 characters to the prefix as is. End the prefix with a separator to create inside a directory, and avoid trailing `X` (BSDs may replace them). https://nodejs.org/docs/latest/api/fs.html#fsmkdtempprefix-options-callback

```js
await mkdtemp(join(tmpdir(), 'foo-'));  // /tmp/foo-itXde2
await mkdtemp(`${tmpDir}${sep}`);       // /tmp/abc123, not /tmpabc123
```

- Prefer `mkdtempDisposable()` (added v24.4.0): disposal removes the directory and its contents, and throws if it cannot. https://nodejs.org/docs/latest/api/fs.html#fspromisesmkdtempdisposableprefix-options

```js
await using dir = await fsPromises.mkdtempDisposable(join(tmpdir(), 'job-'));
```

- `rm -rf` is `rm(path, { recursive: true, force: true })`. `maxRetries` (default 0) and `retryDelay` (100 ms, linear) apply only with `recursive`. https://nodejs.org/docs/latest/api/fs.html#fspromisesrmpath-options
- `readdir({ recursive: true })` lists everything below. Get `Dirent`s from `readdir({ withFileTypes: true })` or `opendir`. https://nodejs.org/docs/latest/api/fs.html#fspromisesreaddirpath-options , https://nodejs.org/docs/latest/api/fs.html#class-fsdirent
- `realpath` does not normalize case on case-insensitive file systems. On Windows, `chmod` changes only the write bit. https://nodejs.org/docs/latest/api/fs.html#fsrealpathpath-options-callback , https://nodejs.org/docs/latest/api/fs.html#fschmodpath-mode-callback

## Glob and copy

- `fs.glob` / `fsPromises.glob` are stable since v24.0.0. Only the promise form returns an async iterator. `exclude` takes glob patterns but negation (`'!foo.js'`) is not supported. `followSymlinks` is added in v26.1.0 (default `false`) and `maxDepth` in v26.9.0. https://nodejs.org/docs/latest/api/fs.html#fspromisesglobpattern-options

```js
for await (const entry of glob('**/*.js')) console.log(entry);
```

- `fs.cp` is stable since v22.3.0. Defaults: `recursive: false`, `force: true` (overwrites). `filter` may return a promise, and skipping a directory skips its contents. No glob support. https://nodejs.org/docs/latest/api/fs.html#fspromisescpsrc-dest-options

## Watching

- `fs.watch` is not consistent across platforms, is unreliable on NFS, SMB and Docker or Vagrant host mounts, and gives no protection against malicious file changes. Never use it as a security control. https://nodejs.org/docs/latest/api/fs.html#caveats , https://nodejs.org/docs/latest/api/fs.html#availability
- On Linux and macOS a watch follows the inode: a deleted and recreated file is no longer watched. https://nodejs.org/docs/latest/api/fs.html#inodes
- `filename` can be `null` even on supported platforms: always have a fallback. https://nodejs.org/docs/latest/api/fs.html#filename-argument
- Stop a watch with `signal`. `persistent` (default true) keeps the process alive. `throwIfNoEntry` is added in v26.1.0. https://nodejs.org/docs/latest/api/fs.html#fswatchfilename-options-listener
- `fsPromises.watch` drops events past `maxQueue` (2048) with a warning unless `overflow: 'error'`. https://nodejs.org/docs/latest/api/fs.html#fspromiseswatchfilename-options
- Fall back to `fs.watchFile` (stat polling every 5007 ms) only where `fs.watch` fails. It fires on access, so compare `curr.mtimeMs` with `prev.mtimeMs`. https://nodejs.org/docs/latest/api/fs.html#fswatchfilefilename-options-listener

## Old patterns

| Old pattern | Replacement | Source |
| :-- | :-- | :-- |
| `fs.exists(path, cb)` | `fs.stat()` / `fs.access()`, or just open and handle `ENOENT` | https://nodejs.org/docs/latest/api/fs.html#fsexistspath-callback |
| `fs.rmdir(path, { recursive: true })` | `fs.rm(path, { recursive: true, force: true })` | https://nodejs.org/docs/latest/api/deprecations.html#DEP0147 |
| `fs.F_OK`, `fs.R_OK`... | `fs.constants.F_OK`... | https://nodejs.org/docs/latest/api/deprecations.html#DEP0176 |
| `dirent.path` | `dirent.parentPath` | https://nodejs.org/docs/latest/api/deprecations.html#DEP0178 |
| `new fs.Stats()` | `fs.stat()` | https://nodejs.org/docs/latest/api/deprecations.html#DEP0180 |
| object with own `toString` passed to `fs.write*` | pass a string | https://nodejs.org/docs/latest/api/deprecations.html#DEP0162 |
| async fs call without callback | always pass a callback (throws) | https://nodejs.org/docs/latest/api/deprecations.html#DEP0013 |
