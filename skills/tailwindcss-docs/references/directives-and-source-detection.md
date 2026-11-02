# Directives, functions and source detection

## Contents

- Directives
- @apply and @reference
- Functions
- Import options
- Bridging a v3 config or plugin
- How classes are detected
- @source
- Safelisting and excluding classes
- Old patterns

## Directives

`@import`, `@theme`, `@source`, `@utility`, `@variant`, `@custom-variant`, `@apply`, `@reference`, plus `@config` and `@plugin` for v3 compatibility. https://tailwindcss.com/docs/functions-and-directives#directives

`@import "tailwindcss"` expands to `@layer theme, base, components, utilities;` and imports `theme.css`, `preflight.css` and `utilities.css` in those layers. https://tailwindcss.com/docs/theme#default-theme-variables

## @apply and @reference

- `@apply` inlines utilities in custom CSS, for example to override the styles of a third-party library: `.select2-dropdown { @apply rounded-b-lg shadow-md; }`. https://tailwindcss.com/docs/functions-and-directives#apply-directive
- A Vue or Svelte `<style>` block and a CSS module do not see your theme, custom utilities or variants. Import them for reference, which emits no CSS twice: https://tailwindcss.com/docs/functions-and-directives#reference-directive

```vue
<style>
  @reference "../../app.css";

  h1 {
    @apply text-2xl font-bold text-red-500;
  }
</style>
```

- With no customizations at all, `@reference "tailwindcss";` is enough. https://tailwindcss.com/docs/functions-and-directives#reference-directive
- Faster alternative in those blocks: `var(--color-blue-500)` instead of `@apply`, which lets Tailwind skip the file. https://tailwindcss.com/docs/compatibility#explicit-context-sharing
- Subpath imports from `package.json` (`"imports": { "#app.css": "./src/css/app.css" }`) work in `@import`, `@reference`, `@plugin` and `@config` with the CLI, Vite and PostCSS. https://tailwindcss.com/docs/functions-and-directives#subpath-imports

## Functions

- `--alpha(var(--color-lime-300) / 50%)` compiles to `color-mix(in oklab, var(--color-lime-300) 50%, transparent)`. https://tailwindcss.com/docs/functions-and-directives#alpha-function
- `--spacing(4)` compiles to `calc(var(--spacing) * 4)`. It also works in arbitrary values: `py-[calc(--spacing(4)-1px)]`. https://tailwindcss.com/docs/functions-and-directives#spacing-function
- `theme()` is deprecated. Use CSS variables. Where a variable is not allowed, such as a media query, pass the variable name: `@media (width >= theme(--breakpoint-xl))`. https://tailwindcss.com/docs/upgrade-guide#using-the-theme-function
- In a Svelte `<style>` block, the docs read a theme value with `theme(--color-gray-100)` after `@reference "tailwindcss";`. https://tailwindcss.com/docs/installation/framework-guides/sveltekit

## Import options

| Option | Effect | Source |
| :--- | :--- | :--- |
| `@import "tailwindcss" prefix(tw);` | classes become `tw:text-red-500`, variables `--tw-color-red-500` | https://tailwindcss.com/docs/styling-with-utility-classes#using-the-prefix-option |
| `@import "tailwindcss" important;` | every utility gets `!important` | https://tailwindcss.com/docs/styling-with-utility-classes#using-the-important-flag |
| `@import "tailwindcss" source("../src");` | sets the scan base path | https://tailwindcss.com/docs/detecting-classes-in-source-files#setting-your-base-path |
| `@import "tailwindcss" source(none);` | turns off automatic detection | https://tailwindcss.com/docs/detecting-classes-in-source-files#disabling-automatic-detection |

## Bridging a v3 config or plugin

- `@config "../../tailwind.config.js";` loads a JS config. `corePlugins`, `safelist` and `separator` are not supported in v4.0. https://tailwindcss.com/docs/functions-and-directives#config-directive
- `@plugin "@tailwindcss/typography";` loads a JS plugin, by package name or local path. https://tailwindcss.com/docs/functions-and-directives#plugin-directive
- What CSS defines merges with the config and plugins where possible, and takes precedence otherwise, so a migration can be gradual. https://tailwindcss.com/docs/functions-and-directives#compatibility

## How classes are detected

- Tailwind reads files as plain text and keeps every token that matches a utility. https://tailwindcss.com/docs/detecting-classes-in-source-files#how-classes-are-detected
- Class names must appear whole in the source. `` `bg-${color}-600` `` and `text-{{ error ? 'red' : 'green' }}-600` are never generated. Map props to full strings: https://tailwindcss.com/docs/detecting-classes-in-source-files#dynamic-class-names

```jsx
const colorVariants = {
  blue: "bg-blue-600 hover:bg-blue-500",
  red: "bg-red-600 hover:bg-red-500",
};
<button className={colorVariants[color]} />
```

- Skipped automatically: files in `.gitignore`, `node_modules`, binary files, CSS files, lock files. https://tailwindcss.com/docs/detecting-classes-in-source-files#which-files-are-scanned
- The base path is the current working directory. In a monorepo run from the root, set `source("../src")`. https://tailwindcss.com/docs/detecting-classes-in-source-files#setting-your-base-path

## @source

Paths are relative to the stylesheet. https://tailwindcss.com/docs/detecting-classes-in-source-files#explicitly-registering-sources

```css
@import "tailwindcss";
@source "../node_modules/@acmecorp/ui-lib"; /* a gitignored library built with Tailwind */
@source not "../src/components/legacy";     /* ignore a path */
```

- With `source(none)`, list every path with `@source`. Useful with several stylesheets. https://tailwindcss.com/docs/detecting-classes-in-source-files#disabling-automatic-detection
- The Symfony guide adds an `@source` that ignores the public dir to prevent recompile loops in watch mode: `@source not "../../public";`. https://tailwindcss.com/docs/installation/framework-guides/symfony
- The AdonisJS guide tells Tailwind to scan the `resources/views` directory with `@source "../views";`. https://tailwindcss.com/docs/installation/framework-guides/adonisjs

## Safelisting and excluding classes

```css
@source inline("underline");
@source inline("{hover:,focus:,}underline");
@source inline("{hover:,}bg-red-{50,{100..900..100},950}");
@source not inline("{hover:,focus:,}bg-red-{50,{100..900..100},950}");
```

`inline()` takes brace expansion for variants and ranges. `@source not inline()` blocks classes even when found. https://tailwindcss.com/docs/detecting-classes-in-source-files#safelisting-specific-utilities

## Old patterns

| v3 | v4 |
| :--- | :--- |
| `safelist: [...]` | `@source inline("...")` https://tailwindcss.com/docs/upgrade-guide#using-a-javascript-config-file |
| `theme(colors.red.500)` | `var(--color-red-500)` https://tailwindcss.com/docs/upgrade-guide#using-the-theme-function |
| JS-based plugin | `@plugin "@tailwindcss/typography";` https://tailwindcss.com/docs/functions-and-directives#plugin-directive |
