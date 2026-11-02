---
name: next-intl-docs
description: Teaches current next-intl 4 practices for internationalizing Next.js apps, drawn from the official documentation. Use when setting up next-intl with the App Router, writing i18n/request.ts, adding a [locale] segment with defineRouting, createMiddleware in proxy.ts (formerly middleware.ts), building locale switchers with createNavigation, translating with useTranslations or getTranslations, passing messages to NextIntlClientProvider, writing ICU messages (plurals, select, rich text), formatting dates, numbers and lists with useFormatter, translating metadata, Server Actions, Route Handlers, localized 404 and error pages, static rendering with next/root-params, type-safe messages through AppConfig, testing with Jest, Vitest, Storybook, or extracting inline messages with useExtracted. Triggers on next-intl, useTranslations, getTranslations, NextIntlClientProvider, getRequestConfig, i18n routing, locale prefix, localePrefix, NEXT_LOCALE, hreflang, setRequestLocale, ICU messages, App Router internationalization.
metadata:
  version: "next-intl 4.x (docs as of 4.14.9)"
  source: https://next-intl.dev/docs/getting-started
  fetched: "2026-11-02"
---

# next-intl 4

This skill covers next-intl 4.x as documented on https://next-intl.dev/docs/getting-started (docs as of 4.14.9). It holds what differs from habit: surprising defaults, pitfalls, legacy APIs and the patterns the docs recommend. Each claim in the reference files ends with the page it comes from.

## How to navigate

1. Read the quick start and quick rules below. They catch the most frequent mistakes.
2. Open the one reference file that matches the task, from the table at the end.
3. When code uses `setRequestLocale`, `requestLocale` or `middleware.ts`, read the "Old patterns" table first.

## Quick start (App Router)

The App Router is the recommended path. The Pages Router stays supported, see `references/setup.md`. https://next-intl.dev/docs/getting-started/pages-router

```ts
// next.config.ts
import {NextConfig} from 'next';
import createNextIntlPlugin from 'next-intl/plugin';

const nextConfig: NextConfig = {};
const withNextIntl = createNextIntlPlugin();
export default withNextIntl(nextConfig);
```
https://next-intl.dev/docs/getting-started/app-router

Without locale-based URLs, `src/i18n/request.ts` resolves the locale from a user preference, a cookie at its simplest:

```tsx
// src/i18n/request.ts
import {cookies} from 'next/headers';
import {getRequestConfig} from 'next-intl/server';

export default getRequestConfig(async () => {
  const store = await cookies();
  const locale = store.get('locale')?.value || 'en';
  return {
    locale,
    messages: (await import(`../../messages/${locale}.json`)).default
  };
});
```
https://next-intl.dev/docs/getting-started/app-router

```tsx
// src/app/layout.tsx
import {NextIntlClientProvider} from 'next-intl';
import {getLocale} from 'next-intl/server';

export default async function RootLayout({children}) {
  const locale = await getLocale();
  return (
    <html lang={locale}>
      <body>
        <NextIntlClientProvider>{children}</NextIntlClientProvider>
      </body>
    </html>
  );
}
```
https://next-intl.dev/docs/getting-started/app-router , https://next-intl.dev/docs/usage/configuration

```tsx
import {useTranslations} from 'next-intl';
import {getTranslations} from 'next-intl/server';

export function HomePage() {
  const t = useTranslations('HomePage'); // non-async component
  return <h1>{t('title')}</h1>;
}

export async function Profile() {
  const t = await getTranslations('ProfilePage'); // async component
  return <h1>{t('title')}</h1>;
}
```
https://next-intl.dev/docs/getting-started/app-router

For `/en/about` style URLs, add `src/i18n/routing.ts`, `src/proxy.ts`, `src/i18n/navigation.ts` and a top-level `[locale]` segment: `references/routing.md`. https://next-intl.dev/docs/routing/setup

## Quick rules

### Setup

- The plugin finds `i18n/request.ts` in `src/` or the project root (`.ts`, `.tsx`, `.js`, `.jsx`). Elsewhere, pass the path: `createNextIntlPlugin('./somewhere/else/request.ts')`. https://next-intl.dev/docs/getting-started/app-router
- The docs example hardcodes `const locale = 'en'`. Replace it with real resolution (cookie, user setting or `next/root-params`). https://next-intl.dev/docs/getting-started/app-router
- `getRequestConfig` runs once per request, in the Server Components render pass, so `cookies()` and `headers()` work there. https://next-intl.dev/docs/usage/configuration
- Set `timeZone` in the request config. The default is the time zone of the server runtime. https://next-intl.dev/docs/usage/configuration

### Routing

- Name the file `src/proxy.ts` on Next.js 16. The import stays `createMiddleware` from `next-intl/middleware`. https://next-intl.dev/docs/routing/setup
- Read the locale with `await rootParams.locale()` from `next/root-params`, validate it with `hasLocale(routing.locales, value)`, else `notFound()`. Next.js before 16.3 needs `experimental.rootParams`. https://next-intl.dev/docs/routing/setup
- Import `Link`, `redirect`, `usePathname`, `useRouter` and `getPathname` from your `i18n/navigation.ts` instead of `next/link` or `next/navigation`. https://next-intl.dev/docs/routing/navigation
- Switch locales with `router.replace(pathname, {locale})` from that module. `<Link locale>` is not prefetched, to avoid overwriting the locale cookie. https://next-intl.dev/docs/routing/navigation
- `redirect` always takes a locale: `redirect({href: '/login', locale})`. Write `return redirect(...)` so TypeScript narrows. https://next-intl.dev/docs/routing/navigation
- The default matcher skips paths with a dot. Add an explicit entry for routes like `/users/jane.doe`. https://next-intl.dev/docs/routing/middleware

### Client Components

- `NextIntlClientProvider` with no `messages` prop sends ALL messages to the client. Pass `messages={pick(messages, 'Namespace')}` or `messages={null}` to limit them. https://next-intl.dev/docs/environments/server-client-components
- Prefer translating in a Server Component and passing labels as props or `children` to the Client Component. https://next-intl.dev/docs/environments/server-client-components
- `onError` and `getMessageFallback` cannot be passed from a Server Component. Set them in your own `'use client'` provider nested inside the server one, which keeps inheriting locale and messages, so do not pass those by hand. https://next-intl.dev/docs/usage/configuration

### Messages

- Keys and namespaces cannot contain `.`, which means nesting. Use nested JSON only. https://next-intl.dev/docs/usage/translations
- ICU argument names and `select` values allow letters, digits and `_`, no dashes: map `en-GB` to `en_GB`. `select` needs an `other` case. https://next-intl.dev/docs/usage/translations
- Self-closing tags need a closing tag in messages: `<br></br>`. https://next-intl.dev/docs/usage/translations
- Use `getTranslations({locale, namespace})` in Server Actions, metadata and Route Handlers, where the locale may not be inferred. https://next-intl.dev/docs/environments/actions-metadata-route-handlers

### Tests

- Vitest with `createNavigation`: `server.deps.inline: ['next-intl']`. Jest: `transformIgnorePatterns: ['node_modules/(?!next-intl)/']`. next-intl is ESM-only. https://next-intl.dev/docs/environments/testing

## Old patterns

| Old | Current |
| :-- | :-- |
| `src/middleware.ts` | `src/proxy.ts` on Next.js 16, same `createMiddleware` code |
| `setRequestLocale(locale)` at the top of every layout and page | `next/root-params` in `i18n/request.ts`, static rendering works without it |
| `getRequestConfig(async ({requestLocale}) => ...)` | `getRequestConfig(async ({locale}) => ...)` with a `rootParams.locale()` fallback |
| Flat message keys `"HomePage.title"` | Nested objects `{"HomePage": {"title": ...}}` |
| `t.raw` HTML in `dangerouslySetInnerHTML` | `t.rich` (React) or `t.markup` (strings), `t.raw` only to skip parsing, sanitized |

Details: `references/routing.md` and `references/messages.md`.

## Reference files

| File | Read it for |
| :-- | :-- |
| [references/setup.md](references/setup.md) | Plugin, request config, provider defaults, locale without routing, message storage, Pages Router |
| [references/routing.md](references/routing.md) | `defineRouting`, `[locale]` segment, `proxy.ts`, matcher, `next/root-params`, static rendering, `localePrefix`, domains, cookie, alternate links, localized pathnames, static export, troubleshooting |
| [references/navigation.md](references/navigation.md) | `createNavigation`, `Link`, `useRouter`, `usePathname`, `redirect`, `getPathname`, locale switcher, typed `href` |
| [references/messages.md](references/messages.md) | `useTranslations`, namespaces, ICU plurals and select, rich text, `t.markup`, `t.raw`, `t.has`, arrays, missing messages, `onError`, time zone, `now` |
| [references/formatting.md](references/formatting.md) | `useFormatter`: dates, relative time, ranges, numbers, lists, display names, ICU skeletons, global formats |
| [references/server-and-client.md](references/server-and-client.md) | Server vs Client Components, picking messages, metadata, Server Actions, Zod, Route Handlers, sitemap, OG images, 404 and error files |
| [references/extraction.md](references/extraction.md) | Experimental `useExtracted` and `getExtracted`, plugin `experimental` options, PO files, precompile, custom codecs, monorepos |
| [references/typescript-and-tooling.md](references/typescript-and-tooling.md) | `AppConfig` augmentation, type-safe arguments, testing with Jest and Vitest, Storybook, `use-intl` core, runtime requirements, linting, translation management, agent conventions |

Crawled pages: `SOURCES.md`.
