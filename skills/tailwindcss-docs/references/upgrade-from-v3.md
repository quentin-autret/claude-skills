# Upgrading from Tailwind CSS v3 to v4

## Contents

- Before migrating
- Tooling and config
- Removed and renamed utilities
- Changed defaults
- Changed selectors and variants
- Arbitrary value syntax
- Preflight changes

## Before migrating

- v4 needs Safari 16.4+, Chrome 111+, Firefox 128+ (it relies on `@property` and `color-mix()`). Stay on v3.4 for older browsers. https://tailwindcss.com/docs/upgrade-guide#browser-requirements
- Default path: run `npx @tailwindcss/upgrade` (Node.js 20+) in a new branch. It updates dependencies, moves the JS config to CSS and edits templates. Review the diff and test in the browser, complex projects need hand fixes. https://tailwindcss.com/docs/upgrade-guide#using-the-upgrade-tool

## Tooling and config

| v3 | v4 | Source |
| :--- | :--- | :--- |
| PostCSS plugin `tailwindcss`, plus `postcss-import` and `autoprefixer` | `plugins: { "@tailwindcss/postcss": {} }` only | https://tailwindcss.com/docs/upgrade-guide#using-postcss |
| PostCSS in a Vite project | `@tailwindcss/vite` plugin, faster | https://tailwindcss.com/docs/upgrade-guide#using-vite |
| `npx tailwindcss` | `npx @tailwindcss/cli` | https://tailwindcss.com/docs/upgrade-guide#using-tailwind-cli |
| `@tailwind base/components/utilities` | `@import "tailwindcss";` | https://tailwindcss.com/docs/upgrade-guide#removed-tailwind-directives |
| `tailwind.config.js` found automatically | `@config "../../tailwind.config.js";` to load it explicitly (JS configs are no longer detected automatically) | https://tailwindcss.com/docs/upgrade-guide#using-a-javascript-config-file |
| `safelist` | `@source inline("...")` | https://tailwindcss.com/docs/upgrade-guide#using-a-javascript-config-file |
| `corePlugins` | removed, no replacement | https://tailwindcss.com/docs/upgrade-guide#disabling-core-plugins |
| `separator` | not supported | https://tailwindcss.com/docs/upgrade-guide#using-a-javascript-config-file |
| Toggling dark mode manually with a class | `@custom-variant dark (&:where(.dark, .dark *));` | https://tailwindcss.com/docs/dark-mode#toggling-dark-mode-manually |
| `resolveConfig` in JS | CSS variables: `var(--color-blue-500)` or `getComputedStyle(document.documentElement).getPropertyValue("--shadow-xl")` | https://tailwindcss.com/docs/upgrade-guide#theme-values-in-javascript |
| `theme(colors.red.500)` | `var(--color-red-500)`. In media queries keep `theme()` with the variable name: `@media (width >= theme(--breakpoint-xl))` | https://tailwindcss.com/docs/upgrade-guide#using-the-theme-function |
| `container: { center, padding }` | `@utility container { margin-inline: auto; padding-inline: 2rem; }` | https://tailwindcss.com/docs/upgrade-guide#container-configuration |
| v3 prefix option | `@import "tailwindcss" prefix(tw);`, classes `tw:flex tw:hover:bg-red-600` | https://tailwindcss.com/docs/upgrade-guide#using-a-prefix |
| `@layer utilities { .x {} }` | `@utility x { }`, so variants work | https://tailwindcss.com/docs/upgrade-guide#adding-custom-utilities |
| plugin in `plugins: []` | `@plugin "@tailwindcss/typography";` | https://tailwindcss.com/docs/functions-and-directives#plugin-directive |

- With a prefix, `@theme` still declares unprefixed names (`--color-avocado-100`). The emitted variables are prefixed (`--tw-color-avocado-100`). https://tailwindcss.com/docs/upgrade-guide#using-a-prefix
- A `@utility btn { ... }` with many properties can still be overridden by single-property utilities: custom utilities sort by the number of properties. https://tailwindcss.com/docs/upgrade-guide#adding-custom-utilities
- `@apply` in a CSS module or a Vue, Svelte or Astro `<style>` block needs `@reference "../../app.css";` at the top, or use theme variables directly (faster). The docs example writes `var(--text-red-500)`. Colors live in `--color-*`, so write `var(--color-red-500)`. https://tailwindcss.com/docs/upgrade-guide#using-apply-with-vue-svelte-or-css-modules

## Removed and renamed utilities

| v3 | v4 | Source |
| :--- | :--- | :--- |
| `bg-opacity-50`, `text-opacity-*`, `border-opacity-*`, `divide-opacity-*`, `ring-opacity-*`, `placeholder-opacity-*` | opacity modifier `bg-black/50` | https://tailwindcss.com/docs/upgrade-guide#removed-deprecated-utilities |
| `flex-shrink-*`, `flex-grow-*` | `shrink-*`, `grow-*` | https://tailwindcss.com/docs/upgrade-guide#removed-deprecated-utilities |
| `overflow-ellipsis` | `text-ellipsis` | https://tailwindcss.com/docs/upgrade-guide#removed-deprecated-utilities |
| `decoration-slice`, `decoration-clone` | `box-decoration-slice`, `box-decoration-clone` | https://tailwindcss.com/docs/upgrade-guide#removed-deprecated-utilities |
| `shadow-sm`, `shadow` | `shadow-xs`, `shadow-sm` | https://tailwindcss.com/docs/upgrade-guide#updated-shadow-radius-and-blur-scales |
| `drop-shadow-sm`, `drop-shadow` | `drop-shadow-xs`, `drop-shadow-sm` | same |
| `blur-sm`, `blur` | `blur-xs`, `blur-sm` | same |
| `backdrop-blur-sm`, `backdrop-blur` | `backdrop-blur-xs`, `backdrop-blur-sm` | same |
| `rounded-sm`, `rounded` | `rounded-xs`, `rounded-sm` | same |
| `outline-none` (transparent outline) | `outline-hidden` | https://tailwindcss.com/docs/upgrade-guide#renamed-outline-utility |
| `outline outline-2` | `outline-2` (style solid is implied) | https://tailwindcss.com/docs/upgrade-guide#renamed-outline-utility |
| `ring` (3px blue) | `ring-3 ring-blue-500` | https://tailwindcss.com/docs/upgrade-guide#default-ring-width-and-color |
| `bg-gradient-to-r` | `bg-linear-to-r` | https://tailwindcss.com/docs/upgrade-guide#using-variants-with-gradients |
| `!flex` | `flex!` (leading `!` is deprecated) | https://tailwindcss.com/docs/upgrade-guide#the-important-modifier |

- The bare v3 names (`shadow`, `rounded`, `blur`) still work for backward compatibility, but the `<utility>-sm` utilities look different unless updated to their `<utility>-xs` versions. https://tailwindcss.com/docs/upgrade-guide#updated-shadow-radius-and-blur-scales
- New `outline-none` sets `outline-style: none`, so `focus:outline-none` becomes `focus:outline-hidden`. https://tailwindcss.com/docs/upgrade-guide#renamed-outline-utility

## Changed defaults

- Border and divide colors are `currentColor` (v3: `gray-200`). Add a color to every `border` and `divide`. To restore v3 globally: https://tailwindcss.com/docs/upgrade-guide#default-border-color

```css
@layer base {
  *, ::after, ::before, ::backdrop, ::file-selector-button {
    border-color: var(--color-gray-200, currentColor);
  }
}
```

- `ring` width is 1px and its color `currentColor`. Bridge, not idiomatic: `@theme { --default-ring-width: 3px; --default-ring-color: var(--color-blue-500); }`. https://tailwindcss.com/docs/upgrade-guide#default-ring-width-and-color
- `transition` and `transition-colors` include `outline-color`. Set the outline color unconditionally to avoid a color transition on focus: `outline-cyan-500 transition hover:outline-2`. https://tailwindcss.com/docs/upgrade-guide#transitioning-outline-color
- A gradient variant overrides only the stop it sets. Drop a `via` stop in a state with `dark:via-none`. https://tailwindcss.com/docs/upgrade-guide#using-variants-with-gradients
- `rotate-*`, `scale-*`, `translate-*` use the individual CSS properties. Reset them with `scale-none` (not `transform-none`), and transition them with `transition-[opacity,scale]` (not `transition-[opacity,transform]`). https://tailwindcss.com/docs/upgrade-guide#individual-transform-properties

## Changed selectors and variants

- `space-y-4` is now `> :not(:last-child) { margin-bottom }` (v3: `> :not([hidden]) ~ :not([hidden])` with margin-top). Inline children and extra child margins can shift. Migrate to flex or grid with `gap`. https://tailwindcss.com/docs/upgrade-guide#space-between-selector
- `divide-*` uses the same `:not(:last-child)` selector. https://tailwindcss.com/docs/upgrade-guide#divide-selector
- Variants stack left to right (v3: right to left): `first:*:pt-0` becomes `*:first:pt-0`. Mostly affects `*` and typography `prose-headings`. https://tailwindcss.com/docs/upgrade-guide#variant-stacking-order
- `hover` is wrapped in `@media (hover: hover)` and does not fire on tap. Restore v3 with `@custom-variant hover (&:hover);`. https://tailwindcss.com/docs/upgrade-guide#hover-styles-on-mobile

## Arbitrary value syntax

- `bg-[--brand-color]` becomes `bg-(--brand-color)`. https://tailwindcss.com/docs/upgrade-guide#variables-in-arbitrary-values
- Commas in `grid-cols-*`, `grid-rows-*` and `object-*` arbitrary values no longer become spaces: write `grid-cols-[max-content_auto]`. https://tailwindcss.com/docs/upgrade-guide#arbitrary-values-in-grid-and-object-position-utilities

## Preflight changes

- Placeholder color is the text color at 50% (v3: `gray-400`). Restore: `@layer base { input::placeholder, textarea::placeholder { color: var(--color-gray-400); } }`. https://tailwindcss.com/docs/upgrade-guide#new-default-placeholder-color
- Buttons use `cursor: default`. Restore: `@layer base { button:not(:disabled), [role="button"]:not(:disabled) { cursor: pointer; } }`. https://tailwindcss.com/docs/upgrade-guide#buttons-use-the-default-cursor
- `<dialog>` margins are reset. Center again with `@layer base { dialog { margin: auto; } }`. https://tailwindcss.com/docs/upgrade-guide#dialog-margins-removed
- `block` or `flex` no longer override the `hidden` attribute. Remove `hidden` to show the element (`hidden="until-found"` excepted). https://tailwindcss.com/docs/upgrade-guide#hidden-attribute-takes-priority
