# Server rendering, static prerendering and resume

## Contents

- Picking the API
- renderToPipeableStream
- renderToReadableStream
- Status codes and errors
- prerender and partial prerendering
- Doc errors not to copy

## Picking the API

| Need | Node.js | Web Streams (Deno, edge) |
| ---- | ------- | ------------------------ |
| Streaming SSR | `renderToPipeableStream` | `renderToReadableStream` |
| Static generation, waits for all data | `prerenderToNodeStream` | `prerender` |
| Resume a prerender into a response | `resumeToPipeableStream` | `resume` |
| Resume a prerender into static HTML | `resumeAndPrerenderToNodeStream` | `resumeAndPrerender` |

https://react.dev/reference/react-dom/server and https://react.dev/reference/react-dom/static

- On Node, use the Node stream APIs. The Web Stream versions run there for compatibility only: slower and without compression by default. https://react.dev/reference/react-dom/server and https://react.dev/blog/2025/10/01/react-19-2
- `renderToString` is legacy: no streaming, no waiting for data, a suspending component emits its fallback. Keep it only where no streams exist. https://react.dev/reference/react-dom/server/renderToString
- `renderToStaticMarkup` gives HTML that cannot be hydrated, for static pages and emails. https://react.dev/reference/react-dom/server/renderToStaticMarkup
- Never import `react-dom/server` in client code. https://react.dev/reference/react-dom/server/renderToString
- Only Suspense-enabled sources (a Promise read with `use`) are awaited, never fetches in Effects or handlers. https://react.dev/reference/react-dom/static/prerender

## renderToPipeableStream

- The root renders the whole document with `<html>`. React adds the doctype and bootstrap scripts. https://react.dev/reference/react-dom/server/renderToPipeableStream
- `pipe` in `onShellReady` to stream, in `onAllReady` for crawlers and static output. https://react.dev/reference/react-dom/server/renderToPipeableStream
- The shell is everything outside `<Suspense>`. Make it minimal but complete: a root-level Suspense yields only a spinner. https://react.dev/reference/react-dom/server/renderToPipeableStream
- An overridden `onError` should still call `console.error`. https://react.dev/reference/react-dom/server/renderToPipeableStream
- `abort()` after a timeout flushes remaining fallbacks and leaves the rest to the client. https://react.dev/reference/react-dom/server/renderToPipeableStream
- `onHeaders` receives `{ Link }` preload hints, usable for 103 Early Hints. `maxHeadersLength` caps them (default 2000 UTF-16 units). https://react.dev/reference/react-dom/server/renderToPipeableStream
- `nonce` takes a string or `{ script, style }`. `onBrowserBailout` reports components that called `use(browser())`. https://react.dev/reference/react-dom/server/renderToPipeableStream
- Hashed asset names: pass `assetMap` as a prop and serialize it with `bootstrapScriptContent` so client and server render the same props. https://react.dev/reference/react-dom/server/renderToPipeableStream

## renderToReadableStream

- `await renderToReadableStream(node, options)` resolves when the shell is ready and rejects when it fails: wrap it in `try`/`catch` and return fallback HTML with status 500. https://react.dev/reference/react-dom/server/renderToReadableStream
- No `onShellReady`, `onAllReady` or `onShellError`. Abort with `signal`, wait with `await stream.allReady` for crawlers. https://react.dev/reference/react-dom/server/renderToReadableStream
- Its `onHeaders` gives a `Headers` instance. https://react.dev/reference/react-dom/server/renderToReadableStream

## Status codes and errors

- The status cannot change once streaming starts. Set it in `onShellReady` from a `didError` flag set in `onError`, and in `onShellError`. https://react.dev/reference/react-dom/server/renderToPipeableStream
- A shell failure fires `onError` and `onShellError`, sends no bytes, and needs a fallback HTML. https://react.dev/reference/react-dom/server/renderToPipeableStream
- An error inside a Suspense boundary emits the fallback and retries on the client. If the retry works, the user sees nothing, but server `onError` and client `onRecoverableError` both fire. https://react.dev/reference/react-dom/server/renderToPipeableStream
- Move content into the shell if its errors must set the status. Error subclasses (`NotFoundError`) can map to codes. https://react.dev/reference/react-dom/server/renderToPipeableStream
- Browser bailouts go to `onBrowserBailout`, not `onError` nor `onRecoverableError`. Without a Suspense boundary the server render fails and is reported through the usual error callbacks. https://react.dev/reference/react-dom/browser

## prerender and partial prerendering

- `const { prelude, postponed } = await prerender(node, options)` waits for all data, then returns the HTML stream and `postponed`, which is `null` when complete. https://react.dev/reference/react-dom/static/prerender
- Prerender APIs take no `nonce`: a nonce must be unique per request. Pass it to `resume`, and only if no scripts went to `prerender`. https://react.dev/reference/react-dom/static/prerender and https://react.dev/reference/react-dom/server/resume
- Aborting through `signal` keeps what is ready, and incomplete boundaries become fallbacks. Store `postponed` (JSON-serializable) and continue with `resume` or `resumeAndPrerender`. https://react.dev/reference/react-dom/static/prerender
- `resume` takes the same node as `prerender`, no bootstrap options and no `identifierPrefix`, which must match `prerender`. It re-renders from the root and skips only fully prerendered subtrees. https://react.dev/reference/react-dom/server/resume
- `prerenderToNodeStream` streams nothing until the whole app has rendered, by design. https://react.dev/reference/react-dom/static/prerenderToNodeStream
- Read `prelude` into a string by looping over `prelude.getReader()`. https://react.dev/reference/react-dom/static/prerender

## Doc errors not to copy

- The `assetMap` examples index `assetMap['/main.js']` while the key is `'main.js'`. https://react.dev/reference/react-dom/server/renderToReadableStream
- Some notes say `resumeToNodeStream`: the Node API is `resumeToPipeableStream`. https://react.dev/reference/react-dom/server/resume
- `resumeAndPrerender` options are only `signal`, `onBrowserBailout`, `onError`, though its example passes `bootstrapScripts`. https://react.dev/reference/react-dom/static/resumeAndPrerender
- Serve HTML as `text/html`, not the `text/plain` of one demo. https://react.dev/reference/react-dom/static/prerenderToNodeStream
