# Theme variables and colors

## Contents

- Theme variables
- Namespaces
- Extending, overriding, removing
- Variables that reference variables
- Animations
- Sharing and emitting the theme
- Reading theme values
- Colors
- Old patterns

## Theme variables

- A theme variable is a CSS variable declared in `@theme`. It creates utilities and is also emitted on `:root` for `var()`, arbitrary values and inline styles. https://tailwindcss.com/docs/theme#what-are-theme-variables
- Use `@theme` when a token needs utilities, `:root` for a plain variable. `@theme` must be top level, never nested in a selector or media query. https://tailwindcss.com/docs/theme#why-theme-instead-of-root
- Any name inside a namespace works: `--font-poppins: Poppins, sans-serif;` gives `font-poppins`. https://tailwindcss.com/docs/theme#relationship-to-utility-classes
- Some namespaces create variants: `--breakpoint-3xl: 120rem;` gives `3xl:`. https://tailwindcss.com/docs/theme#relationship-to-variants
- `bg-red-200`, `font-serif`, `shadow-sm` exist only because the default theme defines them. `flex-col` and `pointer-events-none` are built in. https://tailwindcss.com/docs/theme#default-theme-variables

## Namespaces

| Namespace | Creates | 
| :--- | :--- |
| `--color-*` | `bg-*`, `text-*`, `border-*`, `fill-*` and every color utility |
| `--font-*` | `font-family` utilities |
| `--text-*` | font size, with companion `--text-<name>--line-height` |
| `--font-weight-*`, `--tracking-*`, `--leading-*` | weight, letter spacing, line height |
| `--breakpoint-*` | responsive variants |
| `--container-*` | `@sm:` container variants and sizes like `max-w-md`, `w-md`, `basis-md`, `columns-md` |
| `--spacing` | one base unit (`0.25rem` by default), multiplied by the number in `p-<number>`, `w-<number>`, `gap-<number>` (the theme page lists it as `--spacing-*`) |
| `--radius-*`, `--shadow-*`, `--inset-shadow-*`, `--drop-shadow-*`, `--text-shadow-*`, `--blur-*` | radius, shadows, blur |
| `--perspective-*`, `--aspect-*`, `--ease-*`, `--animate-*`, `--zoom-*`, `--tab-size-*` | their utilities |

https://tailwindcss.com/docs/theme#theme-variable-namespaces (`--text-shadow-*` is not in that table, it comes from https://tailwindcss.com/docs/text-shadow. The `w-md`, `basis-md` and `columns-md` sizes come from the width, flex-basis and columns pages.)

Defaults worth knowing: colors are oklch. Spacing is a single `--spacing: 0.25rem`, not a scale, and utilities such as `p-<number>` compute `calc(var(--spacing) * <number>)` (https://tailwindcss.com/docs/padding). Font sizes carry a line height. https://tailwindcss.com/docs/theme#default-theme-variable-reference

## Extending, overriding, removing

```css
@import "tailwindcss";

@theme {
  --font-script: Great Vibes, cursive; /* adds font-script */
  --breakpoint-sm: 30rem;             /* overrides sm */
  --color-lime-*: initial;            /* removes a color family */
}
```

https://tailwindcss.com/docs/theme#extending-the-default-theme (the `--color-lime-*: initial;` line comes from https://tailwindcss.com/docs/colors#disabling-default-colors)

- `--color-*: initial;` removes every default color, including `bg-red-500`. https://tailwindcss.com/docs/theme#overriding-the-default-theme
- `--*: initial;` removes the whole theme. You then define `--spacing`, fonts and colors yourself. https://tailwindcss.com/docs/theme#using-a-custom-theme
- Font size bundles: https://tailwindcss.com/docs/font-size#customizing-your-theme

```css
@theme {
  --text-tiny: 0.625rem;
  --text-tiny--line-height: 1.5rem;
  --text-tiny--letter-spacing: 0.125rem;
  --text-tiny--font-weight: 500;
}
```

- Font families can carry `--font-display--font-feature-settings` and `--font-display--font-variation-settings`. https://tailwindcss.com/docs/font-family#customizing-your-theme
- A font URL `@import` goes first, above `@import "tailwindcss"`: https://tailwindcss.com/docs/font-family#customizing-your-theme

```css
@import url("https://fonts.googleapis.com/css2?family=Roboto&display=swap");
@import "tailwindcss";
@theme { --font-roboto: "Roboto", sans-serif; }
```

## Variables that reference variables

Use `@theme inline` whenever a theme value is a `var()`: https://tailwindcss.com/docs/theme#referencing-other-variables

```css
@theme inline {
  --font-sans: var(--font-inter);
}
```

Without `inline`, `var(--font-sans)` resolves where `--font-sans` is defined. A `--font-inter` set lower in the tree is not seen and the text falls back to `sans-serif`. https://tailwindcss.com/docs/theme#referencing-other-variables

## Animations

Put `@keyframes` inside `@theme`, next to the `--animate-*` variable: https://tailwindcss.com/docs/animation#customizing-your-theme

```css
@theme {
  --animate-wiggle: wiggle 1s ease-in-out infinite;
  @keyframes wiggle {
    0%, 100% { transform: rotate(-3deg); }
    50% { transform: rotate(3deg); }
  }
}
```

Keyframes outside `@theme` are always emitted, even when unused. https://tailwindcss.com/docs/theme#defining-animation-keyframes

## Sharing and emitting the theme

- Only the theme variables in use are emitted. `@theme static { ... }` emits all of them. https://tailwindcss.com/docs/theme#generating-all-css-variables
- Share a theme as its own CSS file (package or monorepo) and `@import` it after `@import "tailwindcss"`. https://tailwindcss.com/docs/theme#sharing-across-projects

## Reading theme values

- In CSS: `var(--text-base)`, `var(--text-2xl--line-height)`, `var(--font-weight-semibold)`. In a class: `rounded-[calc(var(--radius-xl)-1px)]`. https://tailwindcss.com/docs/theme#using-your-theme-variables
- In JS: pass the variable, `animate={{ backgroundColor: "var(--color-blue-500)" }}`, or read it with `getComputedStyle(document.documentElement).getPropertyValue("--shadow-xl")`. https://tailwindcss.com/docs/theme#referencing-in-javascript

## Colors

- 11 steps per color, `50` to `950`. Families include slate, gray, zinc, neutral, stone, mauve, olive, mist and taupe. https://tailwindcss.com/docs/theme#default-theme-variable-reference
- Opacity: `bg-black/75`, `bg-pink-500/[71.37%]`, `bg-cyan-400/(--my-alpha-value)`. https://tailwindcss.com/docs/colors#adjusting-opacity
- Less obvious color utilities: `decoration-*`, `outline-*`, `shadow-*`, `inset-shadow-*`, `ring-*`, `inset-ring-*`, `accent-*`, `caret-*`, `scrollbar-thumb-*`, `scrollbar-track-*`, `fill-*`, `stroke-*`. https://tailwindcss.com/docs/colors#using-color-utilities
- Alpha in CSS: `--alpha(var(--color-gray-950) / 10%)`. https://tailwindcss.com/docs/colors#referencing-in-css
- Add a color: `@theme { --color-midnight: #121063; }`. https://tailwindcss.com/docs/colors#customizing-your-colors
- Semantic, theme-aware colors: define the value per theme, then expose it with `@theme inline`. https://tailwindcss.com/docs/colors#referencing-other-variables

```css
:root { --acme-canvas-color: /* light value */; }
[data-theme="dark"] { --acme-canvas-color: /* dark value */; }
@theme inline { --color-canvas: var(--acme-canvas-color); }
```

## Old patterns

| v3 | v4 |
| :--- | :--- |
| theme customized in `tailwind.config.js` | variables in `@theme` (https://tailwindcss.com/docs/theme). A JS config is no longer detected automatically and loads only through `@config "../../tailwind.config.js";` https://tailwindcss.com/docs/upgrade-guide#using-a-javascript-config-file |
| `theme(colors.red.500)` | `var(--color-red-500)`, or the variable name inside `theme()` where a variable cannot be used, as in media queries: `@media (width >= theme(--breakpoint-xl))` https://tailwindcss.com/docs/upgrade-guide#using-the-theme-function |
| `resolveConfig(config)` | CSS variables https://tailwindcss.com/docs/upgrade-guide#theme-values-in-javascript |
