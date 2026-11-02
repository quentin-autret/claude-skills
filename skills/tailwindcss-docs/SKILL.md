---
name: tailwindcss-docs
description: Teaches current Tailwind CSS v4 practices drawn from the official documentation. Use for any task that touches Tailwind CSS v4, even a small one, because many v3 habits silently break in v4. Use when writing or reviewing Tailwind CSS v4 code (utility classes, variants, arbitrary values, theme variables in @theme, @utility, @custom-variant, @apply, @reference, @source), setting up Tailwind in a framework or build tool (Vite plugin, PostCSS plugin, CLI, Next.js, SvelteKit, Nuxt, Angular, Laravel, Rails), customizing the theme (colors, fonts, breakpoints, spacing, animations), building dark mode with a class or data attribute, or migrating a project from Tailwind v3 (tailwind.config.js, @tailwind directives, darkMode class, safelist, bg-opacity, renamed shadow and outline utilities). Triggers on tailwindcss, @import "tailwindcss", @tailwindcss/vite, @tailwindcss/postcss, @tailwindcss/cli, @theme, @custom-variant dark, npx @tailwindcss/upgrade, upgrade to Tailwind v4.
metadata:
  version: "Tailwind CSS v4.x (docs built against tailwindcss 4.3.3)"
  source: https://tailwindcss.com/docs
  fetched: "2026-11-02"
---

# Tailwind CSS v4

This skill covers Tailwind CSS v4.x as documented on https://tailwindcss.com/docs. v4 is configured in CSS, not in JavaScript, and many v3 class names and defaults changed. The skill holds what differs from habit: the CSS-first setup, surprising defaults, renamed utilities and the patterns the docs recommend. Each claim in the reference files ends with the page it comes from.

## How to navigate

1. Read the rules below. They catch the most frequent v3 habits that break in v4.
2. Open the one reference file that matches the task, from the table at the end.
3. When the project has a `tailwind.config.js`, `@tailwind` directives or v3 class names, or when reviewing code that may carry v3 habits, read `references/upgrade-from-v3.md` first. The default migration path is `npx @tailwindcss/upgrade` in a new branch, then a review of its diff and a test in the browser. https://tailwindcss.com/docs/upgrade-guide#using-the-upgrade-tool

## Quick start (CSS-first, Vite)

```shell
npm install tailwindcss @tailwindcss/vite
```

```ts
// vite.config.ts
import { defineConfig } from 'vite'
import tailwindcss from '@tailwindcss/vite'
export default defineConfig({ plugins: [tailwindcss()] })
```

```css
/* app.css */
@import "tailwindcss";

@theme {
  --color-midnight: #121063;
  --font-display: "Oswald", sans-serif;
}

@custom-variant dark (&:where(.dark, .dark *));
```

- The whole setup is a package, a plugin and one `@import "tailwindcss";`. The installation steps use no `tailwind.config.js` and no `content` array. https://tailwindcss.com/docs/installation/using-vite
- Tailwind scans source files itself. Files in `.gitignore`, `node_modules`, CSS files and lock files are skipped. https://tailwindcss.com/docs/detecting-classes-in-source-files#which-files-are-scanned
- Next.js, Angular and other non-Vite bundlers use the PostCSS plugin instead: `npm install tailwindcss @tailwindcss/postcss postcss`, then `plugins: { "@tailwindcss/postcss": {} }`. https://tailwindcss.com/docs/installation/using-postcss
- A font URL `@import` (Google Fonts and the like) goes first, above `@import "tailwindcss"`, then the family is declared in `@theme`. https://tailwindcss.com/docs/font-family#customizing-your-theme
- `@theme` variables create utilities: `--color-midnight` gives `bg-midnight`, `text-midnight`, `fill-midnight`, and `--font-display` gives `font-display`. https://tailwindcss.com/docs/theme#what-are-theme-variables

## The rules that matter most

Setup and config

- The PostCSS plugin is `@tailwindcss/postcss`, not `tailwindcss`. Do not add `autoprefixer` or `postcss-import`: imports and vendor prefixes are built in. https://tailwindcss.com/docs/upgrade-guide#using-postcss
- In a Vite config, the framework guides list `tailwindcss()` first in `plugins`, before the framework plugin (React Router, SvelteKit, SolidJS, Qwik). The docs state no ordering requirement. https://tailwindcss.com/docs/installation/framework-guides/react-router https://tailwindcss.com/docs/installation/framework-guides/sveltekit
- Customize in CSS with `@theme`. A JS config is no longer detected and loads only through `@config "../../tailwind.config.js";`, without `corePlugins`, `safelist` or `separator`. https://tailwindcss.com/docs/upgrade-guide#using-a-javascript-config-file
- Safelist with `@source inline("{hover:,}bg-red-{50,{100..900..100},950}");`. https://tailwindcss.com/docs/detecting-classes-in-source-files#safelisting-specific-utilities
- A library built with Tailwind inside `node_modules` is not scanned: add `@source "../node_modules/@acmecorp/ui-lib";` (path relative to the stylesheet). https://tailwindcss.com/docs/detecting-classes-in-source-files#explicitly-registering-sources
- Never build class names dynamically (`` `bg-${color}-600` ``). Map values to complete class strings. https://tailwindcss.com/docs/detecting-classes-in-source-files#dynamic-class-names
- A Vue, Svelte or Astro `<style>` block and a CSS module are compiled separately. Before `@apply` there, add `@reference "../app.css";`, or use `var(--color-blue-500)` instead. https://tailwindcss.com/docs/compatibility#explicit-context-sharing https://tailwindcss.com/docs/compatibility#vue-svelte-and-astro
- Tailwind v4 is not designed to be used with CSS preprocessors like Sass, Less or Stylus, so they cannot be used for stylesheets or `<style>` blocks in Vue, Svelte or Astro. https://tailwindcss.com/docs/compatibility#sass-less-and-stylus
- v4 needs Safari 16.4+, Chrome 111+, Firefox 128+. Stay on v3.4 for older browsers. https://tailwindcss.com/docs/upgrade-guide#browser-requirements

Theme and custom CSS

- When a theme variable references another variable, write `@theme inline { --font-sans: var(--font-inter); }`, otherwise the value resolves where `--font-sans` is defined. https://tailwindcss.com/docs/theme#referencing-other-variables
- Remove a default namespace with `--color-*: initial;`, the whole theme with `--*: initial;`. https://tailwindcss.com/docs/theme#overriding-the-default-theme
- Declare custom utilities with `@utility name { ... }`, not `@layer utilities`, so variants work on them. https://tailwindcss.com/docs/upgrade-guide#adding-custom-utilities
- Read theme values in custom CSS and JS as CSS variables (`var(--color-red-500)`), not with `theme(colors.red.500)` or `resolveConfig`. https://tailwindcss.com/docs/upgrade-guide#theme-values-in-javascript
- Use the same unit for every breakpoint (rem like the defaults), or breakpoints override each other. https://tailwindcss.com/docs/responsive-design#customizing-your-theme

Class syntax

- CSS variable shorthand uses parentheses: `bg-(--brand-color)`, not `bg-[--brand-color]`. https://tailwindcss.com/docs/upgrade-guide#variables-in-arbitrary-values
- `text-(--my-var)` is ambiguous (font size or color), so add a type hint: `text-(length:--my-var)` or `text-(color:--my-var)`. The same hints exist on `font-(family-name:--f)`, `border-(length:--w)`, `bg-(image:--img)` and `shadow-(color:--c)`. https://tailwindcss.com/docs/adding-custom-styles#resolving-ambiguities https://tailwindcss.com/docs/font-family https://tailwindcss.com/docs/border-width https://tailwindcss.com/docs/background-image https://tailwindcss.com/docs/box-shadow
- The important modifier goes at the end: `bg-red-500!`. https://tailwindcss.com/docs/upgrade-guide#the-important-modifier
- Stacked variants apply left to right: `*:first:pt-0`, not `first:*:pt-0`. https://tailwindcss.com/docs/upgrade-guide#variant-stacking-order
- Unprefixed classes target all sizes. `sm:` does not mean mobile: write `text-center sm:text-left`. https://tailwindcss.com/docs/responsive-design#targeting-mobile-screens
- `hover:` only applies on devices that can hover (`@media (hover: hover)`). Treat hover as an enhancement. https://tailwindcss.com/docs/upgrade-guide#hover-styles-on-mobile

Changed defaults

- Border and divide colors default to `currentColor`: always set a color with `border`. https://tailwindcss.com/docs/upgrade-guide#default-border-color
- `ring` is 1px `currentColor`. Write `ring-3 ring-blue-500` for the v3 look. https://tailwindcss.com/docs/upgrade-guide#default-ring-width-and-color
- Hide focus outlines with `outline-hidden` (kept in forced-colors mode): v3 `focus:outline-none` becomes `focus:outline-hidden`. `outline-none` now removes the outline completely. https://tailwindcss.com/docs/upgrade-guide#renamed-outline-utility
- The scales shifted: v3 `shadow-sm`, `rounded-sm`, `blur-sm` are now `-xs`, and v3 bare `shadow`, `rounded`, `blur` are now `-sm`. https://tailwindcss.com/docs/upgrade-guide#updated-shadow-radius-and-blur-scales
- Gradients are `bg-linear-to-r`, plus `bg-radial` and `bg-conic-<angle>`. https://tailwindcss.com/docs/background-image#adding-a-linear-gradient
- `container` neither centers nor pads: write `container mx-auto px-4`. https://tailwindcss.com/docs/max-width#using-breakpoints-container
- For grids, wrapping layouts or children in a custom order, use `gap` instead of `space-x-*` and `space-y-*`. https://tailwindcss.com/docs/margin#limitations
- Buttons have `cursor: default`. https://tailwindcss.com/docs/upgrade-guide#buttons-use-the-default-cursor
- Placeholders use the current text color at 50% opacity. https://tailwindcss.com/docs/upgrade-guide#new-default-placeholder-color
- Display classes no longer override the `hidden` attribute: remove `hidden` to show the element. https://tailwindcss.com/docs/upgrade-guide#hidden-attribute-takes-priority

## Dark mode with a class

```css
@import "tailwindcss";
@custom-variant dark (&:where(.dark, .dark *));
```

```js
// inline in <head>, before the page renders
document.documentElement.classList.toggle(
  "dark",
  localStorage.theme === "dark" ||
    (!("theme" in localStorage) && window.matchMedia("(prefers-color-scheme: dark)").matches),
);
```

Set `localStorage.theme = "dark"` or `"light"`, and `localStorage.removeItem("theme")` to follow the system. https://tailwindcss.com/docs/dark-mode#with-system-theme-support

## Reference files

| File | Read it when |
| :--- | :--- |
| [references/installation-and-frameworks.md](references/installation-and-frameworks.md) | Installing Tailwind, choosing Vite, PostCSS or CLI, framework-specific steps, CSS modules, editor tooling |
| [references/upgrade-from-v3.md](references/upgrade-from-v3.md) | Migrating a v3 project, reviewing code with v3 class names or config |
| [references/utilities-and-variants.md](references/utilities-and-variants.md) | Writing classes: arbitrary values, conflicts, state variants, group and peer, responsive, container queries, dark mode |
| [references/theme-and-colors.md](references/theme-and-colors.md) | Adding or overriding colors, fonts, breakpoints, spacing, animations, theme namespaces |
| [references/custom-styles.md](references/custom-styles.md) | Base styles, Preflight, component classes, `@utility`, functional utilities, `@custom-variant`, `@variant` |
| [references/directives-and-source-detection.md](references/directives-and-source-detection.md) | `@apply`, `@reference`, `--alpha()`, `--spacing()`, `@config`, `@plugin`, `@source`, import options (prefix, important, source) |
| [references/layout-flexbox-grid.md](references/layout-flexbox-grid.md) | Display, position, inset, z-index, columns, flex, grid, alignment and `-safe` alignment |
| [references/spacing-and-sizing.md](references/spacing-and-sizing.md) | Padding, margin, space, width, height, size, container scale, viewport units, logical sizes |
| [references/typography.md](references/typography.md) | Fonts, font size and line height, letter spacing, text wrap, overflow wrap, decoration, content |
| [references/backgrounds-and-borders.md](references/backgrounds-and-borders.md) | Gradients, background image, size and position, radius, border width and color, outlines |
| [references/effects-and-filters.md](references/effects-and-filters.md) | Shadows, rings, text shadow, opacity, blend modes, masks, filters and backdrop filters |
| [references/transitions-transforms-tables.md](references/transitions-transforms-tables.md) | Transitions, discrete transitions, animations, rotate, scale, translate, 3D, zoom, tables |
| [references/interactivity-and-svg.md](references/interactivity-and-svg.md) | Color scheme, field sizing, scrollbars, scroll snap, cursor, will-change, SVG fill and stroke, forced colors |
