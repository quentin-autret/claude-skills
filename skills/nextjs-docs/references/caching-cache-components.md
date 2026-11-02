# Caching with Cache Components

`use cache`, `cacheLife`, `cacheTag`, `updateTag`, `revalidateTag`, `revalidatePath`, `refresh` and cache handlers in Next.js 16.4.4.

## Contents

- Enabling and the default model
- `use cache` rules and cache keys
- Runtime data and the static shell
- `cacheLife` profiles and prerendering
- Invalidation: `updateTag`, `revalidateTag`, `revalidatePath`, `refresh`
- `use cache: private` and `use cache: remote`
- Cache handlers and multi-instance
- Debugging and platform limits

## Enabling and the default model

```ts
// next.config.ts
const nextConfig = { cacheComponents: true, partialPrefetching: true }
```
https://nextjs.org/docs/app/getting-started/caching#enabling-cache-components

- Set `partialPrefetching` explicitly. Unset logs a warning. `false` only while migrating an app that adopted Cache Components before Partial Prefetching.
  https://nextjs.org/docs/app/getting-started/caching#enabling-cache-components
- Data is dynamic by default. Caching is opt-in with `use cache` on a file, component or function.
  https://nextjs.org/docs/app/api-reference/config/next-config-js/cacheComponents
- Cache Components requires the Node.js runtime: delete every `runtime` segment export, even `'nodejs'`.
  https://nextjs.org/docs/app/api-reference/config/next-config-js/cacheComponents

## `use cache` rules and cache keys

```tsx
export async function getUsers() {
  'use cache'
  cacheLife('hours')
  cacheTag('users')
  return db.query('SELECT * FROM users')
}
```
https://nextjs.org/docs/app/getting-started/caching#data-level-caching

- Cached functions must be `async`. File-level `'use cache'` caches every export, `generateMetadata` and `generateStaticParams` included.
  https://nextjs.org/docs/app/api-reference/directives/use-cache#caching-a-modules-exports-with-use-cache
- Cache key: build ID, function ID, arguments and captured closure variables. Every deploy starts empty.
  https://nextjs.org/docs/app/api-reference/directives/use-cache#cache-keys
- Arguments must be serializable: no class instances, functions, symbols or `URL` objects. Return values may be JSX.
  https://nextjs.org/docs/app/api-reference/directives/use-cache#serialization
- `children` pass through a cached component without joining the key, as long as the body never reads them.
  https://nextjs.org/docs/app/api-reference/directives/use-cache#interleaving
- `React.cache` values set outside are invisible inside a `use cache` scope. Pass data as arguments.
  https://nextjs.org/docs/app/api-reference/directives/use-cache#reactcache-isolation
- `use cache` cannot sit directly in a Route Handler body. Call a cached helper.
  https://nextjs.org/docs/app/getting-started/route-handlers#with-cache-components

## Runtime data and the static shell

- `cookies()`, `headers()` or `searchParams` inside `use cache` fail with `next-request-in-use-cache`, even from a nested helper. On dynamic routes this can pass `next build` and fail under `next start`.
  https://nextjs.org/docs/app/api-reference/directives/use-cache#request-time-apis
- Default: read runtime data outside the scope and pass the value. It joins the key.
  ```tsx
  const session = (await cookies()).get('session')?.value
  return <CachedContent sessionId={session} />
  ```
  https://nextjs.org/docs/app/getting-started/caching#passing-runtime-values-to-cached-functions
- Passing an unresolved Promise of runtime data into a cached scope hangs the build ("Filling a cache during prerender timed out", 50 s). Await outside.
  https://nextjs.org/docs/app/api-reference/directives/use-cache#build-hangs-cache-timeout
- Uncached or runtime reads go inside `<Suspense>`. Without it the dev overlay reports "blocking-route". The fallback ships in the static shell.
  https://nextjs.org/docs/app/getting-started/caching#streaming-uncached-data
- `Math.random()`, `Date.now()`, `crypto.randomUUID()`: `await connection()` plus Suspense, or cache the value.
  https://nextjs.org/docs/app/getting-started/caching#random-values-and-timestamps

## `cacheLife` profiles and prerendering

- Pair every `use cache` scope with an explicit `cacheLife` in the same function. Omitted, the `default` profile applies (stale 5m, revalidate 15m, expire never).
  https://nextjs.org/docs/app/api-reference/functions/cacheLife#basic-setup
- Presets (stale / revalidate / expire): `seconds` 30s/1s/60s, `minutes` 5m/1m/1h, `hours` 5m/1h/1d, `days` 5m/1d/1w, `weeks` 5m/1w/30d, `max` 5m/30d/1y.
  https://nextjs.org/docs/app/api-reference/functions/cacheLife#preset-cache-profiles
- One `cacheLife` call per invocation. Branches are fine: `cacheLife('minutes')` for a draft, `cacheLife('days')` when published.
  https://nextjs.org/docs/app/api-reference/functions/cacheLife#conditional-cache-lifetimes
- `cacheLife` at module scope throws. `expire` must exceed `revalidate`.
  https://nextjs.org/docs/app/api-reference/functions/cacheLife#expire
- Reusable lifetimes are named profiles in `next.config.ts`. Redefining built-ins is supported, but a new name surprises readers less.
  https://nextjs.org/docs/app/api-reference/functions/cacheLife#custom-cache-profiles
- Short lifetimes become dynamic holes: `revalidate: 0`, `expire` under 5 min or `stale` under 30 s. Of the presets only `seconds` is excluded from prerendering.
  https://nextjs.org/docs/app/api-reference/functions/cacheLife#prerendering-behavior
- Nested caches: an explicit outer `cacheLife` wins. Without one, a shorter inner lifetime shortens the outer. A short-lived inner inside an outer with no `cacheLife` fails the build.
  https://nextjs.org/docs/app/api-reference/functions/cacheLife#nested-short-lived-caches
- The client router enforces at least 30 s `stale`. Any `revalidateTag`, `revalidatePath`, `updateTag` or `refresh` in a Server Action clears the whole client cache.
  https://nextjs.org/docs/app/api-reference/functions/cacheLife#client-cache-behavior

## Invalidation

| Need | Call | Where |
| --- | --- | --- |
| Read-your-own-writes after a mutation | `updateTag('posts')` | Server Actions only |
| Stale-while-revalidate (webhook, CMS) | `revalidateTag('posts', 'max')` | Server Actions, Route Handlers |
| Expire now from a webhook | `revalidateTag('posts', { expire: 0 })` | Route Handlers |
| Refresh the client router, no tagged data revalidated | `refresh()` | Server Actions only |
| One route | `revalidatePath('/blog')` | Server Actions, Route Handlers |

https://nextjs.org/docs/app/getting-started/revalidating#revalidatetag and https://nextjs.org/docs/app/guides/client-side-data-fetching#coordinate-mutations and https://nextjs.org/docs/app/getting-started/mutating-data

- Prefer tags over `revalidatePath`: precise, no over-invalidation. CMS content: `cacheTag` plus `cacheLife('max')`, invalidated from a webhook.
  https://nextjs.org/docs/app/getting-started/revalidating#what-should-i-cache
- `revalidateTag(tag)` with one argument is deprecated (a TS error) and acts like `{ expire: 0 }`.
  https://nextjs.org/docs/app/api-reference/functions/revalidateTag#revalidation-behavior
- `revalidateTag` marks data stale; the next request starts the revalidation. Not callable in Client Components or Proxy.
  https://nextjs.org/docs/app/api-reference/functions/revalidateTag#revalidation-behavior
- Tags are case-sensitive, at most 256 characters, 128 per `cacheTag` call. Longer tags and tags past the 128th are skipped with a console warning, so revalidating them does nothing.
  https://nextjs.org/docs/app/api-reference/functions/cacheTag#good-to-know
- `revalidatePath`: never append `/page` or `/layout`. Pass `type` (`'page'` or `'layout'`) when the path has a dynamic segment such as `/product/[slug]`.
  https://nextjs.org/docs/app/api-reference/functions/revalidatePath#parameters
- With rewrites, pass the destination path, not the visible URL.
  https://nextjs.org/docs/app/api-reference/functions/revalidatePath#using-revalidatepath-with-rewrites
- `revalidatePath` refreshes only the given path. Other pages using the same tag keep serving cached data until the tag is revalidated.
  https://nextjs.org/docs/app/api-reference/functions/revalidatePath#relationship-with-revalidatetag-and-updatetag

## `use cache: private` and `use cache: remote`

- `use cache: private` may read `cookies()`, `headers()` and `searchParams`, never `connection()`. In production the result is not stored in a server cache across requests (the client router may keep it in browser memory for the `stale` time). Excluded from the static shell.  https://nextjs.org/docs/app/api-reference/directives/use-cache-private
- `use cache: remote` stores in the remote handler, shared across instances. Worth it for slow or rate-limited backends. Avoid for work under about 50 ms.
  https://nextjs.org/docs/app/api-reference/directives/use-cache-remote#when-to-avoid-remote-caching
- Remote inside private, or private inside remote, is an error. No directive persists across deploys: use `fetch` cache or `unstable_cache` for that.
  https://nextjs.org/docs/app/api-reference/directives/use-cache-remote#nesting-rules

## Cache handlers and multi-instance

- `cacheHandlers` (plural) keys: `default` for `use cache`, `remote` for `use cache: remote`, any other name for `'use cache: <name>'`.  https://nextjs.org/docs/app/api-reference/config/next-config-js/cacheHandlers#handler-types
- Revalidation is local per instance. Implement `updateTags()` and `refreshTags()` against shared storage, and catch errors in `refreshTags()`.
  https://nextjs.org/docs/app/guides/how-revalidation-works#multi-instance-considerations

## Debugging and platform limits

- `NEXT_PRIVATE_DEBUG_CACHE=1` logs cache activity.
  https://nextjs.org/docs/app/api-reference/directives/use-cache#debugging-cache-behavior
- On serverless the default in-memory cache rarely survives between requests. Static export does not support `use cache`.
  https://nextjs.org/docs/app/api-reference/directives/use-cache#runtime-caching-considerations
