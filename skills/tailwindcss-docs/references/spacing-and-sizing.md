# Spacing and sizing

## Contents

- The spacing model
- Padding and margin
- Space between children
- Width, size and the container scale
- Height and viewport units
- Min and max sizes
- The container utility
- Logical sizes
- Old patterns

## The spacing model

- `p-<n>`, `m-<n>`, `w-<n>`, `h-<n>`, `min-w-<n>` and the rest compile to `calc(var(--spacing) * <number>)`, driven by the `--spacing` theme variable (the theme page shows `--spacing: 4px;` inside `@theme`). https://tailwindcss.com/docs/padding and https://tailwindcss.com/docs/theme
- Fractions are computed: `w-2/5`, `h-9/10`, `max-w-9/10`, `min-w-3/4` work without config. https://tailwindcss.com/docs/width#using-a-percentage
- `px` is 1px: `p-px`, `-mt-px`, `size-px`. https://tailwindcss.com/docs/margin and https://tailwindcss.com/docs/width
- Custom property shorthand `w-(<custom-property>)` gives `width: var(<custom-property>)`, arbitrary value `w-[<value>]` gives `width: <value>`. https://tailwindcss.com/docs/width

## Padding and margin

- `px` and `py` compile to `padding-inline` and `padding-block`. `mx` and `my` to `margin-inline` and `margin-block`. https://tailwindcss.com/docs/padding
- Logical sides: `ps`, `pe`, `ms`, `me` follow text direction. `pbs`, `pbe`, `mbs`, `mbe` (block start and end) follow writing mode. https://tailwindcss.com/docs/padding#using-logical-properties
- The padding utility table lists no negative values and no `auto` variants (margin lists both). https://tailwindcss.com/docs/padding
- Negative margins: `-mt-8`, `-mx-px`. https://tailwindcss.com/docs/margin#using-negative-values

## Space between children

- Prefer the `gap` utilities when possible, or margin on every child with a matching negative margin on the parent. https://tailwindcss.com/docs/margin#limitations
- `space-x-*` and `space-y-*` add margin to every child except the last (`& > :not(:last-child)`). They break with grids, wrapping layouts and children shown in a different order than the DOM. https://tailwindcss.com/docs/margin#limitations
- Do not combine `space-*` with `divide-*`. Put margin or padding on the children instead. https://tailwindcss.com/docs/margin#limitations
- Reversed children (`flex-row-reverse`) need `space-x-reverse`. https://tailwindcss.com/docs/margin#reversing-children-order

## Width, size and the container scale

- Named widths come from `--container-*`: `w-3xs` 16rem, `w-2xs` 18rem, `w-xs` 20rem, `w-sm` 24rem, `w-md` 28rem, `w-lg` 32rem, `w-xl` 36rem, `w-2xl` 42rem, `w-3xl` 48rem, `w-4xl` 56rem, `w-5xl` 64rem, `w-6xl` 72rem, `w-7xl` 80rem. The same names work on `min-w-`, `max-w-`, `inline-`, `min-inline-`, `max-inline-`, not on heights. https://tailwindcss.com/docs/width#using-the-container-scale
- `size-*` sets width and height together: `size-12`, `size-full`, `size-1/2`. It has no container-scale names. https://tailwindcss.com/docs/width#setting-both-width-and-height
- Viewport widths: `w-screen` (100vw), `w-dvw`, `w-lvw`, `w-svw`. https://tailwindcss.com/docs/width
- Reset with `w-auto`, for example `w-full md:w-auto`. https://tailwindcss.com/docs/width#resetting-the-width

## Height and viewport units

- `h-dvh` follows the browser UI as it expands and collapses. `h-lvh` is the largest viewport height, `h-svh` the smallest, `h-screen` is 100vh. https://tailwindcss.com/docs/height#matching-dynamic-viewport
- `h-lh` is `height: 1lh`. Also `min-h-lh`, `max-h-lh`. https://tailwindcss.com/docs/height

## Min and max sizes

- `min-w-auto` and `min-h-auto` reset. `min-w-0` uses the numeric form. https://tailwindcss.com/docs/min-width#responsive-design
- `max-w-none` and `max-h-none` remove the limit. https://tailwindcss.com/docs/max-width
- Fluid up to a cap: `w-full max-w-96` (the documented examples pair `w-full` with a `max-w-*` value). https://tailwindcss.com/docs/max-width#basic-example
- `max-w-screen` is 100vw. No breakpoint-named `max-w-screen-md` is documented. https://tailwindcss.com/docs/max-width

## The container utility

- `container` sets `width: 100%` and a `max-width` matching the `min-width` of the current breakpoint (the table steps through 40rem, 48rem, 64rem, 80rem and 96rem). https://tailwindcss.com/docs/max-width#using-breakpoints-container
- It does not center and adds no padding: write `container mx-auto px-4`. https://tailwindcss.com/docs/max-width#using-breakpoints-container
- Change its behavior with `@utility container { margin-inline: auto; padding-inline: 2rem; }`. https://tailwindcss.com/docs/upgrade-guide#container-configuration

## Logical sizes

- `inline-<n>`, `min-inline-<n>`, `max-inline-<n>` set `inline-size`. `block-<n>`, `min-block-<n>`, `max-block-<n>` set `block-size`. https://tailwindcss.com/docs/inline-size
- `inline-*` takes the container scale (`max-inline-md`). `block-*` takes `lh`, `screen`, `dvh`, `lvh`, `svh`. https://tailwindcss.com/docs/block-size
- Reset with `inline-auto`, `block-auto`, `max-inline-none`, `max-block-none`. https://tailwindcss.com/docs/inline-size#resetting-the-inline-size

## Old patterns

| v3 | v4 |
| :--- | :--- |
| `space-y-4` on `> :not([hidden]) ~ :not([hidden])` | `:not(:last-child)` selector, prefer `gap` https://tailwindcss.com/docs/upgrade-guide#space-between-selector |
| `container` centered through `container.center` config | `container mx-auto` or `@utility container` https://tailwindcss.com/docs/upgrade-guide#container-configuration |
