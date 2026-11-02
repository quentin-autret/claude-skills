# Backgrounds, borders and outlines

## Contents

- Background color and image
- Gradients
- Background position and size
- Border radius
- Border width, color and style
- Dividers
- Outlines
- Old patterns

## Background color and image

- `bg-(--x)` is a color. An image variable needs the hint: `bg-(image:--my-image)`. https://tailwindcss.com/docs/background-image https://tailwindcss.com/docs/background-color#using-a-custom-value
- Image: `bg-[url(/img/mountains.jpg)]`. `bg-none` removes it. https://tailwindcss.com/docs/background-image#removing-background-images
- Opacity: `bg-sky-500/75`. https://tailwindcss.com/docs/background-color#changing-the-opacity
- Gradient text: `bg-linear-to-r from-pink-500 to-violet-500 bg-clip-text text-transparent`. https://tailwindcss.com/docs/background-clip#cropping-to-text

## Gradients

- Linear: `bg-linear-to-{t,tr,r,br,b,bl,l,tl}`, angles `bg-linear-65`, `-bg-linear-45`. They interpolate in oklab by default. https://tailwindcss.com/docs/background-image#adding-a-linear-gradient
- Radial: `bg-radial`, positioned `bg-radial-[at_50%_75%]`. https://tailwindcss.com/docs/background-image#adding-a-radial-gradient
- Conic: `bg-conic`, `bg-conic-180`. https://tailwindcss.com/docs/background-image#adding-a-conic-gradient
- Stops: `from-*`, `via-*`, `to-*`, positions `from-10% via-30% to-90%`. https://tailwindcss.com/docs/background-image#setting-gradient-color-stops
- Interpolation modifier: `bg-linear-to-r/srgb`, `/hsl`, `/oklab`, `/oklch`, `/longer`, `/shorter`, `/increasing`, `/decreasing`. https://tailwindcss.com/docs/background-image#changing-interpolation-mode
- Full custom gradient: `bg-linear-[25deg,red_5%,yellow_60%,lime_90%,teal]`. https://tailwindcss.com/docs/background-image#using-a-custom-value
- A variant changes only the stop it names. `dark:via-none` turns a 3-stop gradient back into 2 stops. https://tailwindcss.com/docs/upgrade-guide#using-variants-with-gradients

```html
<div class="bg-linear-to-r from-indigo-500 via-purple-500 to-pink-500 dark:via-none">
```

## Background position and size

- Custom values use the explicit prefixes: `bg-position-[center_top_1rem]`, `bg-size-[auto_100px]`, `bg-position-(--x)`, `bg-size-(--x)`. https://tailwindcss.com/docs/background-position#using-a-custom-value https://tailwindcss.com/docs/background-size#using-a-custom-value
- Repeat modes are `bg-repeat-space` and `bg-repeat-round`. https://tailwindcss.com/docs/background-repeat

## Border radius

- Scale from `--radius-*`: `xs` 0.125rem, `sm` 0.25rem, `md` 0.375rem, `lg` 0.5rem, `xl` 0.75rem, `2xl` 1rem, `3xl` 1.5rem, `4xl` 2rem. Add `--radius-5xl: 3rem` for `rounded-5xl`. https://tailwindcss.com/docs/border-radius#customizing-your-theme
- `rounded-full` is `calc(infinity * 1px)`. https://tailwindcss.com/docs/border-radius
- Logical corners follow `dir`: `rounded-s-*`, `rounded-e-*`, `rounded-ss-*`, `rounded-se-*`, `rounded-es-*`, `rounded-ee-*`. https://tailwindcss.com/docs/border-radius#using-logical-properties

## Border width, color and style

- Preflight sets `border: 0 solid` on everything, so `border` adds a solid 1px border in `currentColor`. https://tailwindcss.com/docs/preflight Specify a color wherever you use `border-*` or `divide-*`: `border border-gray-200`. https://tailwindcss.com/docs/upgrade-guide#default-border-color
- `border-<number>` is px. A variable width needs the hint: `border-(length:--x)`. `border-(--x)` is a color. https://tailwindcss.com/docs/border-width#using-a-custom-value https://tailwindcss.com/docs/border-color#using-a-custom-value
- `border-x` and `border-y` map to `border-inline-width` and `border-block-width`. Logical sides: `border-s`, `border-e` (inline), `border-bs`, `border-be` (block). https://tailwindcss.com/docs/border-width#using-logical-properties
- Set all sides, then override one: `border-4 border-indigo-200 border-t-indigo-500`. https://tailwindcss.com/docs/border-color#individual-sides
- `border-indigo-500/50` takes an opacity modifier. https://tailwindcss.com/docs/border-color#changing-the-opacity
- `border-none` removes the border style, most commonly one applied at a smaller breakpoint. https://tailwindcss.com/docs/border-style#removing-a-border

## Dividers

- `divide-x-<n>` and `divide-y-<n>` border every child except the last (`& > :not(:last-child)`): `divide-x-4 divide-dashed divide-indigo-500`. https://tailwindcss.com/docs/border-width#between-children
- With `flex-row-reverse` or `flex-col-reverse`, add `divide-x-reverse` or `divide-y-reverse`. https://tailwindcss.com/docs/border-width#reversing-children-order

## Outlines

- `outline` is 1px. `outline-<n>` is px and implies `outline-style: solid`. https://tailwindcss.com/docs/upgrade-guide#renamed-outline-utility
- Focus ring pattern: `outline-offset-2 outline-sky-500 focus:outline-2`. https://tailwindcss.com/docs/outline-width#applying-on-focus
- `outline-hidden` hides the browser outline but keeps it in forced-colors mode (`2px solid transparent`). It replaces the v3 `outline-none`. https://tailwindcss.com/docs/outline-style#hiding-an-outline
- `outline-none` sets `outline-style: none` and removes the outline completely. https://tailwindcss.com/docs/outline-style#removing-outlines
- With either one, provide your own visible focus style. https://tailwindcss.com/docs/outline-style#hiding-an-outline
- `outline-solid`, `outline-dashed` and the other style utilities set only the style. https://tailwindcss.com/docs/outline-style#basic-example
- Set the outline color without a variant (`outline-cyan-500 transition hover:outline-2`), because `transition` animates `outline-color`. https://tailwindcss.com/docs/upgrade-guide#transitioning-outline-color

## Old patterns

| v3 | v4 |
| :--- | :--- |
| `bg-gradient-to-r` | `bg-linear-to-r` https://tailwindcss.com/docs/background-image#adding-a-linear-gradient |
| `bg-opacity-50`, `border-opacity-50` | `bg-black/50`, `border-black/50` https://tailwindcss.com/docs/upgrade-guide#removed-deprecated-utilities |
| `bg-[--x]` | `bg-(--x)` https://tailwindcss.com/docs/upgrade-guide#variables-in-arbitrary-values |
| `rounded-sm`, `rounded` | `rounded-xs`, `rounded-sm` https://tailwindcss.com/docs/upgrade-guide#updated-shadow-radius-and-blur-scales |
| `border` gray by default | `border` is `currentColor`, add a color https://tailwindcss.com/docs/upgrade-guide#default-border-color |
| `focus:outline-none` | `focus:outline-hidden` https://tailwindcss.com/docs/upgrade-guide#renamed-outline-utility |
| `outline outline-2` | `outline-2` https://tailwindcss.com/docs/upgrade-guide#renamed-outline-utility |
