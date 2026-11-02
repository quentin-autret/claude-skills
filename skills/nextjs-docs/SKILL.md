---
name: nextjs-docs
description: Pitfalls, surprising defaults and recommended patterns of the Next.js 16 App Router, distilled from the official docs. Use when writing, reviewing, debugging, configuring or upgrading a Next.js app (app/ directory, next.config.ts), including Server Components, "use client", Cache Components ("use cache", cacheLife, cacheTag), revalidateTag, updateTag, Server Actions, async params, cookies(), proxy.ts (formerly middleware.ts), Suspense, Link prefetch, generateMetadata, next/image, Turbopack, ESLint, typegen, self-hosting and upgrading to Next.js 16.
metadata:
  version: "16.4.4"
  source: https://nextjs.org/docs
  fetched: "2026-11-02"
---

# Next.js App Router from the official docs

## Overview

This skill holds what the Next.js docs say that is easy to get wrong: the Cache Components model, renamed APIs, async request APIs, Server Action security, Proxy limits and version 16 defaults. Each claim carries the docs URL it comes from.

## Version covered

Next.js 16.4.4, App Router only. For an older install, check "Old patterns" first: `next upgrade` came in 16.1, `io()` and `next/root-params` in 16.3, `ensureStatic`, `prefetch()` and `navigation()` in 16.4.

## How to navigate

1. Read the quick start below.
2. Check whether `next.config` sets `cacheComponents: true`. Caching rules differ with and without it.
3. Open the one reference file matching the task (list at the end).

## Quick start

### Request APIs and params

- `params`, `searchParams`, `cookies()`, `headers()` and `draftMode()` are async. Sync access is removed in version 16.
  https://nextjs.org/docs/app/guides/upgrading/version-16#async-request-apis-breaking-change
- `cookies().set()` works only in Server Functions and Route Handlers.
  https://nextjs.org/docs/app/api-reference/functions/cookies#understanding-cookie-behavior-in-server-components
- Layouts never receive `searchParams`. Read it in the page.
  https://nextjs.org/docs/app/api-reference/functions/use-search-params#layouts

### Caching with Cache Components

- Enable with `cacheComponents: true` and `partialPrefetching: true`. Leaving `partialPrefetching` unset logs a warning.
  https://nextjs.org/docs/app/getting-started/caching#enabling-cache-components
- Never read `cookies()` or `headers()` inside `'use cache'`. Read them outside and pass the value as an argument.
  https://nextjs.org/docs/app/api-reference/directives/use-cache#request-time-apis
- Blocking work outside `<Suspense>` fails `next build`. Stream it, cache it, or set `export const instant = false`.
  https://nextjs.org/docs/app/guides/building#find-blocking-work
- `Date.now()`, `Math.random()` and `crypto.randomUUID()`: wrap them in `use cache` to freeze them in the static shell, or `await io()` inside Suspense to keep them out.
  https://nextjs.org/docs/app/api-reference/functions/io
- `'use cache'` is in-memory and scoped to one deployment. For serverless, use `'use cache: remote'` or a cache handler.
  https://nextjs.org/docs/app/guides/migrating-to-cache-components#fetch-cache-options

### Invalidation

- In a Server Action after a mutation: `updateTag('posts')` (read-your-own-writes, Server Actions only).
  https://nextjs.org/docs/app/getting-started/revalidating#updatetag
- In a webhook or Route Handler: `revalidateTag('posts', 'max')`. The one-argument form is deprecated.
  https://nextjs.org/docs/app/api-reference/functions/revalidateTag#revalidation-behavior
- Without Cache Components, `fetch` is not cached unless `cache: 'force-cache'` is set.
  https://nextjs.org/docs/app/api-reference/functions/fetch#optionscache

### Server Actions

- Authenticate and authorize inside each action. Rendering a form only on an authenticated page is not a security boundary.
  https://nextjs.org/docs/app/guides/server-actions#security
- Return only the data the UI needs, not raw database records.
  https://nextjs.org/docs/app/api-reference/directives/use-server#return-values
- Return expected errors instead of throwing them.
  https://nextjs.org/docs/app/getting-started/error-handling#handling-expected-errors
- Several instances need the same `NEXT_SERVER_ACTIONS_ENCRYPTION_KEY`, or users see "Failed to find Server Action".
  https://nextjs.org/docs/app/guides/self-hosting#server-functions-encryption-key
- Behind a proxy with another host, set `serverActions.allowedOrigins`.
  https://nextjs.org/docs/app/api-reference/config/next-config-js/serverActions#allowedorigins

### Proxy

- `middleware.ts` is now `proxy.ts` and runs on Node.js. The `runtime` option is not available there.
  https://nextjs.org/docs/app/api-reference/file-conventions/proxy#runtime
- Without a `matcher`, Proxy runs on every request, including `_next/static`, `_next/image` and `public/`.
  https://nextjs.org/docs/app/api-reference/file-conventions/proxy#matcher
- Forward request headers with `NextResponse.next({ request: { headers } })`. `NextResponse.next({ headers })` is not good practice.
  https://nextjs.org/docs/app/api-reference/functions/next-response#next
- Keep auth in Proxy to an optimistic cookie check, with no database call, since Proxy also runs on prefetches. Do the secure session check in the data layer and in each Server Action.
  https://nextjs.org/docs/app/guides/authentication#optimistic-checks-with-proxy-optional

### Components and routing

- Pass Server Components as `children` to Client Components instead of importing them.
  https://nextjs.org/docs/app/guides/server-and-client-boundary#crossing-the-boundary
- After streaming starts, `not-found` keeps status 200 (404 before). `forbidden` is the same (403 before).
  https://nextjs.org/docs/app/api-reference/file-conventions/not-found
  https://nextjs.org/docs/app/api-reference/functions/forbidden
- Every parallel route slot needs a `default.js`, or the build fails.
  https://nextjs.org/docs/app/guides/upgrading/version-16#parallel-routes-defaultjs-requirement
- Error boundaries receive `{ error, retry }`, with `error` typed `unknown`. Prefer `retry()` over `reset()`.
  https://nextjs.org/docs/app/api-reference/file-conventions/error#retry
- Prefetching is only enabled in production.
  https://nextjs.org/docs/app/api-reference/components/link#prefetch

### Metadata, images, build

- Do not export both `metadata` and `generateMetadata` from one segment. Metadata is shallowly merged, so a child `openGraph` replaces the parent's.
  https://nextjs.org/docs/app/api-reference/functions/generate-metadata#merging
- `next/image`: `qualities` is required from version 16, `priority` is deprecated for `preload`, `images.domains` is deprecated.
  https://nextjs.org/docs/app/api-reference/components/image#qualities
- Turbopack is the default. `next build` fails with a custom `webpack` config unless you pass `--webpack`.
  https://nextjs.org/docs/app/guides/upgrading/version-16#turbopack-by-default
- `next build` no longer lints. Use the ESLint CLI.
  https://nextjs.org/docs/app/guides/upgrading/version-16#next-lint-command
- Type-check with `next typegen && tsc --noEmit`.
  https://nextjs.org/docs/app/api-reference/cli/next#next-typegen-options
- Debug prerender errors with `next build --debug-prerender`. Never deploy that build.
  https://nextjs.org/docs/app/guides/building#debug-prerender-errors

## Old patterns

Rows come from https://nextjs.org/docs/app/guides/upgrading/version-16 unless a URL is given.

| Old | Current |
| --- | --- |
| `middleware.ts`, `skipMiddlewareUrlNormalize` | `proxy.ts`, `skipProxyUrlNormalize` |
| `revalidateTag('posts')` | `revalidateTag('posts', 'max')`, or `updateTag` in a Server Action |
| `unstable_cacheLife`, `unstable_cacheTag` | `cacheLife`, `cacheTag` |
| `experimental.ppr`, `experimental.dynamicIO`, `experimental.useCache` | `cacheComponents: true` |
| `unstable_cache` | `'use cache'` (old form still works): https://nextjs.org/docs/app/guides/migrating-to-cache-components#unstable_cache |
| `unstable_noStore()`, `dynamic = 'force-dynamic'` | remove under Cache Components: https://nextjs.org/docs/app/guides/migrating-to-cache-components |
| `dynamicParams` export with Cache Components | delete it, use `notFound()`: https://nextjs.org/docs/app/guides/migrating-to-cache-components#generatestaticparams-and-dynamicparams |
| `unstable_instant`, `unstable_prefetch` | `instant`, `prefetch` (16.3): https://nextjs.org/docs/app/api-reference/file-conventions/route-segment-config/prefetch#version-history |
| `unstable_catchError`, `unstable_retry` | `catchError`, `retry` (16.3): https://nextjs.org/docs/app/api-reference/file-conventions/error#version-history |
| `unstable_rootParams` | `next/root-params` |
| `serverRuntimeConfig`, `publicRuntimeConfig` | env vars, read after `await connection()` |
| `next lint` | ESLint CLI |
| `experimental.turbopack` | top-level `turbopack` |
| `experimental.reactCompiler` | `reactCompiler`: https://nextjs.org/docs/app/api-reference/config/next-config-js/reactCompiler#version-history |
| `experimental.typedRoutes` | `typedRoutes`: https://nextjs.org/docs/app/api-reference/config/next-config-js/typedRoutes |
| `<Image priority>`, `onLoadingComplete` | `preload`, `onLoad`: https://nextjs.org/docs/app/api-reference/components/image |
| `images.domains`, `next/legacy/image` | `remotePatterns`, `next/image` |
| `export const runtime = 'edge'` with Cache Components | remove it, use Proxy for edge behavior: https://nextjs.org/docs/app/guides/migrating-to-cache-components#runtime--edge |
| Sync `params`, `searchParams`, `cookies()`, `headers()`, `draftMode()` | `await` them (sync access removed in 16): https://nextjs.org/docs/app/guides/upgrading/version-16#async-request-apis-breaking-change |
| `experimental.adapterPath` | top-level `adapterPath` (16.2.0): https://nextjs.org/docs/app/guides/upgrading/version-16#build-adapters-api-alpha |
| `useFormState` | `useActionState`: https://nextjs.org/docs/app/guides/upgrading/version-15#react-19 |
| `experimental.serverComponentsExternalPackages` | `serverExternalPackages`: https://nextjs.org/docs/app/guides/upgrading/version-15#bundlepagesrouterdependencies |
| `NextRequest.geo`, `NextRequest.ip` | host-provided header or `geolocation`/`ipAddress` from `@vercel/functions`: https://nextjs.org/docs/app/guides/upgrading/version-15#nextrequest-geolocation |
| AMP (`amp` config, `next/amp`, `config = { amp: true }`) | removed in 16: https://nextjs.org/docs/app/guides/upgrading/version-16#amp-support |
| `ImageResponse` from `next/server` | `next/og` (14.0.0): https://nextjs.org/docs/app/api-reference/functions/image-response#version-history |
| `useRouter` from `next/router` in the App Router | `next/navigation`: https://nextjs.org/docs/app/api-reference/functions/use-router#migrating-from-nextrouter |
| `metadata.viewport`, `metadata.themeColor`, `metadata.colorScheme` | `export const viewport` or `generateViewport` (14.0.0): https://nextjs.org/docs/app/api-reference/functions/generate-metadata#version-history |
| `eslint` key in next.config | removed in 16, use the ESLint CLI: https://nextjs.org/docs/app/guides/upgrading/version-16#next-lint-command |

## Reference files

Open the one file matching the task. Sources and fetch details are in [SOURCES.md](SOURCES.md).

| File | Open when |
| --- | --- |
| [caching-cache-components.md](references/caching-cache-components.md) | `use cache`, `cacheLife`, `updateTag`, the static shell, cache handlers |
| [caching-without-cache-components.md](references/caching-without-cache-components.md) | `fetch` defaults, `unstable_cache`, ISR, migrating to Cache Components |
| [server-and-client-components.md](references/server-and-client-components.md) | placing `"use client"`, crossing the boundary, error handling, optimistic UI |
| [file-conventions-and-routing.md](references/file-conventions-and-routing.md) | layouts, `not-found`, parallel routes, Route Handlers, `generateStaticParams`, segment config |
| [data-fetching-and-server-actions.md](references/data-fetching-and-server-actions.md) | reading data, Server Actions, their security and deployment pitfalls, `<Form>`, `after()` |
| [auth-and-data-security.md](references/auth-and-data-security.md) | Data Access Layer, sessions, env variables, CSP, Draft Mode |
| [navigation-and-redirects.md](references/navigation-and-redirects.md) | `useRouter`, `useSearchParams`, `redirect`, `catchError`, `navigation()` |
| [linking-prefetching-and-instant-navigation.md](references/linking-prefetching-and-instant-navigation.md) | `<Link>`, prefetching, `instant`, view transitions |
| [proxy-and-request-apis.md](references/proxy-and-request-apis.md) | `proxy.ts`, matchers, `cookies()`, `headers()`, redirects, rewrites |
| [metadata-and-seo.md](references/metadata-and-seo.md) | `generateMetadata`, Open Graph images, sitemaps, viewport, JSON-LD |
| [images-fonts-scripts.md](references/images-fonts-scripts.md) | `next/image`, fonts, `next/script`, third-party libraries |
| [turbopack-css-and-bundling.md](references/turbopack-css-and-bundling.md) | Turbopack config and limits, CSS, package bundling, MDX |
| [cli-config-and-testing.md](references/cli-config-and-testing.md) | `next dev`, `next build`, `next.config`, TypeScript, ESLint, testing |
| [deployment-and-self-hosting.md](references/deployment-and-self-hosting.md) | Docker, standalone, multi-instance, static export, build adapters |
| [observability-pwa-and-offline.md](references/observability-pwa-and-offline.md) | `instrumentation.ts`, OpenTelemetry, Web Vitals, PWA, offline |
| [upgrading-and-migrating.md](references/upgrading-and-migrating.md) | upgrading to 16, Pages Router to App Router, Vite or CRA migration |
