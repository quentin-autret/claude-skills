# Observability, PWA and offline support

`instrumentation.ts`, `onRequestError`, `instrumentation-client.ts`, OpenTelemetry, Web Vitals, source maps, PWA manifests and push, offline support with `useOffline`, and built-in accessibility in Next.js 16.4.4.

## Contents

- `instrumentation.ts`
- `onRequestError`
- `instrumentation-client.ts`
- OpenTelemetry
- Web Vitals
- Source maps
- PWA
- Offline support
- Accessibility

## `instrumentation.ts`

- Place it at the project root or in `src/`, never inside `app/`. If `pageExtensions` adds a suffix, rename the file to match.
  https://nextjs.org/docs/app/guides/instrumentation
- `register()` runs once per server instance and completes before requests are handled. It may be async.
  https://nextjs.org/docs/app/api-reference/file-conventions/instrumentation#register-optional
- `register` runs in every runtime. Import side-effect packages inside it and guard by runtime:
  ```ts
  export async function register() {
    if (process.env.NEXT_RUNTIME === 'nodejs') {
      await import('./instrumentation-node')
    }
  }
  ```
  https://nextjs.org/docs/app/guides/instrumentation#importing-runtime-specific-code

## `onRequestError`

- Export `onRequestError(err, request, context)` to report server errors. Type it with `Instrumentation.onRequestError` from `next`.
  https://nextjs.org/docs/app/api-reference/file-conventions/instrumentation#onrequesterror-optional
- `err` is `unknown` and may not be the original instance. Narrow it and use `err.digest`. Await async work inside the hook.
  https://nextjs.org/docs/app/api-reference/file-conventions/instrumentation#onrequesterror-optional
- `context.routeType` is `'render'`, `'route'`, `'action'` or `'proxy'`. `context.renderType` is `'dynamic-resume'` for PPR resumes.
  https://nextjs.org/docs/app/api-reference/file-conventions/instrumentation#parameters

## `instrumentation-client.ts`

- Runs after the HTML loads and before hydration. Write code at top level, no export required.
  https://nextjs.org/docs/app/api-reference/file-conventions/instrumentation-client#execution-timing
- Only synchronous top-level code is guaranteed to finish before hydration. A promise or `import()` is fire-and-forget.
  https://nextjs.org/docs/app/api-reference/file-conventions/instrumentation-client#execution-timing
- Keep it under 16 ms (dev warns). Wrap each tracker in try/catch.
  https://nextjs.org/docs/app/api-reference/file-conventions/instrumentation-client#performance-considerations
- Export `onRouterTransitionStart(url, navigationType)` to track navigations (`'push'`, `'replace'`, `'traverse'`).
  https://nextjs.org/docs/app/api-reference/file-conventions/instrumentation-client#router-navigation-tracking
- Polyfills: import statically and apply synchronously after feature detection.
  ```ts
  import ResizeObserverPolyfill from './lib/polyfills/resize-observer'
  if (!window.ResizeObserver) { window.ResizeObserver = ResizeObserverPolyfill }
  ```
  https://nextjs.org/docs/app/api-reference/file-conventions/instrumentation-client#polyfills
- `instrumentationClientInject` in `next.config` is for config plugins. A plugin spreads the existing array: `[...(nextConfig.instrumentationClientInject ?? []), 'pkg/client']`.
  https://nextjs.org/docs/app/api-reference/config/next-config-js/instrumentationClientInject#execution-order

## OpenTelemetry

- Simplest setup: `@vercel/otel` with `registerOTel({ serviceName: 'next-app' })` inside `register()`. Unlike `NodeSDK`, it works on the Edge runtime.
  https://nextjs.org/docs/app/guides/open-telemetry#using-vercelotel
- A manual `NodeSDK` setup is not Edge-compatible: import it only when `NEXT_RUNTIME === 'nodejs'`. Use `resourceFromAttributes` and `spanProcessor`.
  https://nextjs.org/docs/app/guides/open-telemetry#manual-opentelemetry-configuration
- Spans started before the tracer provider is installed (for example at the top level of the instrumentation file) are not recorded. Spans started in `register` after the SDK starts typically become separate root traces. Create spans in request code and end them in `finally`.
  https://nextjs.org/docs/app/guides/open-telemetry#custom-spans
- `NEXT_OTEL_VERBOSE=1` emits more spans than the default.
  https://nextjs.org/docs/app/guides/open-telemetry#testing-your-instrumentation
- `NEXT_OTEL_FETCH_DISABLED=1` turns off the built-in fetch span.
  https://nextjs.org/docs/app/guides/open-telemetry#default-spans-in-nextjs
- Self-hosting needs your own collector or exporter.
  https://nextjs.org/docs/app/guides/open-telemetry#self-hosting

## Web Vitals

- `useReportWebVitals` from `next/web-vitals` needs a Client Component. Put it in a small component imported by the root layout.
  https://nextjs.org/docs/app/api-reference/functions/use-report-web-vitals
- Keep the callback reference stable (a module-level function). A new reference replays the metrics and creates duplicates.
  https://nextjs.org/docs/app/api-reference/functions/use-report-web-vitals
- Send with `navigator.sendBeacon`, falling back to `fetch(url, { method: 'POST', keepalive: true })`.
  https://nextjs.org/docs/app/api-reference/functions/use-report-web-vitals#sending-results-to-external-systems
- Google Analytics needs integers: `Math.round(name === 'CLS' ? value * 1000 : value)`.
  https://nextjs.org/docs/app/api-reference/functions/use-report-web-vitals#sending-results-to-external-systems

## Source maps

- Browser source maps are off in production so source is not leaked to the client. `productionBrowserSourceMaps: true` opts in.
  https://nextjs.org/docs/app/api-reference/config/next-config-js/productionBrowserSourceMaps
- `enablePrerenderSourceMaps` (default `true`) uses existing maps in prerender errors. `false` can reduce build memory but makes prerender errors harder to debug. It does not generate maps itself (use `next build --debug-prerender`).
  https://nextjs.org/docs/app/api-reference/config/next-config-js/enablePrerenderSourceMaps

## PWA

- Write `app/manifest.ts` returning `MetadataRoute.Manifest` (the example lists 192 and 512 px icons). Installability needs only the manifest and HTTPS.
  https://nextjs.org/docs/app/guides/progressive-web-apps#1-creating-the-web-app-manifest
- A custom install button on `beforeinstallprompt` is not recommended: it is not cross-browser and does not work on iOS Safari. Show manual instructions there.
  https://nextjs.org/docs/app/guides/progressive-web-apps#6-adding-to-home-screen
- Register the worker with `scope: '/'` and `updateViaCache: 'none'`.
  https://nextjs.org/docs/app/guides/progressive-web-apps#2-implementing-web-push-notifications
- Serve `/sw.js` with `Cache-Control: no-cache, no-store, must-revalidate`.
  https://nextjs.org/docs/app/guides/progressive-web-apps#8-securing-your-application
- In production store push subscriptions in a database. The VAPID private key goes in `VAPID_PRIVATE_KEY`, without the `NEXT_PUBLIC_` prefix of the public key.
  https://nextjs.org/docs/app/guides/progressive-web-apps#3-implementing-server-actions
- Test push locally with `next dev --experimental-https`.
  https://nextjs.org/docs/app/guides/progressive-web-apps#7-testing-locally
- Web push on iOS needs iOS 16.4+ and an app installed to the home screen.
  https://nextjs.org/docs/app/guides/progressive-web-apps#2-implementing-web-push-notifications

## Offline support

- `experimental.useOffline: true` makes failed soft navigations, prefetches and Server Actions stay pending and retry when the network returns, instead of rejecting.
  https://nextjs.org/docs/app/guides/offline-support
- The guide pairs it with Cache Components and Partial Prefetching so prefetched App Shells render offline. Without Cache Components, a route-level `loading.tsx` gives the same behavior.
  https://nextjs.org/docs/app/guides/offline-support#enable-offline-detection-and-see-the-default-behavior
- `useOffline()` from `next/offline` returns `false` on the server, before hydration and without the flag. It is more reliable than `navigator.onLine`.
  https://nextjs.org/docs/app/guides/offline-support#report-connectivity-inside-the-suspense-fallback
- A full reload offline still fails (that needs a service worker). Direct `fetch()`, SWR and React Query keep their own retry rules.
  https://nextjs.org/docs/app/guides/offline-support
- Test with `next build && next start` (dev mode is not reliable) and the devtools Offline toggle.
  https://nextjs.org/docs/app/guides/offline-support#testing

## Accessibility

- The route announcer reads `document.title`, then the `<h1>`, then the pathname. Give every page a unique title.
  https://nextjs.org/docs/architecture/accessibility#route-announcements
