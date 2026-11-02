# Custom styles: base, Preflight, components, utilities, variants

## Contents

- Arbitrary values, whitespace and type hints
- Base styles and Preflight
- Component classes
- Custom utilities with @utility
- Functional utilities
- Custom variants and @variant
- Old patterns

## Arbitrary values, whitespace and type hints

- `fill-(--my-brand-color)` equals `fill-[var(--my-brand-color)]`. https://tailwindcss.com/docs/adding-custom-styles#using-arbitrary-values
- Arbitrary variants stack: `lg:[&:nth-child(-n+3)]:hover:underline`. https://tailwindcss.com/docs/adding-custom-styles#arbitrary-variants
- Underscore becomes a space: `grid-cols-[1fr_500px_2fr]`. It stays an underscore where a space is invalid (`bg-[url('/what_a_rush.png')]`). Escape a literal one: `before:content-['hello\_world']`. In JSX, wrap it in `String.raw` so the backslash survives. https://tailwindcss.com/docs/adding-custom-styles#handling-whitespace
- Literal values are inferred (`text-[22px]` size, `text-[#bada55]` color). A variable is ambiguous, so add a type: `text-(length:--my-var)`, `text-(color:--my-var)`. https://tailwindcss.com/docs/adding-custom-styles#resolving-ambiguities

## Base styles and Preflight

- Prefer classes on `html` or `body` in markup. Element defaults go in `@layer base`: https://tailwindcss.com/docs/adding-custom-styles#adding-base-styles

```css
@layer base {
  h1 { font-size: var(--text-2xl); }
  h2 { font-size: var(--text-xl); }
}
```

- Preflight (built on modern-normalize) is injected in the `base` layer by `@import "tailwindcss"`. https://tailwindcss.com/docs/preflight
- Preflight resets that surprise: https://tailwindcss.com/docs/preflight#margins-are-removed
  - all margins and paddings are 0
  - every element has `box-sizing: border-box` and `border: 0 solid`, so `border` alone gives 1px solid `currentColor`
  - headings inherit font size and weight, lists have `list-style: none`
  - `img`, `svg`, `video`, `canvas`, `audio`, `iframe`, `embed`, `object` are `display: block` (undo with `inline`)
  - `img` and `video` get `max-width: 100%; height: auto` (undo with `max-w-none`)
  - `[hidden]` is `display: none !important` unless `hidden="until-found"`
- The border reset can break third-party widgets: `@layer base { .google-map * { border-style: none; } }`. https://tailwindcss.com/docs/preflight#border-styles-are-reset
- VoiceOver does not announce unstyled lists: add `role="list"` to a `<ul>` that is a real list. https://tailwindcss.com/docs/preflight#accessibility-considerations
- Without Preflight, import the layers yourself: https://tailwindcss.com/docs/preflight#disabling-preflight

```css
@layer theme, base, components, utilities;
@import "tailwindcss/theme.css" layer(theme);
@import "tailwindcss/utilities.css" layer(utilities);
```

- Then put each option on its import: `source(none)` and `important` on `utilities.css`, `theme(static)` and `theme(inline)` on `theme.css`, `prefix(tw)` on both. https://tailwindcss.com/docs/preflight#disabling-preflight

## Component classes

- You need fewer than you think: a template partial is the default for repeated markup. https://tailwindcss.com/docs/adding-custom-styles#adding-component-classes
- When needed, put them in `@layer components`, so `card rounded-none` still works. It is also the place for third-party styles like `.select2-dropdown`. https://tailwindcss.com/docs/adding-custom-styles#adding-component-classes
- Write them with theme variables and functions: https://tailwindcss.com/docs/styling-with-utility-classes#using-custom-css

```css
@layer components {
  .btn-primary {
    border-radius: calc(infinity * 1px);
    padding-inline: --spacing(5);
    color: var(--color-white);
    box-shadow: var(--shadow-md);
    &:hover {
      @media (hover: hover) { /* hover styles */ }
    }
  }
}
```

## Custom utilities with @utility

- `@utility content-auto { content-visibility: auto; }` lands in the `utilities` layer and works with every variant (`hover:content-auto`). https://tailwindcss.com/docs/adding-custom-styles#simple-utilities
- Complex utilities nest: https://tailwindcss.com/docs/adding-custom-styles#complex-utilities

```css
@utility scrollbar-hidden {
  &::-webkit-scrollbar { display: none; }
}
```

## Functional utilities

`@utility tab-*` can match `tab-2`, `tab-github`, `tab-[76]` (depending on the `--value()` forms used): https://tailwindcss.com/docs/adding-custom-styles#functional-utilities

```css
@utility tab-* {
  tab-size: --value(--tab-size-*, integer, [integer]);
}
```

- `--value(--tab-size-*)` matches theme keys (`tab-github` when `--tab-size-github` exists).
- `--value(integer)` matches bare values. Bare types: `number`, `integer`, `ratio`, `percentage`.
- `--value("inherit", "initial")` matches literals.
- `--value([integer])` matches arbitrary values. Types include `length`, `color`, `angle`, `percentage`, `url`, `*`.
- Arguments resolve left to right. A declaration that resolves to nothing is dropped, so several declarations with different forms can coexist.
- `--value(integer, --default(4))` makes bare `tab` work.
- Negative values: a separate `@utility -inset-*` that multiplies by `-1` (`--spacing(--value(integer) * -1)`, `calc(--value([percentage], [length]) * -1)`).
- Modifiers (the part after the slash) via `--modifier(...)`, same syntax. A declaration that needs an absent modifier is omitted.
- Fractions: `--value(--aspect-ratio-*, ratio, [ratio])` matches `aspect-square`, `aspect-3/4`, `aspect-[7/9]`.

https://tailwindcss.com/docs/adding-custom-styles#functional-utilities

## Custom variants and @variant

- Shorthand: `@custom-variant theme-midnight (&:where([data-theme="midnight"] *));`. https://tailwindcss.com/docs/adding-custom-styles#adding-custom-variants
- Several rules use a block with `@slot`: https://tailwindcss.com/docs/adding-custom-styles#adding-custom-variants

```css
@custom-variant any-hover {
  @media (any-hover: hover) {
    &:hover { @slot; }
  }
}
```

- `@custom-variant` with a built-in name replaces it, for example `@custom-variant hover (&:hover);`. https://tailwindcss.com/docs/upgrade-guide#hover-styles-on-mobile
- `@variant` applies a variant inside custom CSS: `@variant dark { ... }`, stacked `@variant hover:focus { }`, several `@variant hover, focus { }`. https://tailwindcss.com/docs/adding-custom-styles#using-variants

```css
.my-element {
  background: white;
  @variant dark { background: black; }
}
```

## Old patterns

| v3 | v4 |
| :--- | :--- |
| `@layer utilities { .tab-4 { tab-size: 4; } }` | `@utility tab-4 { tab-size: 4; }` https://tailwindcss.com/docs/upgrade-guide#adding-custom-utilities |
| `theme(spacing.12)` in CSS (deprecated, use theme variables) | `--spacing(4)` function or `var(--*)` theme variables https://tailwindcss.com/docs/functions-and-directives#theme-function and https://tailwindcss.com/docs/functions-and-directives#spacing-function |
