# Setup

## Contents

- Choosing a setup
- Plugin
- Request config
- Provider
- Message storage
- Locale without routing
- Pages Router
- Old patterns

## Choosing a setup

- The App Router is the recommended path. The Pages Router remains fully supported. https://next-intl.dev/docs/getting-started/pages-router
- Default: no unique pathnames per locale. The locale comes from a user preference, at its simplest a cookie, read in `i18n/request.ts`. https://next-intl.dev/docs/getting-started/app-router
- Escape hatch: when the app needs `/en/about` style URLs, add a top-level `[locale]` segment and follow the routing setup (`defineRouting`, `proxy.ts`, `createNavigation`). https://next-intl.dev/docs/getting-started/app-router , https://next-intl.dev/docs/routing/setup
- Single-language apps can still use next-intl to keep labels and formatting out of the logic. https://next-intl.dev/docs/getting-started/pages-router

## Plugin

```ts
// next.config.ts
import {NextConfig} from 'next';
import createNextIntlPlugin from 'next-intl/plugin';

const nextConfig: NextConfig = {};
const withNextIntl = createNextIntlPlugin();
export default withNextIntl(nextConfig);
```
https://next-intl.dev/docs/getting-started/app-router

- CommonJS `next.config.js` uses `require('next-intl/plugin')` and `module.exports = withNextIntl(nextConfig)`. https://next-intl.dev/docs/getting-started/app-router
- The plugin looks for `./i18n/request.ts` under `src/` and in the project root, with extension `.ts`, `.tsx`, `.js` or `.jsx`. https://next-intl.dev/docs/usage/plugin
- Other location: `createNextIntlPlugin('./somewhere/else/request.ts')` or `createNextIntlPlugin({requestConfig: './somewhere/else/request.ts'})`. https://next-intl.dev/docs/usage/plugin
- The same plugin hosts the experimental options for message extraction, precompilation and message declarations. https://next-intl.dev/docs/usage/plugin

## Request config

```tsx
// src/i18n/request.ts
import {getRequestConfig} from 'next-intl/server';

export default getRequestConfig(async () => {
  const locale = 'en';
  return {
    locale,
    messages: (await import(`../../messages/${locale}.json`)).default
  };
});
```
https://next-intl.dev/docs/getting-started/app-router

- The hardcoded `'en'` is "static for now". Replace it with real resolution: a cookie or user setting, or `next/root-params` with locale routing. https://next-intl.dev/docs/getting-started/app-router , https://next-intl.dev/docs/routing/setup
- The import path `../../messages/${locale}.json` is relative to `src/i18n/request.ts`. Adjust it when the file moves. https://next-intl.dev/docs/getting-started/app-router
- The config is request-scoped. It runs once per request through React `cache`, triggered by the first component that uses i18n. https://next-intl.dev/docs/usage/configuration
- It runs during the Server Components render pass, so `cookies()` and `headers()` from `next/headers` work. Await them. https://next-intl.dev/docs/usage/configuration , https://next-intl.dev/docs/getting-started/app-router
- It configures server-only code. Client Components receive config through `NextIntlClientProvider`. https://next-intl.dev/docs/usage/configuration
- Return `timeZone` explicitly. Without it, the server runtime's time zone applies. https://next-intl.dev/docs/usage/configuration
- Server and client must share the same locale, time zone and "now" reference. The request config plus the provider keep them aligned. https://next-intl.dev/docs/design-principles

## Provider

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
https://next-intl.dev/docs/usage/configuration

- `NextIntlClientProvider` rendered from a Server Component takes no props in the basic setup. It inherits `locale`, `messages`, `now`, `timeZone` and `formats` from the request config. https://next-intl.dev/docs/getting-started/app-router , https://next-intl.dev/docs/usage/configuration
- Inheritance passes all messages to the client. Pass `messages={pick(messages, ['Navigation'])}` or `messages={null}` to limit them. https://next-intl.dev/docs/usage/configuration
- Nested providers inherit from their ancestors, but each prop is atomic: a nested `messages` replaces the parent's, merge them yourself. https://next-intl.dev/docs/usage/configuration
- `useLocale` returns the `i18n/request.ts` locale in Server Components and the provider's locale in Client Components. https://next-intl.dev/docs/usage/configuration

## Message storage

- Default: one JSON file per locale in the repo (`messages/en.json`, `messages/de-AT.json`), colocated with the code. It enables fast changes in development and type checking from the shape of local messages. https://next-intl.dev/docs/usage/configuration
- Nest messages by component or page namespace (`HomePage.title`). The nested style is the only style, and it is what makes typed messages possible. https://next-intl.dev/docs/design-principles
- Escape hatch: load non-default locales from a remote source such as a translation platform's CDN. Keep at least the default-locale messages local for typed messages. https://next-intl.dev/docs/design-principles
- Split messages by locale, and optionally by server, client or component, to limit what reaches the client. https://next-intl.dev/docs/design-principles
- Several JSON files per locale can be spread-merged at runtime in the request config. When the split is only organizational, an editor tool is the alternative. https://next-intl.dev/docs/usage/translations
- Pass the locale from `getLocale` to CMS and backend requests so content and prices follow the user's language. https://next-intl.dev/docs/design-principles

## Locale without routing

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

- To change the locale, change where it is read from: set the cookie or the user setting. With locale routing, use `Link` or `useRouter` from your navigation APIs instead. https://next-intl.dev/docs/usage/configuration
- With Server Actions and no `[locale]` segment, reset component state with `key={locale}` so a displayed message does not stay in the old locale after a switch. https://next-intl.dev/docs/environments/actions-metadata-route-handlers

## Pages Router

```jsx
// pages/_app.tsx
<NextIntlClientProvider
  locale={router.locale}
  timeZone="Europe/Vienna"
  messages={pageProps.messages}
>
  <Component {...pageProps} />
</NextIntlClientProvider>
```

```js
export async function getStaticProps(context) {
  return {
    props: {messages: (await import(`../../messages/${context.locale}.json`)).default}
  };
}
```
https://next-intl.dev/docs/getting-started/pages-router

- Multiple languages require Next.js i18n routing, which provides `router.locale`. https://next-intl.dev/docs/getting-started/pages-router
- `Europe/Vienna` is the docs' sample value. Set your own `timeZone`. https://next-intl.dev/docs/getting-started/pages-router

## Old patterns

| Old | Current |
| :-- | :-- |
| Flat dotted keys `{"HomePage.title": "..."}` | Nested objects, migrate with lodash `set` (https://next-intl.dev/docs/usage/translations) |
| `getRequestConfig(async ({requestLocale}) => ...)` | `({locale})` plus a `rootParams.locale()` fallback from `next/root-params` (https://next-intl.dev/docs/usage/configuration , https://next-intl.dev/docs/routing/setup) |
