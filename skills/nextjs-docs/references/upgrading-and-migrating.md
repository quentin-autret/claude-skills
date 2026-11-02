# Upgrading to Next.js 16 and migrating

Upgrade commands, codemods, breaking changes of versions 15 and 16, the Pages Router to App Router path, and migration from Vite or Create React App, for Next.js 16.4.4.

## Contents

- Running the upgrade
- Version 16 requirements
- Async request APIs
- Caching changes
- Middleware renamed to Proxy
- Turbopack by default
- Removals in version 16
- Version 15 changes still met in old code
- Pages Router to App Router
- From Vite or Create React App

## Running the upgrade

- `npx next upgrade` (from 16.1). Older versions: `npx @next/codemod@canary upgrade latest`. Also bump `@types/react` and `@types/react-dom`.
  https://nextjs.org/docs/app/getting-started/upgrading#latest-version
- The `upgrade` codemod does not run every migration. If sync `params`, `searchParams`, `cookies()`, `headers()` or `draftMode()` remain, run `npx @next/codemod@canary next-async-request-api .`.
  https://nextjs.org/docs/app/guides/upgrading/version-16#using-the-codemod
- Without a TTY (CI), the codemod accepts every default, Turbopack and React 19 codemods included.
  https://nextjs.org/docs/app/guides/upgrading/codemods#upgrade
- A wrong path (`./app` in a `src/` project) reports `0 ok` instead of failing. Check the file count.
  https://nextjs.org/docs/app/guides/upgrading/codemods
- `@next-codemod-error` comments and `UnsafeUnwrapped*` casts mark spots the codemod could not migrate. Complete the migration before removing them.
  https://nextjs.org/docs/app/guides/upgrading/codemods#next-async-request-api

## Version 16 requirements

- Node.js 20.9+, TypeScript 5.1+, Chrome, Edge and Firefox 111+, Safari 16.4+.
  https://nextjs.org/docs/app/guides/upgrading/version-16#nodejs-runtime-and-browser-support
- Every parallel route slot needs an explicit `default.js`, or the build fails.
  https://nextjs.org/docs/app/guides/upgrading/version-16#parallel-routes-defaultjs-requirement
- Next.js no longer overrides `scroll-behavior: smooth` during navigation. Opt back in with `<html data-scroll-behavior="smooth">`.
  https://nextjs.org/docs/app/guides/upgrading/version-16#scroll-behavior-override
- `next/image` defaults changed: `qualities` `[75]`, `minimumCacheTTL` 4 hours, no `16` in `imageSizes`, 3 redirects max, local IPs blocked.
  https://nextjs.org/docs/app/guides/upgrading/version-16#qualities-default-breaking-change

## Async request APIs

- Version 16 removes sync access to `cookies`, `headers`, `draftMode`, `params` and `searchParams`. Await them.
  ```tsx
  export default async function Page(props: PageProps<'/blog/[slug]'>) {
    const { slug } = await props.params
    const query = await props.searchParams
  }
  ```
  https://nextjs.org/docs/app/guides/upgrading/version-16#async-request-apis-breaking-change
- In a sync Client Component, unwrap with `use(props.params)`.
  https://nextjs.org/docs/app/guides/upgrading/version-15#params--searchparams
- Image functions (`opengraph-image`, `icon`) receive `params` and `id` as Promises. `sitemap({ id })` gets `Promise<string>`: `Number(await id)`.
  https://nextjs.org/docs/app/guides/upgrading/version-16#async-id-parameter-for-sitemap-breaking-change
- Generate `PageProps`, `LayoutProps` and `RouteContext` with `npx next typegen`.
  https://nextjs.org/docs/app/guides/upgrading/version-16#migrating-types-for-async-request-time-apis

## Caching changes

- `revalidateTag('posts')` alone is deprecated and a TS error. Write `revalidateTag('posts', 'max')`, or `updateTag('posts')` in a Server Action.
  https://nextjs.org/docs/app/guides/upgrading/version-16#revalidatetag
- `unstable_cacheLife` and `unstable_cacheTag` become `cacheLife` and `cacheTag` (codemod `remove-unstable-prefix`).
  https://nextjs.org/docs/app/guides/upgrading/version-16#cachelife-and-cachetag
- `experimental.dynamicIO` and `experimental.useCache` are removed. Use `cacheComponents: true` only if you actively used them, otherwise just delete the flags: it is not a rename and can surface build errors for uncached data outside Suspense. With `experimental.ppr` today, stay on your current Next.js 15 canary.
  https://nextjs.org/docs/app/guides/upgrading/version-16#experimentaldynamicio-and-experimentalusecache
- To adopt Cache Components gradually, the `cache-components-instant-false` codemod adds `instant = false` everywhere. Remove it route by route.
  https://nextjs.org/docs/app/guides/upgrading/codemods

## Middleware renamed to Proxy

- Rename `middleware.ts` to `proxy.ts` and the export to `proxy`. Codemod: `npx @next/codemod@canary middleware-to-proxy .`. `skipMiddlewareUrlNormalize` becomes `skipProxyUrlNormalize`.
  https://nextjs.org/docs/app/guides/upgrading/version-16#middleware-to-proxy
- Proxy runs on Node.js only. Middleware that needs `edge` stays `middleware` for now: do not rename it blindly.
  https://nextjs.org/docs/app/guides/upgrading/version-16#middleware-to-proxy

## Turbopack by default

- Remove `--turbopack` and `--turbo` from scripts. Move `experimental.turbopack` to top-level `turbopack`.
  https://nextjs.org/docs/app/guides/upgrading/version-16#turbopack-by-default
- `next build` fails if a custom `webpack` config exists (a plugin can add one silently). Migrate it, or build with `--webpack`.
  https://nextjs.org/docs/app/guides/upgrading/version-16#turbopack-by-default

## Removals in version 16

- `next lint` is gone and `next build` no longer lints. Codemod `next-lint-to-eslint-cli` writes `eslint.config.mjs`. `eslint-config-next@16` needs ESLint 9+.
  https://nextjs.org/docs/app/guides/upgrading/version-16#next-lint-command
- `serverRuntimeConfig`, `publicRuntimeConfig` and `next/config` are removed. Use env vars, read after `await connection()` for runtime values.
  https://nextjs.org/docs/app/guides/upgrading/version-16#runtime-configuration
- AMP is removed. `unstable_rootParams` becomes `next/root-params`. `next/legacy/image` and `images.domains` are deprecated.
  https://nextjs.org/docs/app/guides/upgrading/version-16#amp-support
- `reactCompiler` is stable but not on by default. Do not enable it just to finish the upgrade.
  https://nextjs.org/docs/app/guides/upgrading/version-16#react-compiler-support

## Version 15 changes still met in old code

- `useFormState` became `useActionState` (React 19).
  https://nextjs.org/docs/app/guides/upgrading/version-15#react-19
- `fetch` and Route Handler `GET` are no longer cached by default.
  https://nextjs.org/docs/app/guides/upgrading/version-15#fetch-requests
  https://nextjs.org/docs/app/guides/upgrading/version-15#route-handlers
- `serverComponentsExternalPackages` became `serverExternalPackages`. `NextRequest.geo` and `.ip` are removed.
  https://nextjs.org/docs/app/guides/upgrading/version-15#nextrequest-geolocation

## Pages Router to App Router

- `app/` and `pages/` coexist: migrate page by page. Navigation between the routers is a hard navigation.
  https://nextjs.org/docs/app/guides/migrating/app-router-migration#using-app-router-together-with-pages-router
- Easiest page move: put the old component in a `'use client'` file, import it in a Server Component `page.js`, move data fetching there.
  https://nextjs.org/docs/app/guides/migrating/app-router-migration#step-4-migrating-pages
- Mapping: `getStaticPaths` to `generateStaticParams`, `next/head` to `metadata`, `pages/api` to `route.ts`, `_error` to `error.js`, `404.js` to `not-found.js`.
  https://nextjs.org/docs/app/guides/migrating/app-router-migration#migrating-from-pages-to-app
  https://nextjs.org/docs/app/guides/migrating/app-router-migration#step-3-migrating-nexthead
  https://nextjs.org/docs/app/guides/migrating/app-router-migration#step-6-migrating-data-fetching-methods
- That guide still says `fetch` defaults to `force-cache`. Version 15 changed it to uncached: trust the upgrade guide.
  https://nextjs.org/docs/app/guides/upgrading/version-15#fetch-requests
- Hooks come from `next/navigation`. The new `useRouter` has no `pathname`, `query` or `asPath`. Shared components use `next/compat/router`.
  https://nextjs.org/docs/app/guides/migrating/app-router-migration#step-5-migrating-routing-hooks

## From Vite or Create React App

- First run the app as a client-only SPA on Next.js (`output: 'export'`), then adopt features. Static export has no server features (SSR, APIs).
  https://nextjs.org/docs/app/guides/migrating/from-create-react-app#step-2-create-the-nextjs-configuration-file
- Entry point: `app/[[...slug]]/page.tsx` with `generateStaticParams() { return [{ slug: [''] }] }` and a client wrapper using `dynamic(() => import('../../App'), { ssr: false })`.
  https://nextjs.org/docs/app/guides/migrating/from-vite#step-5-create-the-entrypoint-page
- Rename `VITE_` variables to `NEXT_PUBLIC_`.
  https://nextjs.org/docs/app/guides/migrating/from-vite#step-7-migrate-the-environment-variables
- From Create React App: rename `REACT_APP_` variables to `NEXT_PUBLIC_`.
  https://nextjs.org/docs/app/guides/migrating/from-create-react-app#step-9-migrate-environment-variables
- Static image imports return an object: use `logo.src`.
  https://nextjs.org/docs/app/guides/migrating/from-create-react-app#step-8-update-static-image-imports
