# node:path, node:url and file paths

## Contents

- Platform semantics of node:path
- Path traversal
- path method surprises
- Paths accepted by fs
- WHATWG URL
- File URLs
- Query strings
- Old patterns

## Platform semantics of node:path

- `node:path` follows the host OS: `path.basename('C:\\temp\\myfile.html')` is the whole string on POSIX and `'myfile.html'` on Windows. For a fixed flavor use `path.win32` / `path.posix` (or `node:path/win32`, `node:path/posix`). https://nodejs.org/docs/latest/api/path.html#windows-vs-posix
- On Windows both `/` and `\` are accepted as input but only `\` is emitted, so never compare path strings built with different separators. https://nodejs.org/docs/latest/api/path.html#pathsep
- Split `PATH` with `path.delimiter` (`;` on Windows, `:` on POSIX). https://nodejs.org/docs/latest/api/path.html#pathdelimiter
- On Windows, `path.resolve('C:\\')` and `path.resolve('C:')` can differ (per-drive working directory). https://nodejs.org/docs/latest/api/path.html#windows-vs-posix

## Path traversal

- `path.isAbsolute()` only checks the literal string, so it is not safe for mitigating path traversals: `isAbsolute('/baz/../..')` is `true`. https://nodejs.org/docs/latest/api/path.html#pathisabsolutepath
- `path.join()` resolves `..`, so a user-supplied segment can climb out of the base. Resolve, then check the result is inside the intended root. https://nodejs.org/docs/latest/api/path.html#pathjoinpaths

```js
const root = path.resolve('/srv/files');
const target = path.resolve(root, userInput);
if (path.relative(root, target).startsWith('..') || path.isAbsolute(path.relative(root, target))) {
  throw new Error('outside root');
}
```
https://nodejs.org/docs/latest/api/path.html#pathresolvepaths , https://nodejs.org/docs/latest/api/path.html#pathrelativefrom-to

- `path.normalize()` is not strict POSIX: it collapses a leading `//` and removing `..` can change how the OS resolves a path through symlinks. https://nodejs.org/docs/latest/api/path.html#pathnormalizepath

## path method surprises

- `path.resolve()` works right to left and stops at the first absolute segment: `path.resolve('/foo', '/bar', 'baz')` is `/bar/baz`. Otherwise it falls back to `process.cwd()`. https://nodejs.org/docs/latest/api/path.html#pathresolvepaths
- `path.relative(from, to)` resolves both against the cwd first, and returns `''` for the same path. https://nodejs.org/docs/latest/api/path.html#pathrelativefrom-to
- `basename(p, suffix)` matches the suffix case-sensitively, even on Windows. https://nodejs.org/docs/latest/api/path.html#pathbasenamepath-suffix
- `extname('.index')` is `''`, `extname('index.')` is `'.'`, `extname('index.coffee.md')` is `'.md'`. https://nodejs.org/docs/latest/api/path.html#pathextnamepath
- `path.format()` ignores `root` when `dir` is set and ignores `name`/`ext` when `base` is set. https://nodejs.org/docs/latest/api/path.html#pathformatpathobject
- `path.matchesGlob(path, pattern)` is stable since v24.8.0 (experimental before). https://nodejs.org/docs/latest/api/path.html#pathmatchesglobpath-pattern

## Paths accepted by fs

- fs accepts strings (UTF-8), Buffers (for opaque POSIX byte paths) and `file:` URLs. Relative strings resolve against `process.cwd()`. https://nodejs.org/docs/latest/api/fs.html#file-paths , https://nodejs.org/docs/latest/api/fs.html#buffer-paths
- `file:` URLs containing `%2F` throw on every platform, `%5C` also on Windows. On Windows a hostname becomes a UNC path and a missing drive colon throws `ERR_INVALID_FILE_URL_PATH`. Elsewhere any hostname throws. https://nodejs.org/docs/latest/api/fs.html#platform-specific-considerations
- On Windows `readdirSync('C:\\')` and `readdirSync('C:')` can differ. https://nodejs.org/docs/latest/api/fs.html#per-drive-working-directories-on-windows

## WHATWG URL

- Parse untrusted input with `URL.parse(input, base)` (returns `null`) or test with `URL.canParse()`. `new URL()` throws `TypeError`, including on a relative input without base. https://nodejs.org/docs/latest/api/url.html#urlparseinput-base
- A server `req.url` is only path and query. Without a trusted reverse proxy, build the URL against a fixed origin rather than the client-controlled headers (the doc reads `x-forwarded-host` / `host` only behind a reverse proxy). https://nodejs.org/docs/latest/api/url.html#urlparseurlstring-parsequerystring-slashesdenotehost

```js
const url = new URL(`https://example.com${req.url || '/'}`);
```

- Setters silently ignore invalid values. A port string that starts with digits keeps the digits, out-of-range ports are ignored, and the protocol's default port serializes as `''`. https://nodejs.org/docs/latest/api/url.html#urlport
- `delete url.pathname` does nothing but returns `true`: properties are prototype accessors. https://nodejs.org/docs/latest/api/url.html#class-url
- Mutating through `searchParams` (even `sort()`) re-serializes the query, turning `~` into `%7E`. Replace the whole query with the `search` setter. https://nodejs.org/docs/latest/api/url.html#urlsearchparams
- `URLSearchParams.sort()` is stable, useful for canonical cache keys. https://nodejs.org/docs/latest/api/url.html#urlsearchparamssort
- `url.domainToASCII()` returns `''` for an invalid domain instead of throwing. Unicode hosts are converted to Punycode automatically. https://nodejs.org/docs/latest/api/url.html#urldomaintoasciidomain
- Pass a URL plus extra options to `http.request` with `url.urlToHttpOptions(url)`. https://nodejs.org/docs/latest/api/url.html#urlurltohttpoptionsurl
- `URL.createObjectURL(blob)` keeps the Blob in memory until `revokeObjectURL()`. Blob URLs are per thread. https://nodejs.org/docs/latest/api/url.html#urlcreateobjecturlblob
- `URLPattern` (added v23.8.0, global since v24.0.0) is experimental. https://nodejs.org/docs/latest/api/globals.html#class-urlpattern

## File URLs

- Convert paths with `url.pathToFileURL()` and `url.fileURLToPath()`, never `new URL(path)` or concatenation: `new URL('/foo#1', 'file:')` keeps `#1` as a fragment while `pathToFileURL('/foo#1')` gives `file:///foo%231`. https://nodejs.org/docs/latest/api/url.html#urlpathtofileurlpath-options
- Both accept `{ windows: true | false }` to force a flavor. `url.fileURLToPathBuffer()` (added v24.3.0) returns a Buffer for non-UTF-8 paths. https://nodejs.org/docs/latest/api/url.html#urlfileurltopathurl-options , https://nodejs.org/docs/latest/api/url.html#urlfileurltopathbufferurl-options

## Query strings

- Default to `URLSearchParams` (standard, browser-compatible). `node:querystring` is more performant but not standardized, and accepts custom `sep` and `eq` delimiters. https://nodejs.org/docs/latest/api/querystring.html
- `querystring.parse()` returns a null-prototype object (`hasOwnProperty` is missing), turns repeated keys into arrays and stops after `maxKeys` keys (default 1000, `0` lifts the limit). https://nodejs.org/docs/latest/api/querystring.html#querystringparsestr-sep-eq-options

## Old patterns

| Old pattern | Replacement | Source |
| :-- | :-- | :-- |
| `url.parse()` (host spoofing, no CVEs issued; also covers `url.format(string)` and `url.resolve()`) | `new URL()` / `URL.parse()` | https://nodejs.org/docs/latest/api/deprecations.html#dep0169-insecure-urlparse |
| Legacy URL API as a whole | codemod `npx codemod@latest @nodejs/node-url-to-whatwg-url` | https://nodejs.org/docs/latest/api/deprecations.html#dep0116-legacy-url-api |
| `url.resolve(from, to)` | `new URL(to, from)` | https://nodejs.org/docs/latest/api/url.html#urlresolvefrom-to |
| `url.format(urlString)` | `new URL(...).toString()` | https://nodejs.org/docs/latest/api/url.html#urlformaturlstring |
| `require('punycode')` | npm `punycode`, or `url.domainToASCII()` | https://nodejs.org/docs/latest/api/punycode.html |
| `__dirname` + `path.join` in ESM | `import.meta.dirname` (only on `file:` modules) or `new URL('./x', import.meta.url)` | https://nodejs.org/docs/latest/api/esm.html#importmetadirname |
