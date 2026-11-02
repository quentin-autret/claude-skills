# Deployment and self-hosting

`next start`, Docker and `output: 'standalone'`, multi-instance caching, Server Action keys, version skew, CDN headers, static export, custom servers, multi-zones and build adapters in Next.js 16.4.4.

## Contents

- What a platform needs
- Standalone and Docker
- Multi-instance deployments
- Version skew and build IDs
- Streaming behind proxies
- Caching headers
- Static export
- Custom server
- Multi-zones
- Build adapters

## What a platform needs

- A single `next start` Node.js process supports every feature: Server Components, ISR, PPR, Cache Components, Server Actions, Proxy and `after()`. Image optimization also needs `sharp`.
  https://nextjs.org/docs/app/guides/deploying-to-platforms
- Put a reverse proxy such as nginx in front for malformed requests, slow clients, payload limits and rate limiting.
  https://nextjs.org/docs/app/guides/self-hosting#reverse-proxy
- Measure with `next build` then `next start` locally before production.
  https://nextjs.org/docs/app/guides/production-checklist#before-going-to-production
- Graceful shutdown: send `SIGINT` or `SIGTERM` and allow 10 to 30 s. In-flight requests and pending `after()` callbacks finish.
  https://nextjs.org/docs/app/guides/self-hosting#after

## Standalone and Docker

- Use `output: 'standalone'` for a minimal image. Copy `public` and `.next/static` yourself unless a CDN serves them:
  ```bash
  cp -r public .next/standalone/ && cp -r .next/static .next/standalone/.next/
  ```
  https://nextjs.org/docs/app/api-reference/config/next-config-js/output#automatically-copying-traced-files
- Start with `node .next/standalone/server.js`, configured by `PORT` and `HOSTNAME`.
  https://nextjs.org/docs/app/api-reference/config/next-config-js/output#automatically-copying-traced-files
- In a monorepo, set `outputFileTracingRoot: path.join(__dirname, '../../')`. Fix tracing misses with `outputFileTracingIncludes`, keeping globs narrow.
  ```js
  outputFileTracingIncludes: { '/*': ['node_modules/sharp/**/*'] }
  ```
  https://nextjs.org/docs/app/api-reference/config/next-config-js/output#caveats
- For one image promoted across environments, read env vars on the server after `await connection()`. `NEXT_PUBLIC_*` values are frozen at build.
  ```js
  import { connection } from 'next/server'
  export default async function Component() {
    await connection()
    const value = process.env.MY_VALUE
  }
  ```
  https://nextjs.org/docs/app/guides/self-hosting#environment-variables

## Multi-instance deployments

- The default cache lives in memory (50 MB) and on the local disk of each instance. Every Kubernetes pod has its own copy.
  https://nextjs.org/docs/app/guides/self-hosting#configuring-caching
- `cacheHandler` (singular) covers ISR, Route Handler responses and optimized images (images need `images.customCacheHandler: true`). It is not used by `use cache`, which goes through `cacheHandlers` (plural).
  https://nextjs.org/docs/app/api-reference/config/next-config-js/incrementalCacheHandlerPath
- Shared ISR cache: `cacheHandler: require.resolve('./cache-handler.js')` with `cacheMaxMemorySize: 0`. Store in Redis or S3, never an in-process `Map`.
  https://nextjs.org/docs/app/guides/self-hosting#configuring-caching
- Serialize entries properly: they hold a `Buffer` (`rscData`) and a `Map` (`segmentData`). Otherwise errors like `segmentData.get is not a function` appear.
  https://nextjs.org/docs/app/guides/self-hosting#configuring-caching
- Treat cache keys as opaque: preserve the whole key, it is not a request pathname.
  https://nextjs.org/docs/app/api-reference/config/next-config-js/incrementalCacheHandlerPath#get
- `revalidateTag()` only invalidates the local instance. Implement `refreshTags()` in the `cacheHandlers` handler to sync tags from shared storage.
  https://nextjs.org/docs/app/guides/self-hosting#multi-instance-cache-coordination
- Each build generates its own Server Actions key. Set one `NEXT_SERVER_ACTIONS_ENCRYPTION_KEY` (base64, 16, 24 or 32 bytes) at `next build`, or users get "Failed to find Server Action".
  https://nextjs.org/docs/app/guides/self-hosting#server-functions-encryption-key
- Use `'use cache: remote'` with a custom `cacheHandlers` entry for a cache shared across instances.
  https://nextjs.org/docs/app/guides/self-hosting#shared-cache

## Version skew and build IDs

- Boot every container from the same build. If you rebuild per stage, pin `generateBuildId: async () => process.env.GIT_HASH`.
  https://nextjs.org/docs/app/guides/self-hosting#build-cache
- `deploymentId` (or `NEXT_DEPLOYMENT_ID`) enables skew protection. A mismatch triggers a hard navigation that loses client state.
  https://nextjs.org/docs/app/api-reference/config/next-config-js/deploymentId#how-it-works
- Next.js does not route by `?dpl=`. Your host or CDN must send clients to the matching deployment.
  https://nextjs.org/docs/app/api-reference/config/next-config-js/deploymentId#multi-server-environments

## Streaming behind proxies

- Behind nginx, send `X-Accel-Buffering: no` through `headers()` with source `'/:path*{/}?'`, to disable nginx buffering, which streaming needs. Load balancers and reverse proxies must also pass chunked responses without buffering.
  https://nextjs.org/docs/app/guides/self-hosting#streaming-and-suspense
- Set `compress: false` only when the proxy compresses.
  https://nextjs.org/docs/app/api-reference/config/next-config-js/compress

## Caching headers

- Immutable, hash-named assets get `public, max-age=31536000, immutable`, which cannot be overridden.
  https://nextjs.org/docs/app/guides/self-hosting#automatic-caching
- ISR sends `s-maxage=<revalidate>, stale-while-revalidate`. Dynamic pages and Draft Mode send `private, no-cache, no-store, max-age=0, must-revalidate`.
  https://nextjs.org/docs/app/guides/self-hosting#automatic-caching
- `expireTime` sets the stale window: revalidate 900 with expireTime 3600 gives `s-maxage=900, stale-while-revalidate=2700`.
  https://nextjs.org/docs/app/api-reference/config/next-config-js/expireTime
- `outputHashSalt` changes every output hash to bust asset caches without source changes.
  https://nextjs.org/docs/app/api-reference/config/next-config-js/outputHashSalt

## Static export

- `output: 'export'` writes to `out/`. Unsupported: dynamic routes without `generateStaticParams`, Proxy, rewrites, redirects, headers, cookies, ISR, Server Actions, Draft Mode, the default image loader and intercepting routes.
  https://nextjs.org/docs/app/guides/static-exports#unsupported-features
- Images need a custom `loaderFile`. Route Handlers are `GET` only with `export const dynamic = 'force-static'`.
  https://nextjs.org/docs/app/guides/static-exports#route-handlers
- Your host handles rewrites. With nginx: `try_files $uri $uri.html $uri/ =404;`.
  https://nextjs.org/docs/app/guides/static-exports#deploying

## Custom server

- Only when the built-in router cannot do the job. It cannot combine with standalone output, and `server.js` is not compiled.
  https://nextjs.org/docs/app/guides/custom-server
- Set headers such as `Set-Cookie` before `await handle(req, res)`. Afterwards `setHeader` is silently ignored.
  https://nextjs.org/docs/app/guides/custom-server

## Multi-zones

- A zone is a normal app with an `assetPrefix` (the default zone needs none). Route with `rewrites`, keeping paths unique per zone.
  https://nextjs.org/docs/app/guides/multi-zones#how-to-route-requests-to-the-right-zone
- Link across zones with `<a>`, not `<Link>`.
  https://nextjs.org/docs/app/guides/multi-zones#linking-between-zones
- Server Actions across zones need the user-facing origin in `experimental.serverActions.allowedOrigins`.
  https://nextjs.org/docs/app/guides/multi-zones#server-actions

## Build adapters

- Adapters let platforms customize how apps are built and deployed. Enable with top-level `adapterPath: require.resolve('./my-adapter.js')`.
  https://nextjs.org/docs/app/api-reference/adapters/configuration
- An adapter is build-time only. Runtime caching stays in `cacheHandler` and `cacheHandlers`.
  https://nextjs.org/docs/app/api-reference/adapters/runtime-integration
- `modifyConfig` is called for any CLI command that loads the config. `context.phase` gives the build phase, and it returns the modified config.
  https://nextjs.org/docs/app/api-reference/adapters/api-reference#async-modifyconfigconfig-context
- Reproduce Next.js route matching with `resolveRoutes()` from `@next/routing`, using the data from `onBuildComplete`.
  https://nextjs.org/docs/app/api-reference/adapters/routing-with-next-routing
- `outputs.middleware` covers `proxy.ts` too, with `pathname` always `/_middleware`.
  https://nextjs.org/docs/app/api-reference/adapters/output-types#middleware-outputsmiddleware
- Use `requestMeta.onCacheEntryV2`. `onCacheEntry` is deprecated.
  https://nextjs.org/docs/app/api-reference/adapters/implementing-ppr-in-an-adapter
- Set `supportsImmutableAssets: false` only to opt out of immutable static assets when troubleshooting adapter-specific issues.
  https://nextjs.org/docs/app/api-reference/config/next-config-js/supportsImmutableAssets
