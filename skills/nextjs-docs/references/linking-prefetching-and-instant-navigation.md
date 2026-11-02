# Linking, prefetching and instant navigation

`<Link>`, prefetch modes, Partial Prefetching, the `prefetch()` function, the `instant`, `prefetch` and `ensureStatic` segment config, UI state preservation and view transitions in Next.js 16.4.4.

## Contents

- `<Link>` essentials
- Prefetching without Partial Prefetching
- Partial Prefetching
- `prefetch()` and URL data
- Segment config: `instant`, `prefetch`, `ensureStatic`
- UI state preserved with Activity
- View transitions
- Testing instant navigation

## `<Link>` essentials

- Prefetching only runs in production. Test it on `next build` and `next start`.
  https://nextjs.org/docs/app/api-reference/components/link#prefetch
- `<Link>` must hydrate before it prefetches. Large bundles delay prefetching.
  https://nextjs.org/docs/app/getting-started/linking-and-navigating#hydration-not-completed
- A dynamic route without `loading.tsx` is not fully prefetched, and the click waits for the server.
  https://nextjs.org/docs/app/getting-started/linking-and-navigating#dynamic-routes-without-loadingtsx
- `onNavigate` with `e.preventDefault()` blocks navigation (unsaved form). It runs only for same-origin client navigation, never on Ctrl or Cmd click.
  https://nextjs.org/docs/app/api-reference/components/link#onnavigate
- Sticky header hiding targets: `html { scroll-padding-top: 64px }`.
  https://nextjs.org/docs/app/api-reference/components/link#scroll-offset-with-sticky-headers
- `useLinkStatus` (inside a `<Link>` descendant) is a patch for slow navigations. Start the hint invisible with a 100 ms delay and keep a fixed size.
  https://nextjs.org/docs/app/api-reference/functions/use-link-status
- Proxy rewrites: `<Link as="/dashboard" href="/auth/dashboard">` gives both the shown URL and the one to prefetch.
  https://nextjs.org/docs/app/api-reference/components/link#prefetching-links-in-proxy
- Prefetch can run Server Component code. Move analytics out of layouts into a Client Component `useEffect`.
  https://nextjs.org/docs/app/guides/prefetching#triggering-unwanted-side-effects-during-prefetching

## Prefetching without Partial Prefetching

- Default without Cache Components: a static route prefetches in full, a dynamic route is skipped unless it has a `loading.js`, which then bounds the prefetch.
  https://nextjs.org/docs/app/guides/prefetching#prefetching-static-vs-dynamic-routes
- Large link lists: `prefetch={false}`, or prefetch on hover only:
  ```tsx
  const [active, setActive] = useState(false)
  <Link href={href} prefetch={active ? null : false} onMouseEnter={() => setActive(true)}>
  ```
  https://nextjs.org/docs/app/guides/prefetching#hover-triggered-prefetch
- Prefetch outside the viewport with `router.prefetch('/pricing')`.
  https://nextjs.org/docs/app/guides/prefetching#manual-prefetch

## Partial Prefetching

- Needs `cacheComponents: true` and `partialPrefetching: true`. Without Cache Components the config throws.
  https://nextjs.org/docs/app/api-reference/config/next-config-js/partialPrefetching#usage
- Default `<Link>` prefetches one shared App Shell per route. URL data (`params`, `searchParams`) streams in after navigation.
  https://nextjs.org/docs/app/guides/prefetching#partial-prefetching
- `prefetch={true}` now means App Shell plus cached content that depends on the link's URL. It costs one server invocation per visible link, so opt in per link and never on large card grids.
  https://nextjs.org/docs/app/guides/optimizing-prefetching#trade-offs
- In conditionals return `'auto'` (equivalent to omitting `prefetch` or passing `undefined`) to keep the default App Shell prefetch; `false` disables prefetching: `prefetch={isFeatured ? true : 'auto'}`. Hover pattern: `prefetch={active ? true : 'auto'}`.
  https://nextjs.org/docs/app/guides/optimizing-prefetching#resolve-url-data-at-prefetch-time
- The App Shell keeps cached content only when `stale` is at least 5 minutes (every preset except `seconds`).
  https://nextjs.org/docs/app/guides/adopting-partial-prefetching#uncached-content
- Read `params` and `searchParams` only inside a `<Suspense>` child, even for params listed in `generateStaticParams`.
  https://nextjs.org/docs/app/guides/optimizing-prefetching#resolve-url-data-at-prefetch-time
- Migrating each old `prefetch={true}` link: static or cached, remove the prop. Session content, read `cookies()` outside and pass it to a `use cache` function. URL data, keep `prefetch={true}`.
  https://nextjs.org/docs/app/guides/adopting-partial-prefetching#migrate-existing-full-prefetches
- Adopt per segment with `export const prefetch = 'partial'`, then remove the exports with the `remove-partial-prefetch` codemod.
  https://nextjs.org/docs/app/guides/adopting-partial-prefetching#adopting-incrementally

## `prefetch()` and URL data

- `await prefetch()` from `next/cache` keeps the runtime work below it out of the App Shell but inside per-link prefetches. Unlike `connection()`, the content stays cacheable.
  ```tsx
  async function Recommendations() {
    await prefetch()
    const items = await getRecommendations()
  }
  ```
  https://nextjs.org/docs/app/api-reference/functions/prefetch
- Call it inside a `<Suspense>` boundary (outside, dev shows an Instant Insight). It has no effect without Partial Prefetching and throws inside `use cache`, `after()`, `generateStaticParams` and Client Components. When the file also exports the `prefetch` segment config, import the function under another name.
  https://nextjs.org/docs/app/api-reference/functions/prefetch
- To defer work until the real navigation, use `navigation()` instead.
  https://nextjs.org/docs/app/guides/optimizing-prefetching#defer-work-to-a-later-stage

## Segment config: `instant`, `prefetch`, `ensureStatic`

- `export const instant = false` lets a segment block navigation. Place it as low as possible: on the root layout it disables static shell validation for the whole app. Validation runs in dev only at the default `warning` level.
  https://nextjs.org/docs/app/api-reference/file-conventions/route-segment-config/instant
- By default every page is validated. `experimental.instantInsights.validationLevel: 'manual-warning'` limits it to segments that export `instant`.
  https://nextjs.org/docs/app/api-reference/file-conventions/route-segment-config/instant#configuring-validation-defaults
- `prefetch` config takes `'partial'` or `'force-disabled'`, set on the destination. `<Link prefetch={false}>` always wins.
  https://nextjs.org/docs/app/api-reference/file-conventions/route-segment-config/prefetch
- `ensureStatic` (`'shell'`, `'prefetch'`, `'navigation'`) requires `cacheComponents` and selects which navigation stage output must be static. Only `'navigation'` adds validation in `next dev` and `next build`. A child may be stricter than its parent, never weaker (the build fails).
  https://nextjs.org/docs/app/api-reference/file-conventions/route-segment-config/ensureStatic
- `ensureStatic = 'navigation'` rejects even inside Suspense: uncached data, `cookies()`, `headers()`, short `cacheLife`. A dynamic route must export `generateStaticParams` returning at least 1 complete param set.
  https://nextjs.org/docs/app/api-reference/file-conventions/route-segment-config/ensureStatic#navigation

## UI state preserved with Activity

- With Cache Components, hidden routes keep DOM and React state (`display: none`), up to 3 routes. Effects clean up on hide.
  https://nextjs.org/docs/app/guides/preserving-ui-state
- Dropdowns stay open on return: reset in a `useLayoutEffect` cleanup. Derive dialog state from a search param.
  https://nextjs.org/docs/app/guides/preserving-ui-state
- Log out with `window.location.href` for a full reload that clears client state. Pause `<video>` and `<audio>` in a `useLayoutEffect` cleanup.
  https://nextjs.org/docs/app/guides/preserving-ui-state
- E2E selectors can match hidden DOM: use `getByRole` or a visible filter.
  https://nextjs.org/docs/app/guides/preserving-ui-state

## View transitions

- `<ViewTransition>` animates Transitions, Suspense and `useDeferredValue`, never a plain `setState`. Route navigations are Transitions.
  https://nextjs.org/docs/app/guides/view-transitions
- Put wrappers in `page.tsx`, never in layouts, which persist so enter and exit never fire.
  https://nextjs.org/docs/app/guides/view-transitions
- A shared morph pairs only when the destination renders in the same commit (prefetched). Directional motion needs `transitionTypes` on each `<Link>`. Browser back and forward carry no type.
  https://nextjs.org/docs/app/guides/view-transitions#step-3-add-directional-motion-for-navigation

## Testing instant navigation

- Assert what shows right after navigation with `instant(page, fn)` from `@next/playwright`.
  https://nextjs.org/docs/app/guides/instant-navigation#prevent-regressions-with-e2e-tests
- On `next start`, set `experimental.exposeTestingApiInProductionBuild` (needs `cacheComponents`) at `next build` time, never for a user-facing deployment.
  https://nextjs.org/docs/app/api-reference/config/next-config-js/exposeTestingApiInProductionBuild
