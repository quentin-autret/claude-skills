# Routing

## Contents

- Setup files
- Matcher
- Static rendering
- Locale prefix
- Detection and cookie
- Domains
- Alternate links
- Localized pathnames
- basePath and unknown locales
- Static export
- Troubleshooting
- Old patterns

## Setup files

```ts
// src/i18n/routing.ts
import {defineRouting} from 'next-intl/routing';
export const routing = defineRouting({locales: ['en', 'de'], defaultLocale: 'en'});
```

```ts
// src/proxy.ts
import createMiddleware from 'next-intl/middleware';
import {routing} from './i18n/routing';

export default createMiddleware(routing);
export const config = {matcher: '/((?!api|trpc|_next|_vercel|.*\\..*).*)'};
```

```tsx
// src/i18n/request.ts
import * as rootParams from 'next/root-params';
import {notFound} from 'next/navigation';
import {getRequestConfig} from 'next-intl/server';
import {hasLocale} from 'next-intl';
import {routing} from './routing';

export default getRequestConfig(async ({locale}) => {
  if (!locale) {
    const paramValue = await rootParams.locale();
    if (hasLocale(routing.locales, paramValue)) locale = paramValue;
    else notFound();
  }
  return {locale /* messages, timeZone ... */};
});
```
https://next-intl.dev/docs/routing/setup

- Define routing once in `routing.ts` and share it between the proxy and the navigation APIs. https://next-intl.dev/docs/routing/configuration
- Move all layouts and pages into `src/app/[locale]/`. https://next-intl.dev/docs/routing/setup
- `proxy.ts` was named `middleware.ts` until Next.js 16. The `next-intl/middleware` import and `createMiddleware` stay. https://next-intl.dev/docs/routing/middleware
- `next/root-params` is on by default in Next.js 16.3+. Earlier versions need `experimental.rootParams` in the Next.js config. https://next-intl.dev/docs/routing/setup
- The `locale` argument of `getRequestConfig` serves edge cases that override the locale. It may be undefined, hence the fallback. https://next-intl.dev/docs/routing/setup
- Add a `not-found` page: `notFound()` runs when no valid locale is found. https://next-intl.dev/docs/routing/middleware

Composing with other proxy logic:

```ts
const handleI18nRouting = createMiddleware(routing);
export default async function proxy(request: NextRequest) {
  const response = handleI18nRouting(request);
  // edit response here
  return response;
}
```
https://next-intl.dev/docs/routing/middleware

## Matcher

- Default `'/((?!api|trpc|_next|_vercel|.*\\..*).*)'` skips any path with a dot. https://next-intl.dev/docs/routing/middleware
- Entries are OR-ed. For pathnames with dots, add one: `matcher: ['/((?!api|trpc|_next|_vercel|.*\\..*).*)', '/([\\w-]+)?/users/(.+)']`. https://next-intl.dev/docs/routing/middleware
- Exclude third-party internal endpoints (for example `/_vercel/insights/view`) so they are not rewritten. https://next-intl.dev/docs/routing/middleware
- With a customized `localePrefix`, exclude OG image routes, which Next.js cannot reach through the rewrite: `'/((?!api|_next|_vercel|.*/opengraph-image|.*\\..*).*)'`. https://next-intl.dev/docs/environments/actions-metadata-route-handlers

## Static rendering

- With `next/root-params`, the app is eligible for static rendering automatically. Add `generateStaticParams` to the `[locale]` layout (all pages) or to single pages. https://next-intl.dev/docs/routing/setup

```tsx
export function generateStaticParams() {
  return routing.locales.map((locale) => ({locale}));
}
```
https://next-intl.dev/docs/routing/setup

- Returning only some locales, or none, is allowed. https://next-intl.dev/docs/routing/setup
- Legacy `setRequestLocale` setup only: in `generateMetadata`, forward the locale from `params`: `getTranslations({locale, namespace: 'Metadata'})`. https://next-intl.dev/docs/routing/setup
- Do not reach for `force-static`. Follow the static rendering setup. https://next-intl.dev/docs/routing/middleware

## Locale prefix

- Default `localePrefix: 'always'`: `/en/about`. https://next-intl.dev/docs/routing/configuration
- `'as-needed'`: the default locale has no prefix. The matcher must catch unprefixed paths. The cookie may redirect `/` to `/de`. A superfluous `/en/about` redirects to `/about`. https://next-intl.dev/docs/routing/configuration
- `'never'`: pages still live in `[locale]`, alternate links are disabled, the matcher must catch unprefixed paths. Consider a longer cookie `maxAge`. If you only want a user-setting locale, skip locale routing entirely. https://next-intl.dev/docs/routing/configuration
- Custom prefixes:

```ts
localePrefix: {mode: 'always', prefixes: {'en-US': '/us', 'de-AT': '/eu/at'}}
```
https://next-intl.dev/docs/routing/configuration

- The matcher must match the custom prefixes. The prefix is rewritten internally, so `[locale]` holds `en-US`, never `/us`, and the prefix cannot be read. Derive the region with `new Intl.Locale(locale)`. https://next-intl.dev/docs/routing/configuration

## Detection and cookie

- Order with prefixes: URL prefix, then `NEXT_LOCALE` cookie, then `accept-language`, then `defaultLocale`. Visiting a prefixed route updates the cookie. https://next-intl.dev/docs/routing/middleware
- `accept-language` uses the "best fit" algorithm of `@formatjs/intl-localematcher`, so `en-GB` can match `en-US`. https://next-intl.dev/docs/routing/middleware
- `localeDetection: false` ignores cookie and `accept-language`. Only the prefix and a matching domain decide. https://next-intl.dev/docs/routing/configuration
- The cookie is set only when the user picks a locale that differs from `accept-language`. It is a session cookie (no `max-age`) by default, `sameSite: lax`, `path` following `basePath`. https://next-intl.dev/docs/routing/configuration
- Custom options merge with the defaults: `localeCookie: {name: 'USER_LOCALE', maxAge: 60 * 60 * 24 * 365}`. `localeCookie: false` disables it. https://next-intl.dev/docs/routing/configuration

## Domains

- Locales must be unique across domains: use regional variants (`en-US`, `en-GB`). https://next-intl.dev/docs/routing/configuration
- No matching domain (for example `localhost`) falls back to prefix matching. Each domain can override `localePrefix`. https://next-intl.dev/docs/routing/configuration
- The host is read from `x-forwarded-host`, then `host`. A locale unsupported on the current domain redirects to a domain that supports it. https://next-intl.dev/docs/routing/middleware

## Alternate links

- The proxy sets a `link` header with `hreflang` and `x-default` entries, honoring `domains`, `pathnames` and `basePath`. https://next-intl.dev/docs/routing/configuration
- Pages that exist only in some locales, or CMS pathnames: `alternateLinks: false`, then provide links through the sitemap or the Metadata API. Small tweaks: compose the proxy and edit the `link` header. https://next-intl.dev/docs/routing/configuration

## Localized pathnames

```ts
pathnames: {
  '/about': {de: '/über-uns'},
  '/news/[articleSlug]': {de: '/neuigkeiten/[articleSlug]'}
}
```
https://next-intl.dev/docs/routing/configuration

- Keys are internal pathnames (file-system routes), values the external ones per locale. https://next-intl.dev/docs/routing/configuration
- Static pathnames win over overlapping dynamic ones. Catch-all and optional catch-all work. Non-ASCII is encoded and trailing slashes normalized. https://next-intl.dev/docs/routing/configuration
- CMS URLs: localize only the static part and take a localized slug from the CMS. The locale switcher and alternate links must know the CMS URLs. https://next-intl.dev/docs/routing/configuration
- `revalidatePath`: the localized path (`/de/neuigkeiten/x`) for dynamically generated routes, the internal path (`/de/news/x`) for routes from `generateStaticParams`. When in doubt, `revalidateTag`. https://next-intl.dev/docs/routing/configuration

## basePath and unknown locales

- `basePath` and `trailingSlash` from the Next.js config are honored. With a `basePath`, the matcher needs an explicit `'/'` entry, because it is relative to the base path. https://next-intl.dev/docs/routing/configuration
- Locales unknown at build time: build the routing config per request in the proxy, call `createNavigation` without `locales` and keep the rest in sync. https://next-intl.dev/docs/routing/configuration

## Static export

- With `output: 'export'` no proxy runs: keep `localePrefix: 'always'`, no `pathnames`, no server negotiation, static rendering required. https://next-intl.dev/docs/routing/middleware
- Add `app/page.tsx` that redirects to the default locale and an `app/layout.tsx` that returns `children`. https://next-intl.dev/docs/routing/middleware

## Troubleshooting

- Proxy not running: check the file location, the matcher (dots included) and any composition. https://next-intl.dev/docs/routing/middleware
- Unlocalized content despite a prefix means the proxy did not run. https://next-intl.dev/docs/routing/middleware
- "Unable to find next-intl locale": return a `locale` from `getRequestConfig` where the proxy is not expected to run. https://next-intl.dev/docs/routing/middleware

## Old patterns

| Old | Current |
| :-- | :-- |
| `src/middleware.ts` | `src/proxy.ts` (Next.js 16), same code (https://next-intl.dev/docs/routing/setup) |
| `setRequestLocale(locale)` called first in every layout and page | `next/root-params` in `i18n/request.ts` (https://next-intl.dev/docs/routing/setup) |
| `getRequestConfig(async ({requestLocale}) => ...)` | `({locale})` with `rootParams.locale()` (https://next-intl.dev/docs/usage/configuration) |

- `setRequestLocale` stays for backwards compatibility. Without it, next-intl read the `x-next-intl-locale` header, which forced dynamic rendering. https://next-intl.dev/docs/routing/setup
- Code still on `requestLocale`: it is `undefined` outside `[locale]`. Invalid values (catch-alls like `/unknown.txt`) must be replaced by a valid locale, optionally with `notFound()` in the root layout. https://next-intl.dev/docs/usage/configuration
