# Navigation hooks, redirects and interrupts

`useRouter`, `useSearchParams`, `usePathname`, segment hooks, `redirect`, `permanentRedirect`, `notFound`, `forbidden`, `unauthorized`, `catchError`, `unstable_rethrow` and `navigation()` in Next.js 16.4.4.

## Contents

- `useRouter`
- `useSearchParams` and `searchParams`
- Pathname and segment hooks
- Choosing a redirect
- `redirect` and the interrupt functions
- `catchError` and `unstable_rethrow`
- `navigation()`

## `useRouter`

- Import from `next/navigation`. Prefer `<Link>` unless you have a specific requirement for programmatic navigation.
  https://nextjs.org/docs/app/api-reference/functions/use-router
- Never pass untrusted URLs to `router.push` or `router.replace`: a `javascript:` URL runs in the page (XSS).
  https://nextjs.org/docs/app/api-reference/functions/use-router
- `router.refresh()` clears the client cache of the current route but does not invalidate the server cache. Use `revalidatePath` or `revalidateTag` for that.
  https://nextjs.org/docs/app/api-reference/functions/use-router
- `router.push(href, { scroll: false })` keeps the scroll position. `transitionTypes` (string array) is passed to `React.addTransitionType` inside the navigation Transition.
  https://nextjs.org/docs/app/api-reference/functions/use-router#disabling-scroll-to-top
- There is no `router.events`. Compose `usePathname()` and `useSearchParams()` in a `useEffect`, in a Client Component wrapped in `<Suspense fallback={null}>`.
  https://nextjs.org/docs/app/api-reference/functions/use-router#router-events
- With Cache Components, client state survives navigations. `router.bfcacheId` used as a `key` resets a subtree on fresh navigations but keeps it on back and forward. Prefer resetting in an event handler.
  ```tsx
  const { bfcacheId } = useRouter()
  return <form key={bfcacheId}>{/* ... */}</form>
  ```
  https://nextjs.org/docs/app/api-reference/functions/use-router#bfcacheid

## `useSearchParams` and `searchParams`

- In a page (Server Component), read the `searchParams` prop. Layouts do not receive it.
  https://nextjs.org/docs/app/api-reference/functions/use-search-params
- `useSearchParams` returns a read-only `URLSearchParams`. To update, build a new string and navigate:
  ```tsx
  const params = new URLSearchParams(searchParams.toString())
  params.set('sort', 'asc')
  router.push(pathname + '?' + params.toString())
  ```
  https://nextjs.org/docs/app/api-reference/functions/use-search-params#updating-searchparams
- Wrap a Client Component that calls `useSearchParams` in `<Suspense>`. Without it a static page fails the production build ("Missing Suspense boundary with useSearchParams"). Dev does not show it.
  https://nextjs.org/docs/app/api-reference/functions/use-search-params#prerendering
- For a dynamic route, `await connection()` in a Server Component replaces `export const dynamic = 'force-dynamic'`.
  https://nextjs.org/docs/app/api-reference/functions/use-search-params#dynamic-rendering
- Update the URL without navigating with `window.history.pushState` or `replaceState`. Both sync with `usePathname` and `useSearchParams`.
  https://nextjs.org/docs/app/getting-started/linking-and-navigating#native-history-api

## Pathname and segment hooks

- `usePathname` and `useSelectedLayoutSegment(s)` are Client Component hooks, usually imported into a Server Layout.
  https://nextjs.org/docs/app/api-reference/functions/use-pathname
- With Cache Components these hooks suspend on routes whose params are not covered by `generateStaticParams`. The build fails without a `<Suspense>` ancestor, even for a static sidebar in a layout. Wrap just that component.
  https://nextjs.org/docs/app/api-reference/functions/use-pathname#cache-components
- On a prerendered page reached through a rewrite (`next.config` or Proxy), the HTML is built for the source path while the client reads the rewritten path, which can mismatch on hydration. Render a stable fallback and update after mount.
  https://nextjs.org/docs/app/api-reference/functions/use-pathname#avoid-hydration-mismatch-with-rewrites
- `useSelectedLayoutSegment` reads one level below its layout. Catch-all segments come back as one joined string (`'a/b/c'`), and inside the array of `useSelectedLayoutSegments` as one joined item.
  https://nextjs.org/docs/app/api-reference/functions/use-selected-layout-segments
- `useSelectedLayoutSegments` includes route groups such as `(group)`: filter items that start with a bracket.
  https://nextjs.org/docs/app/api-reference/functions/use-selected-layout-segments

## Choosing a redirect

| Situation | Use | Status |
| --- | --- | --- |
| Known path map | `redirects` in `next.config` | 307 or 308 |
| Conditional, before render | `NextResponse.redirect` in Proxy | any |
| During render, in an action or handler | `redirect()` / `permanentRedirect()` | 307 / 308 (303 from a Server Action without JavaScript) |
| Event handler in a Client Component | `router.push()` | N/A |

https://nextjs.org/docs/app/guides/redirecting

- `next.config` `redirects` run before Proxy. Past about 1000 redirects, read a redirect map in Proxy instead.
  https://nextjs.org/docs/app/guides/redirecting#managing-redirects-at-scale-advanced

## `redirect` and the interrupt functions

- `redirect`, `permanentRedirect`, `notFound`, `forbidden` and `unauthorized` work by throwing. Call them outside `try` blocks, or the catch swallows the interrupt.
  https://nextjs.org/docs/app/api-reference/functions/redirect#behavior
- No `return` needed after them: they are typed `never`.
  ```tsx
  const user = await fetchUser(id)
  if (!user) { notFound() }
  ```
  https://nextjs.org/docs/app/api-reference/functions/not-found
- `redirect` works in Client Components during render, not in event handlers.
  https://nextjs.org/docs/app/api-reference/functions/redirect#behavior
- `redirect` defaults to `replace`, except `push` in Server Actions.
  https://nextjs.org/docs/app/api-reference/functions/redirect#parameters
- An interrupt inside an un-awaited promise is never caught: no UI, in development only an `unhandledRejection` in the server log. Always await.
  https://nextjs.org/docs/app/api-reference/functions/forbidden
- `forbidden()` and `unauthorized()` need `experimental.authInterrupts: true` and cannot be called in the root layout.
  https://nextjs.org/docs/app/api-reference/functions/unauthorized
- A real 401, 403 or 404 status needs the check before streaming starts. Afterwards the UI renders with status 200 and `noindex`.
  https://nextjs.org/docs/app/api-reference/file-conventions/forbidden

## `catchError` and `unstable_rethrow`

- `catchError` (from `next/error`, `'use client'`) builds a component-level boundary. The fallback gets `(props, { error, retry, reset })`, `error` typed `unknown`.
  ```tsx
  import { catchError, type ErrorInfo } from 'next/error'
  export default catchError(ErrorFallback)
  ```
  https://nextjs.org/docs/app/api-reference/functions/catchError
- Default to `retry()`: it re-fetches and re-renders in a Transition. `reset()` only clears state and cannot recover from Server Component errors.
  https://nextjs.org/docs/app/api-reference/functions/catchError#recovering-from-errors
- Do not wrap an `error.js` default export with `catchError`: it already renders inside a built-in boundary.
  https://nextjs.org/docs/app/api-reference/functions/catchError
- When a `catch` can see both app errors and framework interrupts, call `unstable_rethrow(err)` first in the block.
  ```tsx
  } catch (err) {
    unstable_rethrow(err)
    console.error(err)
  }
  ```
  https://nextjs.org/docs/app/api-reference/functions/unstable_rethrow
- When a route segment is marked to throw unless static, a Request-time API call also throws, and that throw must not be caught either.
  https://nextjs.org/docs/app/api-reference/functions/unstable_rethrow

## `navigation()`

- `await navigation()` (from `next/cache`) defers the runtime work below it out of the App Shell and per-link prefetches until the user navigates. Requires `cacheComponents`.
  ```tsx
  async function Comments({ params }) {
    await navigation()
    const { slug } = await params
  }
  ```
  https://nextjs.org/docs/app/api-reference/functions/navigation#usage
- Wrap the awaiting component in `<Suspense>`: outside one, development shows an Instant Insight. It throws inside `use cache`, `unstable_cache()`, `after()`, `generateStaticParams` and Client Components.
  https://nextjs.org/docs/app/api-reference/functions/navigation#good-to-know
- Unlike `connection()`, work after `navigation()` can still use `use cache`.
  https://nextjs.org/docs/app/api-reference/functions/navigation#how-navigation-differs-from-connection
