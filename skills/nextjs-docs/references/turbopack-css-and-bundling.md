# Turbopack, CSS and bundling

Turbopack configuration and limits, CSS ordering, Sass, CSS-in-JS, package bundling options, MDX and avoiding a flash before hydration in Next.js 16.4.4.

## Contents

- Turbopack defaults
- Turbopack configuration
- Turbopack limits
- CSS
- CSS-in-JS
- Package bundling
- MDX
- Preventing a flash before hydration

## Turbopack defaults

- Turbopack is the default for `next dev` and `next build`. Opt out with `--webpack`.
  https://nextjs.org/docs/app/api-reference/turbopack#getting-started
- The `webpack()` function in `next.config` is ignored by Turbopack, and webpack plugins are unsupported (loaders work).
  https://nextjs.org/docs/app/api-reference/turbopack#unsupported-and-unplanned-features
- CSS, PostCSS, Sass and modern JS are built in: add no css-loader, postcss-loader or babel-loader.
  https://nextjs.org/docs/app/api-reference/config/next-config-js/turbopack
- A Babel config file is used automatically. Turbopack does not type-check: run `tsc` or rely on the IDE.
  https://nextjs.org/docs/app/api-reference/turbopack#language-features
- The filesystem cache is on for dev (`.next/dev/cache/turbopack`) and build (`.next/cache/turbopack`). In CI or containers restore `.next/cache`, or set `experimental.turbopackFileSystemCacheForBuild: false`.
  https://nextjs.org/docs/app/api-reference/config/next-config-js/turbopackFileSystemCache#build-environments

## Turbopack configuration

- Config lives under top-level `turbopack`, no longer `experimental.turbo`.
  https://nextjs.org/docs/app/api-reference/config/next-config-js/turbopack
- Loaders:
  ```js
  turbopack: { rules: { '*.svg': { loaders: ['@svgr/webpack'], as: '*.js' } } }
  ```
  https://nextjs.org/docs/app/api-reference/config/next-config-js/turbopack#configuring-webpack-loaders
- Restrict loaders with `condition: { not: 'foreign' }` to skip `node_modules`. A module `type` (for example `type: 'asset'`) can be set without any loader.
  https://nextjs.org/docs/app/api-reference/config/next-config-js/turbopack#advanced-webpack-loader-conditions
- `resolveAlias: { fs: { browser: './empty.ts' } }` replaces webpack `resolve.fallback`. Only the `browser` condition is supported.
  https://nextjs.org/docs/app/guides/upgrading/version-16#resolve-alias-fallback
- `resolveExtensions` overwrites the defaults: list every extension you need.
  https://nextjs.org/docs/app/api-reference/config/next-config-js/turbopack#resolving-custom-extensions
- `?raw` imports need a rule: `'*.txt': { condition: { query: '?raw' }, type: 'text' }`.
  https://nextjs.org/docs/app/api-reference/turbopack#query-strings

## Turbopack limits

- Files outside the project root are not resolved. The root is detected from the lockfile. In a monorepo set `turbopack.root` to an absolute path, never as broad as the home directory.
  ```js
  module.exports = { turbopack: { root: path.join(__dirname, '..') } }
  ```
  https://nextjs.org/docs/app/api-reference/config/next-config-js/turbopack#root-directory
- Loaders get a subset of the loader API, must return JavaScript, and take only plain JSON-like options.
  https://nextjs.org/docs/app/api-reference/config/next-config-js/turbopack
- The Sass `~` prefix is unsupported: write `@import 'bootstrap/...'`, or alias `resolveAlias: { '~*': '*' }`.
  https://nextjs.org/docs/app/api-reference/turbopack#sass-node_modules-imports
- Legacy CSS Modules features are unsupported: standalone `:local` and `:global`, `@value`, ICSS `:import` and `:export`. A plain `.css` composed from a module is treated as global.
  https://nextjs.org/docs/app/api-reference/turbopack#unsupported-and-unplanned-features
- CSS Module order follows JS import order. Lightning CSS keeps 5 decimals, which can shift rendering slightly.
  https://nextjs.org/docs/app/api-reference/turbopack#css-module-ordering
- Unsupported: Yarn PnP, `experimental.urlImports`, `sassOptions.functions`.
  https://nextjs.org/docs/app/api-reference/turbopack

## CSS

- Default: Tailwind for most styling, CSS Modules for component styles, global CSS only for truly global rules.
  https://nextjs.org/docs/app/getting-started/css#recommendations
- Tailwind v4: `@tailwindcss/postcss` in `postcss.config.mjs`, `@import 'tailwindcss';` in `app/globals.css`, imported in the root layout.
  https://nextjs.org/docs/app/getting-started/css#tailwind-css
- CSS order follows import order: turn off import-sorting lint rules, and verify the final order with `next build` because dev can differ.
  https://nextjs.org/docs/app/getting-started/css#development-vs-production
- Global CSS is not removed on navigation, so it can conflict across routes.
  https://nextjs.org/docs/app/getting-started/css#global-css
- `experimental.cssChunking` defaults to `true`. If `b.css` depends on `a.css` and styles break (webpack), use `'strict'`.
  https://nextjs.org/docs/app/api-reference/config/next-config-js/cssChunking#choosing-a-strategy
- `experimental.inlineCss` suits atomic CSS and first-time visitors, not large shared bundles. It works in production builds only.
  https://nextjs.org/docs/app/api-reference/config/next-config-js/inlineCss#trade-offs

## CSS-in-JS

- Works only in Client Components, with a library that supports concurrent React.
  https://nextjs.org/docs/app/guides/css-in-js
- Setup: a style registry, `useServerInsertedHTML` from `next/navigation`, and a Client Component registry around `children` in the root layout. `styled-components` needs v6 or newer and `compiler: { styledComponents: true }`.
  https://nextjs.org/docs/app/guides/css-in-js#styled-components

## Package bundling

- Analyze with `npx next analyze`. Add `--output` to diff before and after.
  https://nextjs.org/docs/app/guides/package-bundling#nextjs-bundle-analyzer
- Move heavy render-only work (syntax highlighting, markdown) into Server Components so the library never ships.
  https://nextjs.org/docs/app/guides/package-bundling#heavy-client-workloads
- Server Components bundle their dependencies. Opt a Node-specific package out with `serverExternalPackages`. `@prisma/client`, `sharp` and `pg` are external already.
  https://nextjs.org/docs/app/api-reference/config/next-config-js/serverExternalPackages
- A dependency shipping raw TS or JSX needs `transpilePackages` (names only). Turbopack transpiles monorepo workspace packages automatically (webpack does so for the App Router only).
  https://nextjs.org/docs/app/api-reference/config/next-config-js/transpilePackages#when-you-need-it
- A package in both `transpilePackages` and `serverExternalPackages` makes the build throw.
  https://nextjs.org/docs/app/api-reference/config/next-config-js/transpilePackages
- `experimental.optimizePackageImports` already covers `lucide-react`, `date-fns`, `lodash-es`, `@mui/material`, `@mui/icons-material` and `react-icons/*`.
  https://nextjs.org/docs/app/api-reference/config/next-config-js/optimizePackageImports

## MDX

- `@next/mdx` with the App Router requires `mdx-components.tsx` at the root, exporting `useMDXComponents()` with no argument.
  https://nextjs.org/docs/app/api-reference/file-conventions/mdx-components
- With Turbopack, pass remark and rehype plugins as strings or `[name, options]` tuples. Plugins with function options cannot be used.
  ```js
  remarkPlugins: ['remark-gfm', ['remark-toc', { heading: 'The Table' }]]
  ```
  https://nextjs.org/docs/app/guides/mdx#using-plugins-with-turbopack
- Frontmatter is not supported by default.
  https://nextjs.org/docs/app/guides/mdx#frontmatter
- The remark and rehype ecosystem is ESM only, so use `next.config.mjs` or `.ts`.
  https://nextjs.org/docs/app/guides/mdx#remark-and-rehype-plugins

## Preventing a flash before hydration

- Set theme or locale text with a synchronous inline `<script>` in `<head>`, plus `suppressHydrationWarning` on `<html>`. `useEffect` flashes after hydration, and `useLayoutEffect` still flashes before React hydrates.
  https://nextjs.org/docs/app/guides/preventing-flash-before-hydration#why-not-useeffect
- Keep the theme in a cookie but read it in the inline script, not with `cookies()` in the root layout, so the page stays static.
  https://nextjs.org/docs/app/guides/preventing-flash-before-hydration#storing-the-theme-in-a-cookie
- Inline scripts inserted by soft navigation do not execute. Use a Client Component helper.
  https://nextjs.org/docs/app/guides/preventing-flash-before-hydration#extracting-a-reusable-component
