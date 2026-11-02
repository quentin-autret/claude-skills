# Authentication and data security

Data Access Layer, session checks, Proxy limits, Cache Components with sessions, environment variables, CSP nonces, Draft Mode and taint in Next.js 16.4.4.

## Contents

- Data Access Layer
- Where to check authentication
- Sessions and cookies
- Auth with Cache Components
- Environment variables
- Content Security Policy
- Draft Mode
- Taint and dev origins

## Data Access Layer

- Pick one data approach and do not mix: a Data Access Layer (DAL) for new projects, existing HTTP APIs for large apps, component-level queries for prototypes only.
  https://nextjs.org/docs/app/guides/data-security#data-fetching-approaches
- The DAL runs on the server only, checks authorization and returns minimal DTOs. Only the DAL reads secrets from `process.env`.
  https://nextjs.org/docs/app/guides/data-security#data-access-layer
- Put `import 'server-only'` in DAL, session and DTO modules.
  https://nextjs.org/docs/app/guides/data-security#preventing-client-side-execution-of-server-only-code
- Memoize the session check per render with React `cache`:
  ```ts
  export const verifySession = cache(async () => {
    const cookie = (await cookies()).get('session')?.value
    const session = await decrypt(cookie)
    if (!session?.userId) { redirect('/login') }
    return { isAuth: true, userId: session.userId }
  })
  ```
  https://nextjs.org/docs/app/guides/authentication#creating-a-data-access-layer-dal
- React `cache` does nothing in Server Actions and Route Handlers (no render pass): the function runs on every call.
  https://nextjs.org/docs/app/guides/authentication#creating-a-data-access-layer-dal
- Keep `'use server'` actions thin. Delegate to a `server-only` DAL function that does auth, authorization and the query.
  https://nextjs.org/docs/app/guides/data-security#using-a-data-access-layer-for-mutations

## Where to check authentication

- Check next to the data (DAL or leaf component), not in layouts. Layouts do not re-render on navigation and do not control whether nested segments or slots run.
  https://nextjs.org/docs/app/guides/authentication#layouts-and-auth-checks
- Never `return null` in a layout as a guard: nested segments and Server Actions stay reachable.
  https://nextjs.org/docs/app/guides/authentication#auth-checks-in-leaf-components
- Treat every Server Action and Route Handler as a public endpoint. A page redirect does not protect an inline action.
  https://nextjs.org/docs/app/guides/authentication#server-actions
- Check ownership, not only identity: `if (post.authorId !== session.user.id) throw new Error('Forbidden')`.
  https://nextjs.org/docs/app/guides/data-security#authentication-and-authorization
- Treat `params`, `searchParams`, headers and form data as untrusted. Never gate admin UI on a search param.
  https://nextjs.org/docs/app/guides/data-security#validating-client-input
- Proxy does only optimistic cookie checks, no database calls (it also runs on prefetches). Secure checks against the database belong in the DAL. Proxy is the way to protect static routes that share data between users.
  https://nextjs.org/docs/app/guides/authentication#optimistic-checks-with-proxy-optional
- Never mutate during render (no cookie deletes, no revalidation). Log out with a Server Action in a form.
  https://nextjs.org/docs/app/guides/data-security#avoiding-side-effects-during-rendering
- Rate-limit expensive actions (email, writes).
  https://nextjs.org/docs/app/guides/data-security#rate-limiting

## Sessions and cookies

- Keep the session payload minimal (id, role), without personal data or secrets.
  https://nextjs.org/docs/app/guides/authentication#2-encrypting-and-decrypting-sessions
- `cookies()` reads in Server Components but writes only in Server Actions and Route Handlers.
  https://nextjs.org/docs/app/api-reference/functions/cookies#understanding-cookie-behavior-in-server-components
- Only `path` has a default (`'/'`). Set the rest explicitly:
  ```ts
  cookieStore.set('session', session, { httpOnly: true, secure: true, expires: expiresAt, sameSite: 'lax', path: '/' })
  ```
  https://nextjs.org/docs/app/guides/authentication#3-setting-cookies-recommended-options
- Awaiting `cookies()` or the DAL at the top of a layout holds `{children}` and delays the first chunk. Move it into a nested component under `<Suspense>`.
  https://nextjs.org/docs/app/guides/authentication#auth-and-streaming

## Auth with Cache Components

- Plain `use cache` and `use cache: remote` cannot read `cookies()`. `use cache: private` can, and keeps results in the browser only.
  https://nextjs.org/docs/app/guides/authentication-with-cache-components#step-1-read-the-current-user
- Cache keys and tags are stored in plain text: key on a stable user id, never on tokens or raw emails.
  https://nextjs.org/docs/app/guides/authentication-with-cache-components#step-4-cache-session-derived-data
- Keep the cached `getNotesByUserId(userId)` unexported, and resolve the user inside the exported getter, so no caller can pass another user's id.
  https://nextjs.org/docs/app/guides/authentication-with-cache-components#step-4-cache-session-derived-data
- After a write, re-read the session in the action and call ``updateTag(`notes:${userId}`)``.
  https://nextjs.org/docs/app/guides/authentication-with-cache-components#step-5-update-session-derived-data
- Calling `unauthorized()` or `forbidden()` after streaming starts keeps status 200.
  https://nextjs.org/docs/app/guides/optimizing-the-static-shell#step-2-move-authentication-behind-suspense

## Environment variables

- `NEXT_PUBLIC_*` values are inlined at `next build` and frozen per build. Dynamic lookups such as `process.env[name]` are not inlined.
  https://nextjs.org/docs/app/guides/environment-variables#bundling-environment-variables-for-the-browser
- For runtime values (one build or Docker image promoted across environments), read them on the server during dynamic rendering, after `await connection()`.
  https://nextjs.org/docs/app/guides/environment-variables#runtime-environment-variables
- `.env*` files load from the project root only, never from `src/`. Order: `process.env`, `.env.$(NODE_ENV).local`, `.env.local` (skipped in test), `.env.$(NODE_ENV)`, `.env`.
  https://nextjs.org/docs/app/guides/environment-variables#environment-variable-load-order
- `$VAR` expands inside `.env`. Escape a literal dollar as `\$`. Load the files outside Next.js with `loadEnvConfig(process.cwd())` from `@next/env`.
  https://nextjs.org/docs/app/guides/environment-variables#referencing-other-variables
  https://nextjs.org/docs/app/guides/environment-variables#loading-environment-variables-with-next-env
- The `env` key of `next.config` always bundles values into client JS, whatever the prefix. `NEXT_PUBLIC_` only has an effect through the environment or `.env` files.
  https://nextjs.org/docs/app/api-reference/config/next-config-js/env

## Content Security Policy

- Generate a nonce per request in `proxy.ts`, set `Content-Security-Policy` on both request and response headers. Next.js reads `'nonce-...'` and applies it to its scripts.
  https://nextjs.org/docs/app/guides/content-security-policy#how-nonces-work-in-nextjs
- A nonce CSP forces dynamic rendering of every page: no static optimization, no ISR, no PPR.
  https://nextjs.org/docs/app/guides/content-security-policy#static-vs-dynamic-rendering-with-csp
- The CSP matcher should skip prefetches with `missing: [{ type: 'header', key: 'next-router-prefetch' }]` and skip static assets.
  https://nextjs.org/docs/app/guides/content-security-policy#adding-a-nonce-with-proxy
- `'unsafe-eval'` is only needed in dev. To keep static pages under a strict CSP, use experimental SRI (`experimental.sri`).
  https://nextjs.org/docs/app/guides/content-security-policy#subresource-integrity-experimental

## Draft Mode

- The enable handler must check a secret and the slug, then redirect to the slug fetched from the CMS, never the query param (open redirect).
  https://nextjs.org/docs/app/guides/draft-mode#step-2-access-the-route-handler-from-your-headless-cms
- Exit through POST (Server Action in a `<form>` or POST Route Handler). A `<Link>` to the exit handler needs `prefetch={false}`, or a prefetch clears the cookie.
  https://nextjs.org/docs/app/api-reference/functions/draft-mode#disabling-draft-mode
- Draft Mode bypasses every cache. Inside `use cache`, `isEnabled` is readable but `enable()` and `disable()` throw.
  https://nextjs.org/docs/app/guides/draft-mode#what-draft-mode-does

## Taint and dev origins

- `experimental.taint` tracks object references only: copies, derived data and `process.env.X` strings pass freely. Never the only protection.
  https://nextjs.org/docs/app/api-reference/config/next-config-js/taint
- The dev server blocks cross-origin requests to dev assets. Tunnels need `allowedDevOrigins` (hostnames only, `*.x.dev` does not match `x.dev`).
  https://nextjs.org/docs/app/api-reference/config/next-config-js/allowedDevOrigins
