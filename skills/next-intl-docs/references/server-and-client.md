# Server and Client Components

## Contents

- Which API where
- Client Components
- Provider errors
- Metadata, Server Actions and Route Handlers
- Sitemap, OG images and manifest
- Not found and error files

## Which API where

- Default: keep i18n on the server. Messages never leave it and no client i18n code or message splitting is needed. https://next-intl.dev/docs/environments/server-client-components
- Async Server Components use the awaitable APIs from `next-intl/server`: `getTranslations`, `getFormatter`, `getNow`, `getTimeZone`, `getMessages`, `getLocale`. Hooks cannot run in async components. https://next-intl.dev/docs/environments/server-client-components
- Non-async components import hooks from `next-intl` (`useTranslations`, `useFormatter`, `useLocale`, `useNow`, `useTimeZone`). next-intl picks the server or client implementation through `react-server` conditional exports. https://next-intl.dev/docs/environments/server-client-components
- Prefer non-async components when they can be shared: they run on server and client and test more simply. Both styles perform the same. https://next-intl.dev/docs/environments/server-client-components
- A hook in a Server Component may suspend once while `i18n/request.ts` initializes, re-running the component code before the hook. After that it runs synchronously. https://next-intl.dev/docs/environments/server-client-components

## Client Components

Options, best performance first. https://next-intl.dev/docs/environments/server-client-components

1. Translate in a Server Component and pass the labels as props or `children`.
2. Move dynamic state (pagination, filters) to the server: search params (shareable URL, history), cookies or the database.
3. Wrap only the interactive part in a `NextIntlClientProvider` with picked messages.
4. Provide all messages, which is what a bare `NextIntlClientProvider` does.

```tsx
import pick from 'lodash/pick';
import {NextIntlClientProvider, useMessages} from 'next-intl';

const messages = useMessages();
<NextIntlClientProvider messages={pick(messages, 'ClientCounter')}>
  <ClientCounter />
</NextIntlClientProvider>
```
https://next-intl.dev/docs/environments/server-client-components

- A bare provider sends ALL messages. `messages={null}` sends none. https://next-intl.dev/docs/environments/server-client-components
- Which namespaces to pick is chosen by hand from what the wrapped components use. Automatic tree shaking is not available. https://next-intl.dev/docs/environments/server-client-components
- Messages passed to the provider are emitted in the streamed markup, which can raise total blocking time and INP. Measure before optimizing. https://next-intl.dev/docs/environments/server-client-components
- Forms: move the interactive parts into leaf Client Components rather than marking the whole form `'use client'`. https://next-intl.dev/docs/environments/server-client-components

## Provider errors

- "Failed to call `useTranslations` because the context from `NextIntlClientProvider` was not found": wrap the Client Component in the provider, or the component ended up in the client module graph by accident and should be passed as `children`. https://next-intl.dev/docs/environments/server-client-components
- "Functions cannot be passed directly to Client Components": `onError` and `getMessageFallback` are not serializable. Define them in your own `'use client'` component that renders `NextIntlClientProvider`, and render that component inside the server-rendered `NextIntlClientProvider`, never in its place: the inner provider inherits locale, messages and the rest from the outer one, so pass it only `onError` and `getMessageFallback`. https://next-intl.dev/docs/environments/server-client-components, https://next-intl.dev/docs/usage/configuration

## Metadata, Server Actions and Route Handlers

- Use the awaitable functions of `next-intl/server`, and pass `locale` explicitly where it cannot be inferred. https://next-intl.dev/docs/environments/actions-metadata-route-handlers

```tsx
export async function generateMetadata({params}) {
  const {locale} = await params;
  const t = await getTranslations({locale, namespace: 'Metadata'});
  return {title: t('title')};
}
```
https://next-intl.dev/docs/routing/setup , https://next-intl.dev/docs/environments/actions-metadata-route-handlers

```tsx
// Server Action, locale sent by the client
const t = await getTranslations({namespace: 'LoginForm', locale});
```
https://next-intl.dev/docs/environments/actions-metadata-route-handlers

- Without a `[locale]` segment, give the form `key={locale}` so a displayed result does not stay in the old locale after a switch. With `[locale]` this is automatic. https://next-intl.dev/docs/environments/actions-metadata-route-handlers
- Zod: keep the schema locale-agnostic and localize through the per-parse `error` map. A shared helper such as `getValidationErrorMap` can reuse the messages. https://next-intl.dev/docs/environments/actions-metadata-route-handlers
- Route Handlers get the locale from a search param, a layout segment or `accept-language`, and validate it: https://next-intl.dev/docs/environments/actions-metadata-route-handlers

```tsx
if (!hasLocale(routing.locales, locale)) {
  return NextResponse.json({error: 'Invalid locale'}, {status: 400});
}
```
https://next-intl.dev/docs/environments/actions-metadata-route-handlers

## Sitemap, OG images and manifest

- Alternate languages already go out in a `link` response header. Add sitemap `alternates` with `getPathname` only for specific needs such as locale-dependent CMS slugs. https://next-intl.dev/docs/environments/actions-metadata-route-handlers
- With a customized `localePrefix`, Next.js does not know about the proxy rewrite and `opengraph-image` may be unreachable. Exclude it: `'/((?!api|_next|_vercel|.*/opengraph-image|.*\\..*).*)'`. https://next-intl.dev/docs/environments/actions-metadata-route-handlers
- The manifest lives outside `[locale]`, so pass a representative locale explicitly. https://next-intl.dev/docs/environments/actions-metadata-route-handlers

## Not found and error files

- Localized 404: `app/[locale]/not-found.tsx`. It renders only when `notFound()` is called inside a route. https://next-intl.dev/docs/environments/error-files
- Unknown routes: add `app/[locale]/[...rest]/page.tsx` that calls `notFound()`. https://next-intl.dev/docs/environments/error-files

```tsx
import {notFound} from 'next/navigation';
export default function CatchAllPage() {
  notFound();
}
```
https://next-intl.dev/docs/environments/error-files

- Unknown `[locale]` values reach the 404 only when `i18n/request.ts` calls `notFound()`. https://next-intl.dev/docs/environments/error-files
- Requests the matcher skips (`/unknown.txt`) have no locale. Next.js's experimental `global-not-found` covers them. https://next-intl.dev/docs/environments/error-files
- `error.tsx` is a Client Component. Provide at least the `Error` namespace in the layout: `<NextIntlClientProvider messages={pick(messages, 'Error')}>`. https://next-intl.dev/docs/environments/error-files
- `error.tsx` loads right after app initialization. Export `lazy(() => import('./Error'))` to keep next-intl out of the initial bundle. https://next-intl.dev/docs/environments/error-files
