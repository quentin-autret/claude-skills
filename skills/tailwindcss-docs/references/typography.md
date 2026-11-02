# Typography

## Contents

- Type hints for text and font shorthands
- Font family and web fonts
- Font size and line height
- Weight, stretch, features, numerals
- Letter spacing, indent, tab size
- Wrapping, overflow, hyphens
- Decoration and underline
- Lists, alignment, content
- Old patterns

## Type hints for text and font shorthands

`text-*`, `font-*` and `decoration-*` each cover several properties. A variable needs a hint when it is not the default type:

| Class | Sets |
| :--- | :--- |
| `text-(--x)` | color |
| `text-(length:--x)` | font size |
| `font-(--x)` | font weight |
| `font-(family-name:--x)` | font family |
| `decoration-(--x)` | decoration color |
| `decoration-(length:--x)` | decoration thickness |

https://tailwindcss.com/docs/color, https://tailwindcss.com/docs/font-size#using-a-custom-value, https://tailwindcss.com/docs/font-family#using-a-custom-value, https://tailwindcss.com/docs/text-decoration-thickness#using-a-custom-value

## Font family and web fonts

- `font-sans`, `font-serif`, `font-mono` read `--font-sans`, `--font-serif`, `--font-mono`. `--font-<name>` in `@theme` adds `font-<name>`. https://tailwindcss.com/docs/font-family#customizing-your-theme
- A font URL `@import` must be the first rule, above `@import "tailwindcss"`, because that import is inlined: https://tailwindcss.com/docs/font-family#customizing-your-theme

```css
@import url("https://fonts.googleapis.com/css2?family=Roboto&display=swap");
@import "tailwindcss";
@theme { --font-roboto: "Roboto", sans-serif; }
```

- Load custom fonts with the `@font-face` at-rule. https://tailwindcss.com/docs/font-family#customizing-your-theme
- A family can carry its features: https://tailwindcss.com/docs/font-family#customizing-your-theme

```css
@theme {
  --font-display: "Oswald", sans-serif;
  --font-display--font-feature-settings: "cv02", "cv03", "cv04", "cv11";
  --font-display--font-variation-settings: "opsz" 32;
}
```

## Font size and line height

- `text-xs` to `text-9xl` read `--text-<name>` and set a line height from `--text-<name>--line-height`. https://tailwindcss.com/docs/font-size
- Size and line height in one class: `text-sm/6`, `text-lg/(--my-leading)`, `text-base/[1.75]`. https://tailwindcss.com/docs/line-height#basic-example
- `leading-<n>` is `calc(var(--spacing) * <n>)`, so `leading-6` follows the spacing scale. `leading-none` is 1. https://tailwindcss.com/docs/line-height#setting-independently
- A theme size can bundle `--text-<name>--line-height`, `--text-<name>--letter-spacing` and `--text-<name>--font-weight`. https://tailwindcss.com/docs/font-size#customizing-your-theme

## Weight, stretch, features, numerals

- Weights read `--font-weight-*`: `--font-weight-extrablack: 1000` gives `font-extrablack`. https://tailwindcss.com/docs/font-weight#customizing-your-theme
- `font-stretch-condensed`, `font-stretch-125%`. Only fonts with multiple width variations respond, otherwise the browser selects the closest match. https://tailwindcss.com/docs/font-stretch
- `font-features-['smcp','onum']`, `font-features-(--my-features)`. https://tailwindcss.com/docs/font-feature-settings#enabling-multiple-features
- Numeric variants compose: `slashed-zero tabular-nums`. https://tailwindcss.com/docs/font-variant-numeric#stacking-multiple-utilities
- `normal-nums` resets them, for example `slashed-zero tabular-nums md:normal-nums`. https://tailwindcss.com/docs/font-variant-numeric#resetting-numeric-font-variants
- `antialiased` sets both `-webkit-font-smoothing` and `-moz-osx-font-smoothing`. https://tailwindcss.com/docs/font-smoothing

## Letter spacing, indent, tab size

- `tracking-tighter` (-0.05em) to `tracking-widest` (0.1em) read `--tracking-*`. Negative values mainly make sense with a numeric scale, for example `--tracking-2: 0.025em` in the theme gives `-tracking-2`. https://tailwindcss.com/docs/letter-spacing#using-negative-values
- `indent-<n>` follows the spacing scale, negative `-indent-8`. https://tailwindcss.com/docs/text-indent
- `tab-2`, `tab-8`, `tab-(--var)`. https://tailwindcss.com/docs/tab-size

## Wrapping, overflow, hyphens

- `text-balance` suits headings: browsers balance only about 6 lines or fewer. `text-pretty` avoids orphans at some cost in speed. https://tailwindcss.com/docs/text-wrap#balanced-text-wrapping
- `truncate` sets overflow, ellipsis and nowrap. `text-ellipsis` and `text-clip` set only `text-overflow`, so add `overflow-hidden`. https://tailwindcss.com/docs/text-overflow#adding-an-ellipsis
- Overflow wrap: `wrap-break-word`, `wrap-anywhere`, `wrap-normal`. `wrap-anywhere` inside a flex child avoids the usual `min-width: 0`. https://tailwindcss.com/docs/overflow-wrap#wrapping-anywhere
- `break-keep` only affects CJK text. https://tailwindcss.com/docs/word-break#break-keep
- `line-clamp-3 lg:line-clamp-none`. https://tailwindcss.com/docs/line-clamp#undoing-line-clamping
- `hyphens-auto` lets the browser pick hyphenation points based on the language (the docs example sets `lang="de"`). `hyphens-manual` (the default) breaks only at `&shy;`, `hyphens-none` not even there. https://tailwindcss.com/docs/hyphens#automatic-hyphenation
- `whitespace-break-spaces` wraps trailing spaces instead of hanging them. https://tailwindcss.com/docs/white-space#break-spaces

## Decoration and underline

- `decoration-<n>` is in px (`decoration-2`), plus `decoration-from-font`. https://tailwindcss.com/docs/text-decoration-thickness
- `decoration-sky-500/30` takes an opacity modifier. https://tailwindcss.com/docs/text-decoration-color
- `underline-offset-<n>` is px, negative `-underline-offset-2`. https://tailwindcss.com/docs/text-underline-offset

## Lists, alignment, content

- Built-in list types are only `list-disc`, `list-decimal`, `list-none`. Others: `list-[upper-roman]`. https://tailwindcss.com/docs/list-style-type#using-a-custom-value
- `list-image-[url(/img/checkmark.png)]`. https://tailwindcss.com/docs/list-style-image
- `text-start` and `text-end` use logical properties, mapping to left or right based on the text direction. https://tailwindcss.com/docs/text-align#using-logical-properties
- `content-*` works with `before:` and `after:`: `after:content-['_↗']`, `before:content-[attr(before)]`. Spaces are underscores, a literal one is `\_`. https://tailwindcss.com/docs/content#using-spaces-and-underscores

## Old patterns

| v3 | v4 |
| :--- | :--- |
| `text-opacity-50` | `text-black/50` https://tailwindcss.com/docs/upgrade-guide#removed-deprecated-utilities |
| `overflow-ellipsis` | `text-ellipsis` https://tailwindcss.com/docs/upgrade-guide#removed-deprecated-utilities |
| `decoration-slice`, `decoration-clone` | `box-decoration-slice`, `box-decoration-clone` https://tailwindcss.com/docs/upgrade-guide#removed-deprecated-utilities |