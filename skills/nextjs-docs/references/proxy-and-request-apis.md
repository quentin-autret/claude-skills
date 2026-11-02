# Proxy and request APIs

`proxy.ts` (formerly middleware), `cookies()`, `headers()`, `NextRequest`, `NextResponse`, and the `headers`, `redirects` and `rewrites` keys of `next.config` in Next.js 16.4.4.

## Contents

- `proxy.ts` basics
- Matchers
- Passing data and headers
- Proxy is not an auth boundary
- `cookies()` and `headers()`
- `redirects` in `next.config`
- `rewrites` in `next.config`
- `headers` in `next.config`

## `proxy.ts` basics

```ts
// proxy.ts
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'
export function proxy(request: NextRequest) {
  return NextResponse.redirect(new URL('/home', request.url))
}
export const config = { matcher: '/about/:path*' }
```
https://nextjs.org/docs/app/api-reference/file-conventions/proxy

- One file per project, at the root or in `src/` next to `app`. Export `proxy` (named or default). Split logic into modules.
  https://nextjs.org/docs/app/api-reference/file-conventions/proxy
- Proxy runs on Node.js. Setting `runtime` in the file throws.
  https://nextjs.org/docs/app/api-reference/file-conventions/proxy#runtime
- Simple path maps belong in `redirects` in `next.config`.
  https://nextjs.org/docs/app/getting-started/proxy#use-cases
- Next.js recommends Proxy as a last resort.
  https://nextjs.org/docs/app/api-reference/file-conventions/proxy#why-the-change
- `fetch` cache options (`cache`, `next.revalidate`, `next.tags`) have no effect in Proxy.
  https://nextjs.org/docs/app/getting-started/proxy#use-cases
- Background work after the response: `event.waitUntil(promise)` with the second argument (`NextFetchEvent`). The `NextProxy` type infers both.
  https://nextjs.org/docs/app/api-reference/file-conventions/proxy#params
- Static export does not support Proxy.
  https://nextjs.org/docs/app/api-reference/file-conventions/proxy#platform-support

## Matchers

- Always set a `matcher`. Without one Proxy runs on `_next/static`, `_next/image` and `public/` files, so an auth redirect can block CSS, JS and images.
  ```js
  export const config = {
    matcher: ['/((?!api|_next/static|_next/image|favicon.ico|sitemap.xml|robots.txt).*)'],
  }
  ```
  https://nextjs.org/docs/app/api-reference/file-conventions/proxy#matcher
- Matcher values must be constants: variables are ignored. `:path` matches 1 segment, `:path*` matches nested ones.
  https://nextjs.org/docs/app/api-reference/file-conventions/proxy#matcher
- Skip prefetches with the object form: `missing: [{ type: 'header', key: 'next-router-prefetch' }]`.
  https://nextjs.org/docs/app/api-reference/file-conventions/proxy#negative-matching
- Proxy still runs for `/_next/data/*` even when the matcher excludes it, on purpose.
  https://nextjs.org/docs/app/api-reference/file-conventions/proxy#negative-matching
- Order: `next.config` `headers`, `redirects`, Proxy, `beforeFiles` rewrites, filesystem, `afterFiles`, dynamic routes, `fallback`.
  https://nextjs.org/docs/app/api-reference/file-conventions/proxy#execution-order
- Exclude metadata files (for example `favicon.ico`, `sitemap.xml`, `robots.txt`) from the matcher.
  https://nextjs.org/docs/app/api-reference/file-conventions/proxy#negative-matching

## Passing data and headers

- Forward headers upstream with `NextResponse.next({ request: { headers } })`. `NextResponse.next({ headers })` sends them to the client instead and can override `Content-Type`, breaking Server Actions and streaming.
  ```ts
  const newHeaders = new Headers(request.headers)
  newHeaders.set('x-version', '123')
  return NextResponse.next({ request: { headers: newHeaders } })
  ```
  https://nextjs.org/docs/app/api-reference/functions/next-response#next
- Do not copy every incoming header upstream. Allow-list them and drop `authorization` and `cookie`.
  https://nextjs.org/docs/app/api-reference/functions/next-response#next
- Pass data to the app through headers, cookies, rewrites or the URL, never through shared modules or globals.
  https://nextjs.org/docs/app/api-reference/file-conventions/proxy
- Large request headers set in Proxy can trigger 431.
  https://nextjs.org/docs/app/api-reference/file-conventions/proxy#setting-headers
- Set outgoing cookies through `response.cookies`.
  https://nextjs.org/docs/app/api-reference/file-conventions/proxy#using-cookies
- Next.js strips Flight headers (`rsc`, `next-router-state-tree`) from `request.headers`. `NextResponse.rewrite()` propagates the required RSC headers upstream. A custom `fetch()` rewrite can miss them unless you forward them manually.
  https://nextjs.org/docs/app/api-reference/file-conventions/proxy#rsc-requests-and-rewrites
- With Proxy, request bodies are buffered up to 10 MB (`experimental.proxyClientMaxBodySize`). Beyond that only the first 10 MB is buffered, and a warning is logged naming the route.
  https://nextjs.org/docs/app/api-reference/config/next-config-js/proxyClientMaxBodySize
- `NextRequest.ip` and `.geo` were removed in v15.0.0.
  https://nextjs.org/docs/app/api-reference/functions/next-request#version-history

## Proxy is not an auth boundary

- Server Functions (Server Actions) are POST requests to the route where they are used. A matcher that excludes a path skips the calls there, and moving one to another route can silently remove coverage. Verify auth inside each Server Function.
  https://nextjs.org/docs/app/api-reference/file-conventions/proxy#execution-order
- CORS for `/api/:path*`: handle the `OPTIONS` preflight and echo the origin only when it is allow-listed.
  https://nextjs.org/docs/app/api-reference/file-conventions/proxy#cors

## `cookies()` and `headers()`

- Both are async: `const cookieStore = await cookies()`.
  https://nextjs.org/docs/app/api-reference/functions/cookies
- Writing cookies (`set`, `delete`) only works in Server Functions and Route Handlers: HTTP does not allow setting cookies after streaming starts.
  https://nextjs.org/docs/app/api-reference/functions/cookies#good-to-know
- `headers()` is read-only: it cannot set or delete headers.
  https://nextjs.org/docs/app/api-reference/functions/headers#good-to-know
- With Cache Components, call both inside a `<Suspense>` boundary, or the route cannot prerender.
  https://nextjs.org/docs/app/api-reference/functions/cookies#good-to-know
- `delete` only removes a cookie set with the same domain and protocol.
  https://nextjs.org/docs/app/api-reference/functions/cookies#good-to-know
- `userAgent(request)` from `next/server` gives `isBot`, `browser`, `device` and `os`. `device.type` is `undefined` on desktop.
  https://nextjs.org/docs/app/api-reference/functions/userAgent

## `redirects` in `next.config`

- `permanent: true` is 308 (cached forever by browsers), `false` is 307. Both preserve the method. The query string passes through.
  https://nextjs.org/docs/app/api-reference/config/next-config-js/redirects
- Keep the `/` before `:param` in `source` and `destination`, otherwise it is literal and can loop.
  https://nextjs.org/docs/app/api-reference/config/next-config-js/redirects
- Escape `( ) { } : * + ?` in `source` with `\\`.
  https://nextjs.org/docs/app/api-reference/config/next-config-js/redirects#regex-path-matching
- `has` and `missing` match `header`, `cookie`, `host` or `query`. Use a named capture group to reuse a value in `destination`.
  https://nextjs.org/docs/app/api-reference/config/next-config-js/redirects#header-cookie-and-query-matching

## `rewrites` in `next.config`

- A plain array runs after the filesystem and before dynamic routes. `beforeFiles` can override pages and `public` files. `fallback` proxies to an old site during incremental adoption.
  ```js
  rewrites() { return { beforeFiles: [], afterFiles: [], fallback: [{ source: '/:path*', destination: 'https://old.example.com/:path*' }] } }
  ```
  https://nextjs.org/docs/app/api-reference/config/next-config-js/rewrites
- When `destination` uses any param, no params are added to the query automatically.
  https://nextjs.org/docs/app/api-reference/config/next-config-js/rewrites#rewrite-parameters
- Rewrites apply to client navigation too: the URL stays, the content changes.
  https://nextjs.org/docs/app/api-reference/config/next-config-js/rewrites

## `headers` in `next.config`

- When 2 entries set the same key, the last wins. `Cache-Control` on hashed assets cannot be overridden.
  https://nextjs.org/docs/app/api-reference/config/next-config-js/headers#header-overriding-behavior
  (`Cache-Control` note: https://nextjs.org/docs/app/api-reference/config/next-config-js/headers#cache-control)
- Prefer CSP `frame-ancestors` over `X-Frame-Options`.
  https://nextjs.org/docs/app/api-reference/config/next-config-js/headers#x-frame-options
- With i18n, `source: '/(.*)'` misses `/`.
  https://nextjs.org/docs/app/api-reference/config/next-config-js/headers#headers-with-i18n-support
