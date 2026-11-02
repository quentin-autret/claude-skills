# Data fetching, streaming and Server Actions

Reading data in Server Components, streaming with Suspense, mutating with Server Actions, forms, `after()` and client data libraries in Next.js 16.4.4.

## Contents

- Reading data
- Streaming and the HTTP contract
- Server Actions basics
- Updating the UI after a mutation
- Server Action security
- Deployment pitfalls of Server Actions
- `<Form>` and `after()`
- Client data libraries

## Reading data

- Fetch in async Server Components with `fetch` or the ORM directly. `fetch` is not cached by default and blocks rendering: cache it with `use cache` or stream it inside `<Suspense>`.
  https://nextjs.org/docs/app/getting-started/fetching-data#with-the-fetch-api
- Never call your own Route Handlers from Server Components: prerender fails at build (no server) and runtime pays an extra HTTP round trip.
  https://nextjs.org/docs/app/guides/backend-for-frontend#server-components
- Identical `fetch` calls in one render are memoized, so fetch where the data is needed. For ORM calls, wrap the function in React `cache` (one request scope only).
  https://nextjs.org/docs/app/getting-started/fetching-data#reusing-data-with-reactcache
- Start independent requests first, then `Promise.all`. Sequential awaits are a waterfall.
  ```tsx
  const artistData = getArtist(username)
  const albumsData = getAlbums(username)
  const [artist, albums] = await Promise.all([artistData, albumsData])
  ```
  https://nextjs.org/docs/app/getting-started/fetching-data#parallel-data-fetching
- Preload: call `void getItem(id)` before blocking work, then call it again in the consumer. The function must dedupe (`fetch`, React `cache` or `use cache`).
  https://nextjs.org/docs/app/getting-started/fetching-data#preloading-data
- Do not read data with Server Actions: the client queues them one at a time.
  https://nextjs.org/docs/app/guides/backend-for-frontend#server-actions

## Streaming and the HTTP contract

- Prefer an explicit `<Suspense>` next to the uncached access over `loading.js`, which only wraps `page.js`.
  https://nextjs.org/docs/app/guides/streaming
- Awaiting `params`, `searchParams`, `cookies()` or `headers()` at the top of a layout or page makes everything below dynamic. Pass the promise down and resolve it inside Suspense.
  https://nextjs.org/docs/app/guides/streaming#push-dynamic-access-down
- Once streaming starts, status and headers are fixed. A mid-stream `notFound()` injects `noindex`, a mid-stream `redirect()` becomes a client redirect. Call `notFound()` before any await that suspends.
  https://nextjs.org/docs/app/guides/streaming#the-http-contract
- Buffering defeats streaming: nginx (send `X-Accel-Buffering: no`), CDNs, compression, Safari under 1024 bytes.
  https://nextjs.org/docs/app/guides/streaming#what-can-affect-streaming
- HTML-limited bots wait for `generateMetadata` to resolve before page content streams, so metadata lands in the initial `<head>`. Tune who counts as one with `htmlLimitedBots`.
  https://nextjs.org/docs/app/guides/streaming#bots-and-crawlers

## Server Actions basics

- `'use server'` at the top of an async function, or at the top of a file for all exports. Client Components cannot define them: import from a `'use server'` file or receive one as a prop.
  https://nextjs.org/docs/app/getting-started/mutating-data#client-components
- Extra arguments: `action.bind(null, userId)`. Hidden inputs put the value in the HTML, unencoded.
  https://nextjs.org/docs/app/guides/forms#passing-additional-arguments
- With `useActionState`, the action receives `(prevState, formData)`. Its third value is `pending`.
  https://nextjs.org/docs/app/guides/forms#validation-errors
- Validate with Zod or Valibot on the server and return `{ errors }` early instead of throwing.
  https://nextjs.org/docs/app/guides/forms#form-validation
- `Object.fromEntries(formData)` also contains `$ACTION_`-prefixed keys.
  https://nextjs.org/docs/app/guides/forms#how-it-works
- Server Component forms work without JavaScript. In Client Components, submissions queue until hydration.
  https://nextjs.org/docs/app/getting-started/mutating-data#server-components
- The client dispatches actions one at a time. Never `Promise.all` actions from the client: do parallel work inside one action.
  https://nextjs.org/docs/app/guides/server-actions#sequential-dispatch-on-the-client

## Updating the UI after a mutation

- `updateTag(tag)` for read-your-own-writes. `revalidateTag(tag, 'max')` serves stale data and its re-render is not in the action response. `refresh()` refetches the current route's RSC Payload without invalidating cached data.
  https://nextjs.org/docs/app/guides/server-actions#choosing-a-cache-update
- Call `revalidate*` and `updateTag` before `redirect()`: `redirect` throws, nothing after it runs.
  https://nextjs.org/docs/app/getting-started/mutating-data#redirect-after-a-mutation
- One action response carries the return value and fresh RSC payload when the action calls `updateTag`, `revalidatePath`, `refresh`, sets cookies or redirects. Otherwise nothing re-renders.
  https://nextjs.org/docs/app/guides/server-actions#a-single-response-carries-data-and-ui
- Setting or deleting a cookie in an action re-renders the current page and layouts.  https://nextjs.org/docs/app/getting-started/mutating-data#cookies

## Server Action security

- Every exported action is reachable by a direct POST, even when the form only renders for signed-in users. Authenticate and authorize inside each action.
  https://nextjs.org/docs/app/guides/server-actions#security
- Zod checks shape, not ownership. Send an ID, then re-read the row with the session and an ownership filter.
  https://nextjs.org/docs/app/guides/server-actions#security
- Read authentication from cookies or headers rather than accepting tokens as action parameters.
  https://nextjs.org/docs/app/api-reference/directives/use-server#authentication-and-authorization
- Return only what the UI needs, never raw rows: return values are serialized to the client.
  https://nextjs.org/docs/app/api-reference/directives/use-server#return-values
- CSRF: the `Origin` header must match `Host` or `X-Forwarded-Host`. Behind a proxy with another host, list it in `serverActions.allowedOrigins` (host only, `*` one label, `**` several, only at the start).
  https://nextjs.org/docs/app/api-reference/config/next-config-js/serverActions#allowedorigins
- Request bodies are limited to 1 MB (`serverActions.bodySizeLimit`, for example `'3mb'`), multipart overhead included.
  https://nextjs.org/docs/app/api-reference/config/next-config-js/serverActions#bodysizelimit

```ts
// next.config.ts
experimental: { serverActions: { allowedOrigins: ['my-proxy.com'], bodySizeLimit: '3mb' } }
```
https://nextjs.org/docs/app/api-reference/config/next-config-js/serverActions

## Deployment pitfalls of Server Actions

- New deployments typically generate new action IDs (rotated at most every 14 days even with unchanged source). A stale tab gets "Failed to find Server Action". Prefer rolling deploys and a retry path in the UI.
  https://nextjs.org/docs/app/guides/server-actions#deployment-considerations
- Variables captured by an inline action are encrypted before being sent to the client. With several instances, set one `NEXT_SERVER_ACTIONS_ENCRYPTION_KEY` for all.
  https://nextjs.org/docs/app/guides/server-actions#security

## `<Form>` and `after()`

- `<Form>` from `next/form` with a string `action` is a GET form with prefetch and client navigation. With a function `action`, `replace` and `scroll` are ignored. `method`, `encType` and `target` are unsupported.
  https://nextjs.org/docs/app/api-reference/components/form#caveats
- `after(cb)` from `next/server` runs logging after the response, in Server Components, Server Actions, Route Handlers and Proxy. It runs even after errors or `redirect`.
  ```tsx
  after(() => { log() })
  ```
  https://nextjs.org/docs/app/api-reference/functions/after
- Inside `after` in a Server Component, `cookies()` and `headers()` throw: read them before and close over the value. `after` does not make a route dynamic.
  https://nextjs.org/docs/app/api-reference/functions/after

## Client data libraries

- Use SWR or TanStack Query only when Client Components need a shared browser cache (polling, focus revalidation). For a one-time read, pass a promise and `use()` it.
  https://nextjs.org/docs/app/guides/client-side-data-fetching
- SWR `<SWRConfig fallback>` keys must match the `useSWR` key exactly or the fallback is ignored.
  https://nextjs.org/docs/app/guides/client-side-data-fetching/swr
- TanStack: one new `QueryClient` per server render, one reused in the browser. With Cache Components, `dehydrate()` reads `Date.now()` and fails prerender: use the documented prerenderable helper.
  https://nextjs.org/docs/app/guides/client-side-data-fetching/tanstack-query#build-a-prerenderable-hydration-state
- Three caches (server, client router, library) have independent freshness. Coordinate keys and invalidation.
  https://nextjs.org/docs/app/guides/client-side-data-fetching#cache-server-data-with-cache-components-optional
