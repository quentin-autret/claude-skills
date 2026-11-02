# Installation and framework setup

## Contents

- Choosing the integration
- Vite, PostCSS, CLI, Play CDN
- Framework specifics
- Browser support
- Preprocessors, CSS modules and component style blocks
- Editor tooling
- Old patterns

## Choosing the integration

- Most seamless for Vite-based frameworks: the Vite plugin (Laravel, SvelteKit, React Router, Nuxt, SolidJS and others). https://tailwindcss.com/docs/installation/using-vite
- Next.js and Angular: the PostCSS plugin (the Gatsby, Parcel, Meteor, Symfony, Rspack and Laravel Mix guides also use it). https://tailwindcss.com/docs/installation/using-postcss
- No build tool: the Tailwind CLI, also shipped as a standalone executable that needs no Node.js. https://tailwindcss.com/docs/installation/tailwind-cli
- An unlisted framework falls back to the CLI, the Vite plugin or the PostCSS plugin. https://tailwindcss.com/docs/installation/framework-guides
- Most setups are: install, register the plugin, `@import "tailwindcss";` in one CSS file (Ruby on Rails uses the `tailwindcss-rails` gem and Phoenix the `tailwind` Hex package instead of an npm plugin). JavaScript config files are no longer detected automatically in v4 and class detection is automatic, with `@source` to add paths. https://tailwindcss.com/docs/upgrade-guide#using-a-javascript-config-file and https://tailwindcss.com/docs/detecting-classes-in-source-files
- Packages: core `tailwindcss` plus one integration, `@tailwindcss/vite`, `@tailwindcss/postcss`, `@tailwindcss/cli` or `@tailwindcss/browser`. https://tailwindcss.com/docs/installation/using-postcss and https://tailwindcss.com/docs/installation/play-cdn

## Vite, PostCSS, CLI, Play CDN

Vite:

```ts
// vite.config.ts
import { defineConfig } from 'vite'
import tailwindcss from '@tailwindcss/vite'
export default defineConfig({ plugins: [tailwindcss() /* then the framework plugin */] })
```

Every Vite guide that shows a framework plugin places `tailwindcss()` before it (`solidPlugin()`, `sveltekit()`, `reactRouter()`, `tanstackStart()`, `qwikCity()`, `adonisjs()`). https://tailwindcss.com/docs/installation/framework-guides/sveltekit

PostCSS:

```shell
npm install tailwindcss @tailwindcss/postcss postcss
```

```js
// postcss.config.mjs
export default { plugins: { "@tailwindcss/postcss": {} } }
```

`@tailwindcss/postcss` is the only plugin needed: in v4 imports and vendor prefixing are handled automatically, so `postcss-import` and `autoprefixer` can be removed. https://tailwindcss.com/docs/upgrade-guide#using-postcss

CLI:

```shell
npm install tailwindcss @tailwindcss/cli
npx @tailwindcss/cli -i ./src/input.css -o ./src/output.css --watch
```

https://tailwindcss.com/docs/installation/tailwind-cli

Play CDN, development only, not for production:

```html
<script src="https://cdn.jsdelivr.net/npm/@tailwindcss/browser@4"></script>
<style type="text/tailwindcss">
  @theme { --color-clifford: #da373d; }
</style>
```

https://tailwindcss.com/docs/installation/play-cdn

## Framework specifics

Only what differs from the default setup.

| Framework | Setup detail | Source |
| :--- | :--- | :--- |
| Next.js | PostCSS plugin in `postcss.config.mjs`, import in `app/globals.css` | https://tailwindcss.com/docs/installation/framework-guides/nextjs |
| Angular | `npm install tailwindcss @tailwindcss/postcss postcss --force` (Angular declares a peer dependency on Tailwind v3). Config file is `.postcssrc.json`: `{ "plugins": { "@tailwindcss/postcss": {} } }`. Import in `src/styles.css` | https://tailwindcss.com/docs/installation/framework-guides/angular |
| SvelteKit | `src/app.css` imported from `src/routes/+layout.svelte`. A `<style>` block that must be processed by Tailwind starts with `@reference "tailwindcss";` | https://tailwindcss.com/docs/installation/framework-guides/sveltekit |
| React Router | Vite plugin before `reactRouter()` and `tsconfigPaths()`. Import in `app/app.css` | https://tailwindcss.com/docs/installation/framework-guides/react-router |
| TanStack Start | Import `src/styles.css` in the root route: in `src/routes/__root.tsx`, `import appCss from '../styles.css?url'` and `head: () => ({ links: [{ rel: 'stylesheet', href: appCss }] })` | https://tailwindcss.com/docs/installation/framework-guides/tanstack-start |
| Nuxt | Vite plugin through `nuxt.config.ts`: `vite: { plugins: [tailwindcss()] }`, plus `css: ['~/assets/css/main.css']` (file in `app/assets/css/`) | https://tailwindcss.com/docs/installation/framework-guides/nuxt |
| Astro | `vite: { plugins: [tailwindcss()] }` in `astro.config.mjs`. Create `src/styles/global.css` and import `../styles/global.css` in the frontmatter of the page (`index.astro`) | https://tailwindcss.com/docs/installation/framework-guides/astro |
| Ember.js | `vite.config.mjs`. In `index.html`, replace the `/@embroider/virtual/app.css` link by `/app/styles/app.css` so Vite processes it, keep `vendor.css` | https://tailwindcss.com/docs/installation/framework-guides/emberjs |
| AdonisJS | Add `@source "../views";` after the import in `resources/css/app.css` | https://tailwindcss.com/docs/installation/framework-guides/adonisjs |
| Laravel (Vite) | `resources/css/app.css` lists the Blade sources, paths relative to the stylesheet (below, per the source-detection page) | https://tailwindcss.com/docs/installation/framework-guides/laravel/vite and https://tailwindcss.com/docs/detecting-classes-in-source-files |
| Laravel (Mix) | In `webpack.mix.js`, `mix.js(...).postCss("resources/css/app.css", "public/css", [require("@tailwindcss/postcss")])`, same CSS, dev command `npm run watch` | https://tailwindcss.com/docs/installation/framework-guides/laravel/mix |
| Gatsby | Add `gatsby-plugin-postcss`, CommonJS `postcss.config.js`, import the CSS from `gatsby-browser.js` | https://tailwindcss.com/docs/installation/framework-guides/gatsby |
| Parcel | `.postcssrc` (JSON) at the project root | https://tailwindcss.com/docs/installation/framework-guides/parcel |
| Meteor | Also install `postcss-load-config`. Import in `client/main.css` | https://tailwindcss.com/docs/installation/framework-guides/meteor |
| Symfony | Webpack Encore instead of AssetMapper, `Encore.enablePostCssLoader()`. Add `@source not "../../public";` or watch mode recompiles in a loop | https://tailwindcss.com/docs/installation/framework-guides/symfony |
| Rspack | Rule `{ test: /\.css$/, use: ["postcss-loader"], type: "css" }` | https://tailwindcss.com/docs/installation/framework-guides/rspack |
| Ruby on Rails | `bundle add tailwindcss-rails`, `./bin/rails tailwindcss:install`, run `./bin/dev` | https://tailwindcss.com/docs/installation/framework-guides/ruby-on-rails |
| Phoenix | Hex package `{:tailwind, "~> 0.3", runtime: Mix.env() == :dev}`, pin a v4 binary with `config :tailwind, version: "4.1.10"`. Remove `import "../css/app.css"` from `assets/js/app.js` | https://tailwindcss.com/docs/installation/framework-guides/phoenix |

Laravel sources:

```css
@import "tailwindcss";
@source "../../vendor/laravel/framework/src/Illuminate/Pagination/resources/views/*.blade.php";
@source "../../storage/framework/views/*.php";
@source "../**/*.blade.php";
@source "../**/*.js";
```

https://tailwindcss.com/docs/installation/framework-guides/laravel/vite

## Browser support

- v4 targets Chrome 111, Safari 16.4 and Firefox 128. https://tailwindcss.com/docs/compatibility#browser-support
- `field-sizing: content`, `@starting-style` and `text-wrap: balance` have limited support. Skip them when target browsers lack it. https://tailwindcss.com/docs/compatibility#browser-support

## Preprocessors, CSS modules and component style blocks

- Do not use Sass, Less or Stylus. Tailwind bundles `@import`ed CSS itself, Lightning CSS flattens native nesting, and `color-mix()`, `min()`, `max()`, `round()` replace preprocessor functions. https://tailwindcss.com/docs/compatibility#sass-less-and-stylus
- Avoid CSS modules and Vue, Svelte or Astro `<style>` blocks: each one is processed separately (50 modules run Tailwind 50 times) and sees no `@theme`. https://tailwindcss.com/docs/compatibility#performance
- When a module needs `@apply`, give it context:

```css
/* Button.module.css */
@reference "../app.css";
button { @apply bg-blue-500; }
```

https://tailwindcss.com/docs/compatibility#explicit-context-sharing

- Alternative that improves build performance: use the variable directly instead of `@apply`, `background: var(--color-blue-500);`, which lets Tailwind skip the file. https://tailwindcss.com/docs/compatibility#explicit-context-sharing

## Editor tooling

- `@theme`, `@variant` and `@source` can raise "unknown at-rule" warnings. The Tailwind CSS IntelliSense extension for VS Code adds a Tailwind CSS language mode. A strict editor may need native CSS validation turned off. https://tailwindcss.com/docs/editor-setup#syntax-support
- `prettier-plugin-tailwindcss` sorts classes in the recommended order. https://tailwindcss.com/docs/editor-setup#class-sorting-with-prettier

## Old patterns

| v3 | v4 |
| :--- | :--- |
| `@tailwind base; @tailwind components; @tailwind utilities;` | `@import "tailwindcss";` https://tailwindcss.com/docs/upgrade-guide#removed-tailwind-directives |
| PostCSS plugin `tailwindcss` with `autoprefixer` | `@tailwindcss/postcss` alone https://tailwindcss.com/docs/upgrade-guide#using-postcss |
| `npx tailwindcss -i ... -o ...` | `npx @tailwindcss/cli -i ... -o ...` https://tailwindcss.com/docs/upgrade-guide#using-tailwind-cli |
| `tailwind.config.js` with `content` globs | automatic detection plus `@source` https://tailwindcss.com/docs/detecting-classes-in-source-files |
