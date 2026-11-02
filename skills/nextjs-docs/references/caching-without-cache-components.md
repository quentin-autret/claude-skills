# Caching without Cache Components, and migrating

The previous caching model (`fetch` options, `unstable_cache`, `dynamic`, `revalidate`, ISR), CDN headers, and how each piece maps to Cache Components in Next.js 16.4.4.

## Contents

- `fetch` caching defaults
- `unstable_cache` and request deduplication
- Route segment config of the previous model
- ISR caveats and debugging
- Client router cache and CDN
- Migrating to Cache Components
- `connection()`, `io()` and sync IO

## `fetch` caching defaults

- `fetch` is not cached by default. Opt in per request with `cache: 'force-cache'`. This also caches POST and requests carrying authorization or cookie headers.
  https://nextjs.org/docs/app/api-reference/functions/fetch#optionscache
- A `fetch` with no `cache` option runs once at `next build` if reached before any Request-time API, on every request after one.
  https://nextjs.org/docs/app/guides/caching-without-cache-components#fetchcache
- `force-cache` matches URL, method, headers and body. Only 200 responses are stored.
  https://nextjs.org/docs/app/api-reference/functions/fetch#optionscache
- `{ revalidate: 3600, cache: 'no-store' }` is invalid: both are ignored with a dev warning.
  https://nextjs.org/docs/app/api-reference/functions/fetch#optionsnextrevalidate
- The lowest `revalidate` across the layouts, pages and fetches of a route sets its frequency.
  https://nextjs.org/docs/app/guides/caching-without-cache-components#revalidation-frequency
- One fetch with `revalidate: 0` or an explicit `no-store` makes the route dynamically rendered.
  https://nextjs.org/docs/app/guides/incremental-static-regeneration#caveats
- Same-URL GET fetches dedupe within one render pass only, never in Route Handlers.
  https://nextjs.org/docs/app/api-reference/functions/fetch#memoization
- In dev the HMR cache serves stale fetch results, even for `no-store`, until navigation or full reload.
  https://nextjs.org/docs/app/api-reference/functions/fetch#troubleshooting

## `unstable_cache` and request deduplication

```ts
export const getCachedUser = unstable_cache(
  async (id: string) => db.user.find(id),
  ['user'],
  { tags: ['user'], revalidate: 3600 },
)
```
https://nextjs.org/docs/app/api-reference/functions/unstable_cache

- Pass cookies and headers values as arguments, never read them inside.
  https://nextjs.org/docs/app/guides/caching-without-cache-components#unstable_cache-for-non-fetch-functions
- `keyParts` only matter for closed-over variables not passed as parameters. `tags` do not identify the function. Omit `revalidate` to cache indefinitely.
  https://nextjs.org/docs/app/api-reference/functions/unstable_cache
- Inside `unstable_cache`, `connection()`, draft mode toggles and revalidation calls are forbidden. `unstable_noStore` inside it does not opt out: the cache configuration decides.
  https://nextjs.org/docs/app/api-reference/functions/unstable_noStore
- Unlike `use cache`, the `fetch` Data Cache and `unstable_cache` persist across deployments and instances.
  https://nextjs.org/docs/app/guides/migrating-to-cache-components#fetch-cache-options
- Dedupe non-fetch reads within one render with React `cache()`.
  https://nextjs.org/docs/app/guides/caching-without-cache-components#deduplicating-requests
- To preload, call a `cache()`-wrapped getter early (`void getItem(id)`), with `import 'server-only'` in the utility module.
  https://nextjs.org/docs/app/guides/caching-without-cache-components#preloading-data

## Route segment config of the previous model

- `dynamic = 'force-static'` makes `cookies()`, `headers()` and `useSearchParams()` return empty values. `'error'` fails on any Request-time API. `'force-dynamic'` equals `no-store` on every fetch.
  https://nextjs.org/docs/app/guides/caching-without-cache-components#dynamic
- Route `revalidate` must be a literal: `600` works, `60 * 10` does not. It does not override a fetch's own `revalidate`.
  https://nextjs.org/docs/app/guides/caching-without-cache-components#route-segment-config-revalidate
- Leave shared parent layouts at `'auto'` and configure only the child segments that diverge.
  https://nextjs.org/docs/app/guides/caching-without-cache-components#cross-route-segment-behavior
- Route Handler `GET` is not cached by default. Opt in with `export const dynamic = 'force-static'`.
  https://nextjs.org/docs/app/getting-started/route-handlers#caching
- Prefer a high `revalidate` (an hour, not a second). Need precision: on-demand revalidation.
  https://nextjs.org/docs/app/guides/incremental-static-regeneration#time-based-revalidation

## ISR caveats and debugging

- ISR is Node.js runtime only, absent from static export. The default file-system cache is per instance: several instances need a shared cache handler.
  https://nextjs.org/docs/app/guides/incremental-static-regeneration#caveats
- Proxy does not run for on-demand ISR: revalidate the exact path, not a rewritten one.
  https://nextjs.org/docs/app/guides/incremental-static-regeneration#caveats
- A failed revalidation keeps serving the last good data and retries on the next request.
  https://nextjs.org/docs/app/guides/incremental-static-regeneration#handling-uncaught-exceptions
- Debug with the `x-nextjs-cache` header (HIT, STALE, MISS, REVALIDATED), `NEXT_PRIVATE_DEBUG_CACHE=1` and `logging.fetches.fullUrl`.
  https://nextjs.org/docs/app/guides/incremental-static-regeneration#troubleshooting

## Client router cache and CDN

- `staleTimes` is experimental: `dynamic` defaults to 0 s, `static` to 5 min.
  https://nextjs.org/docs/app/api-reference/config/next-config-js/staleTimes
- Behind a CDN, call the CDN purge API (HTML and RSC variants) after `revalidateTag` or `revalidatePath`, otherwise it serves until `s-maxage` expires.
  https://nextjs.org/docs/app/guides/cdn-caching#cache-control-headers
- Forward the `rsc` header and keep the `_rsc` search param in the CDN cache key. Without `next-url`, interception routes are not supported (the target page shows instead).
  https://nextjs.org/docs/app/guides/cdn-caching#handling-headers-at-the-cdn

## Migrating to Cache Components

| Previous | Cache Components |
| --- | --- |
| `dynamic = 'force-dynamic'` | delete it, uncached is the default |
| `dynamic = 'force-static'` | `use cache` + `cacheLife('max')` |
| `revalidate = 3600` | `cacheLife('hours')` |
| `fetchCache` | `use cache`, or leave uncached |
| fetch `next.revalidate` / `next.tags` | `cacheLife` / `cacheTag` |
| `unstable_cache` | `use cache` + `cacheLife` + `cacheTag` (optional, still works) |
| `unstable_noStore()` | delete it, or `connection()` inside Suspense |
| `dynamicParams = false` | `notFound()` for unknown params |

https://nextjs.org/docs/app/guides/migrating-to-cache-components

```tsx
async function getData() {
  'use cache'
  cacheLife('hours')
  cacheTag('data')
  const res = await fetch(url)
  return res.json()
}
```
https://nextjs.org/docs/app/guides/migrating-to-cache-components

- With the flag on, `dynamic`, `revalidate` and `fetchCache` exports error, `dynamicParams` fails the build, and `runtime` must go.
  https://nextjs.org/docs/app/guides/migrating-to-cache-components#enable-cache-components
- `generateStaticParams` returning `[]` errors (`empty-generate-static-params`). Return at least 1 param.
  https://nextjs.org/docs/app/guides/migrating-to-cache-components#generatestaticparams-and-dynamicparams
- `generateMetadata` or `generateViewport` reading runtime or uncached data on an otherwise prerenderable page errors. Add `use cache` for external data, or a dynamic marker component (`<Suspense><Connection /></Suspense>`, awaiting `connection()`) for runtime data.
  https://nextjs.org/docs/app/guides/migrating-to-cache-components#generatemetadata-and-generateviewport
- A cookie or header driving `<html>` attributes in the root layout makes the shell request-bound. Use an inline head script.
  https://nextjs.org/docs/app/guides/migrating-to-cache-components#cookies-headers-and-searchparams
- Reading uncached or runtime data in a `GET` handler bails out of prerender by throwing. An existing try/catch catches it, and a logging `catch` adds build noise (`experimental.hideLogsAfterAbort: true` hides it).
  https://nextjs.org/docs/app/guides/migrating-to-cache-components#route-handlers-get
- Adopt route by route with `export const instant = false`. It only allows blocking, it does not force dynamic. Validation results show in the dev overlay, never in the HTTP status.
  https://nextjs.org/docs/app/guides/migrating-to-cache-components#opting-out-of-validation
- ISR with Cache Components: with Partial Prefetching, await `params` inside Suspense so unlisted paths get the fallback instantly. Without it, await `params` outside any Suspense (including `loading.tsx`) or no URL-specific ISR entry is created. Links to unlisted pages need `prefetch={true}`.
  https://nextjs.org/docs/app/guides/incremental-static-regeneration-cache-components

## `connection()`, `io()` and sync IO

- `new Date()`, `Date.now()`, `Math.random()` or `crypto.randomUUID()` during prerender fail the build, and `instant = false` does not clear it.
  https://nextjs.org/docs/app/guides/migrating-to-cache-components#adopting-incrementally
- Default fix: `await io()` (from `next/cache`) before the read, inside Suspense. It suspends only during prerender, and code after it stays cacheable. Client Components use `use(io())`.
  https://nextjs.org/docs/app/api-reference/functions/io
- `io()` is not needed after `cookies()`, `headers()` or an awaited fetch or query.
  https://nextjs.org/docs/app/api-reference/functions/io
- `connection()` only when the work must wait for a real user request, for example sync DB drivers such as better-sqlite3.
  https://nextjs.org/docs/app/api-reference/functions/connection#synchronous-database-drivers
