# Navigation

## Navigation APIs

```ts
// src/i18n/navigation.ts
import {createNavigation} from 'next-intl/navigation';
import {routing} from './routing';

export const {Link, redirect, usePathname, useRouter, getPathname} = createNavigation(routing);
```
https://next-intl.dev/docs/routing/navigation

- Import these from `@/i18n/navigation` instead of `Link` from `next/link` or the router hooks from `next/navigation`. They are thin wrappers that add the locale prefix and localized pathnames. https://next-intl.dev/docs/routing/navigation
- Locales unknown at build time: call `createNavigation()` without `locales`. https://next-intl.dev/docs/routing/navigation

## Link

```tsx
<Link href="/about">About</Link>
<Link href={{pathname: '/users/[userId]', params: {userId: '5'}}}>Susan</Link>
```
https://next-intl.dev/docs/routing/navigation

- With `pathnames`, `href` is strictly typed and dynamic params go in `{pathname, params}`. Without `pathnames`, pass the final string. https://next-intl.dev/docs/routing/navigation
- Route unknown to the config: add `// @ts-expect-error`. It passes through as-is, absolute ones get a prefix. https://next-intl.dev/docs/routing/navigation
- `<Link locale="de">` always renders a prefix, even with `as-needed` and the default locale, so the cookie updates first. Links can be clicked before hydration. https://next-intl.dev/docs/routing/navigation
- `<Link locale>` is not prefetched, to avoid overwriting the locale cookie. https://next-intl.dev/docs/routing/navigation
- Wrapper components type their props with `ComponentProps<typeof Link>`. https://next-intl.dev/docs/routing/navigation
- Active link: compare `useSelectedLayoutSegment()` with `href` and set `aria-current`. https://next-intl.dev/docs/routing/navigation

## Switching the locale

Default: `useRouter` plus `usePathname`, in a small Client Component.

```tsx
'use client';
import {usePathname, useRouter} from '@/i18n/navigation';

const pathname = usePathname();
const router = useRouter();
router.replace(pathname, {locale: 'de'});
```
https://next-intl.dev/docs/routing/navigation

- With `pathnames`, also pass the params: `router.replace({pathname, params}, {locale})` with `params` from `useParams()`. https://next-intl.dev/docs/routing/navigation
- With `as-needed`, `useRouter` updates the cookie on the client and skips the redirect detour that `<Link locale>` takes. https://next-intl.dev/docs/routing/navigation
- Render the labels in a Server Component and make only the `select` a Client Component. https://next-intl.dev/docs/environments/server-client-components
- `usePathname` returns the internal template (`/news/[articleSlug]`) without the locale prefix. https://next-intl.dev/docs/routing/navigation

## redirect

```tsx
import {redirect} from '@/i18n/navigation';

return redirect({href: '/login', locale});
redirect({href: '/about', locale: 'en', forcePrefix: true});
```
https://next-intl.dev/docs/routing/navigation

- `locale` is always required. https://next-intl.dev/docs/routing/navigation
- TypeScript does not narrow after `redirect(...)`. Write `return redirect(...)`. https://next-intl.dev/docs/routing/navigation
- `forcePrefix: true` forces a prefix so the cookie updates first. `permanentRedirect` works the same way. https://next-intl.dev/docs/routing/navigation

## getPathname

```ts
getPathname({locale: 'en', href: '/about'});
```
https://next-intl.dev/docs/routing/navigation

- It returns the path WITHOUT `basePath`. Prefix it yourself if you need it. https://next-intl.dev/docs/routing/configuration
- Use it in sitemap `alternates` only for specific needs, such as CMS slugs that depend on the locale. The proxy already sends alternate links in a `link` header. https://next-intl.dev/docs/environments/actions-metadata-route-handlers
