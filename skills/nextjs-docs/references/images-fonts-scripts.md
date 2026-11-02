# Images, fonts, scripts and lazy loading

`next/image` and its version 16 defaults, `next/font`, `next/script`, `@next/third-parties`, `next/dynamic` and video in Next.js 16.4.4.

## Contents

- `next/image` sizing
- LCP and loading
- Remote and local patterns
- Version 16 image defaults
- Other image pitfalls
- Fonts
- Scripts
- Third-party libraries
- Lazy loading

## `next/image` sizing

- `width` and `height` are the intrinsic size, used for aspect ratio. CSS sets the rendered size. Both are required unless the image is a static import or uses `fill`.
  https://nextjs.org/docs/app/api-reference/components/image#width-and-height
- Unknown dimensions: `fill` with a positioned parent (`relative`, `fixed` or `absolute`).
  https://nextjs.org/docs/app/api-reference/components/image#fill
- Always pass `sizes` with `fill` or CSS-responsive images. Without it the browser assumes `100vw` and downloads oversized files.
  ```tsx
  <Image src={src} alt="" fill sizes="(max-width: 768px) 100vw, (max-width: 1200px) 50vw, 33vw" />
  ```
  https://nextjs.org/docs/app/api-reference/components/image#sizes
- A custom width in `style` needs `height: 'auto'` to keep the ratio.
  https://nextjs.org/docs/app/api-reference/components/image#style
- Static imports provide `width`, `height` and `blurDataURL`. Remote images need `width`, `height` and optional `blurDataURL` by hand.
  https://nextjs.org/docs/app/getting-started/images#remote-images
- Keep `blurDataURL` tiny (10px or less): it is enlarged and blurred.
  https://nextjs.org/docs/app/api-reference/components/image#blurdataurl

## LCP and loading

- `priority` is deprecated. Use `preload` only when one image is certainly the LCP element, otherwise `loading="eager"` or `fetchPriority="high"`. Never combine `preload` with either.
  https://nextjs.org/docs/app/api-reference/components/image#preload
- Light and dark variants: 2 `<Image>` toggled by CSS. Keep them lazy and use `fetchPriority="high"`, never `preload` (both would load).
  https://nextjs.org/docs/app/api-reference/components/image#theme-detection
- Function props such as `onLoad`, `onError` and `loader` need a Client Component. `onLoadingComplete` is deprecated (since 14) in favor of `onLoad`.
  https://nextjs.org/docs/app/api-reference/components/image#onloadingcomplete

## Remote and local patterns

- `remotePatterns` as narrow as possible. Omitted fields mean `**`. Non-matching URLs return 400. `images.domains` is deprecated.
  ```js
  images: { remotePatterns: [new URL('https://example.com/account123/**')] }
  ```
  https://nextjs.org/docs/app/api-reference/components/image#remotepatterns
- `*` matches 1 segment or subdomain, `**` any number, only at the end of a pathname or the start of a hostname.
  https://nextjs.org/docs/app/api-reference/components/image#remotepatterns
- Omitting `search` allows any query. Pin it with `search: ''` or an exact value.
  https://nextjs.org/docs/app/api-reference/components/image#localpatterns
- Local images with a query string need `localPatterns: [{ pathname: '/assets/**', search: '?v=1' }]`.
  https://nextjs.org/docs/app/guides/upgrading/version-16#local-images-with-query-strings-breaking-change

## Version 16 image defaults

- `qualities` defaults to `[75]`. Another `quality` prop is coerced to the closest allowed value, and a direct `/_next/image?q=` with another value returns 400. List every quality you use.
  https://nextjs.org/docs/app/api-reference/components/image#qualities
- `minimumCacheTTL` defaults to 4 hours. There is no invalidation: change the `src` or delete `<distDir>/cache/images`.
  https://nextjs.org/docs/app/api-reference/components/image#minimumcachettl
- `imageSizes` no longer contains `16`.
  https://nextjs.org/docs/app/guides/upgrading/version-16#imagesizes-default-breaking-change
- The loader follows up to 3 redirects by default (`maximumRedirects`), without re-checking `remotePatterns`.
  https://nextjs.org/docs/app/api-reference/components/image#maximumredirects
- Optimizing images from local IPs is blocked. `dangerouslyAllowLocalIP: true` only on private networks (SSRF).
  https://nextjs.org/docs/app/api-reference/components/image#dangerouslyallowlocalip

## Other image pitfalls

- The default loader never forwards headers: authenticated sources need `unoptimized`.
  https://nextjs.org/docs/app/api-reference/components/image#src
- `.svg` sources skip optimization automatically. With `dangerouslyAllowSVG`, also set `contentDispositionType: 'attachment'` and a `contentSecurityPolicy` (strongly recommended).
  https://nextjs.org/docs/app/api-reference/components/image#dangerouslyallowsvg
- Behind a proxy or CDN, configure it to forward the `Accept` header, which Next.js uses to pick the output format. AVIF takes about 50% longer to encode (and compresses 20% smaller than WebP).
  https://nextjs.org/docs/app/api-reference/components/image#formats
- `getImageProps()` serves `<picture>` and CSS backgrounds. It cannot be used with `placeholder`.
  https://nextjs.org/docs/app/api-reference/components/image#getimageprops
- With `basePath`, `src` must include the base path by hand.
  https://nextjs.org/docs/app/api-reference/config/next-config-js/basePath#images

## Fonts

- Define each font once in a shared file (`app/fonts.ts`) and import it: every call creates a hosted instance. A root layout font applies and preloads app-wide.
  https://nextjs.org/docs/app/api-reference/components/font#using-a-font-definitions-file
- Prefer variable fonts. A non-variable font needs `weight`. Multi-word names use underscores: `Roboto_Mono`.
  https://nextjs.org/docs/app/api-reference/components/font#weight
- Set `subsets`: omitting it with `preload: true` warns.
  https://nextjs.org/docs/app/api-reference/components/font#specifying-a-subset
- Tailwind v4: `variable: '--font-inter'`, add `inter.variable` to `<html>`, then `@theme inline { --font-sans: var(--font-inter); }`.
  https://nextjs.org/docs/app/api-reference/components/font#with-tailwind-css
- A local `src` path is relative to the file that calls `localFont`.
  https://nextjs.org/docs/app/getting-started/fonts#local-fonts

## Scripts

- Default strategy `afterInteractive` (tag managers). `lazyOnload` for chat widgets. `beforeInteractive` only in a root layout, for cookie consent managers or bot detectors.
  https://nextjs.org/docs/app/api-reference/components/script#strategy
- `beforeInteractive` runs once per document, never again on client navigation.
  https://nextjs.org/docs/app/api-reference/components/script#beforeinteractive
- `worker` does not work in the App Router.
  https://nextjs.org/docs/app/api-reference/components/script#worker
- Inline scripts need an `id`. `onLoad`, `onReady` and `onError` need a Client Component. `onLoad` and `onError` cannot be used with `beforeInteractive`.
  https://nextjs.org/docs/app/guides/scripts#inline-scripts
- A script loads once per layout across navigations. Include it only where needed.
  https://nextjs.org/docs/app/guides/scripts#layout-scripts

## Third-party libraries

- `GoogleAnalytics` from `@next/third-parties` tracks pageviews on history changes. Manual pageview events require disabling the default measurement, or data is duplicated. With GTM already present, the page recommends configuring GA through GTM.
  https://nextjs.org/docs/app/guides/third-party-libraries#google-analytics
- `sendGTMEvent` needs `GoogleTagManager` included in a parent layout, page or component, or in the same file.
  https://nextjs.org/docs/app/guides/third-party-libraries#google-tag-manager

## Lazy loading

- `next/dynamic` applies to Client Components. Server Components are code-split already.
  https://nextjs.org/docs/app/guides/lazy-loading
- `ssr: false` is only allowed inside Client Components and errors in Server Components.
  https://nextjs.org/docs/app/guides/lazy-loading#skipping-ssr
- Load a library on demand in a handler: `const Fuse = (await import('fuse.js')).default`.
  https://nextjs.org/docs/app/guides/lazy-loading#loading-external-libraries
- Video: with `autoPlay`, also add `muted` (so it plays in most browsers) and `playsInline` (iOS compatibility).
  https://nextjs.org/docs/app/guides/videos#common-video-tag-attributes
- Set `width` and `height` or a CSS `aspect-ratio` on `<video>` and `<iframe>` so they do not collapse and shift the layout.
  https://nextjs.org/docs/app/guides/videos#video-best-practices
