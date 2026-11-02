# Effects, masks and filters

## Contents

- Box shadow, inset shadow and rings
- Text shadow
- Opacity and blend modes
- Masks
- Filters
- Backdrop filters
- Old patterns

## Box shadow, inset shadow and rings

- Scale from `--shadow-*`: `shadow-2xs`, `shadow-xs`, `shadow-sm`, `shadow-md`, `shadow-lg`, `shadow-xl`, `shadow-2xl`. `shadow-none` outputs `0 0 #0000`. https://tailwindcss.com/docs/box-shadow
- Related utility families: `shadow-*`, `inset-shadow-*` (`--inset-shadow-*`: `2xs`, `xs`, `sm`), `ring-*` and `inset-ring-*`. Remove them with `shadow-none`, `inset-shadow-none`, `ring-0`, `inset-ring-0`. https://tailwindcss.com/docs/box-shadow#removing-a-box-shadow
- `ring` is 1px, `ring-<n>` is px, and the ring color defaults to `currentColor`. https://tailwindcss.com/docs/box-shadow#adding-a-ring
- Default shadows are faint (25% or less, inset 5%). Raise them with a modifier: `shadow-xl/20`, `inset-shadow-sm/50`. https://tailwindcss.com/docs/box-shadow#changing-the-opacity
- Colored shadows are 100% opaque by default: `shadow-cyan-500/50`, `ring-blue-500/50`. https://tailwindcss.com/docs/box-shadow#setting-the-shadow-color
- `shadow-(--x)` sets the shadow, `shadow-(color:--x)` only its color. Arbitrary: `shadow-[0_35px_35px_rgba(0,0,0,0.25)]`. https://tailwindcss.com/docs/box-shadow#using-a-custom-value
- Theme: define `--shadow-3xl: 0 35px 35px rgba(0, 0, 0, 0.25)` in the `--shadow-*` theme namespace (inside `@theme`) to get `shadow-3xl`. https://tailwindcss.com/docs/box-shadow#customizing-shadows

## Text shadow

- `text-shadow-2xs`, `-xs`, `-sm`, `-md`, `-lg`, `-none`, from `--text-shadow-*`. https://tailwindcss.com/docs/text-shadow
- Faint by default (20% or less): `text-shadow-lg/20`, or about 50% for a stronger one. https://tailwindcss.com/docs/text-shadow#changing-the-opacity
- Color: `text-shadow-sky-300`, usable in an arbitrary value as `text-shadow-[0px_5px_0px_var(--tw-text-shadow-color)]`. https://tailwindcss.com/docs/text-shadow#setting-the-shadow-color
- `text-shadow-lg dark:text-shadow-none`. https://tailwindcss.com/docs/text-shadow#removing-a-text-shadow

## Opacity and blend modes

- `opacity-<n>` is a percentage (`opacity: <number>%`). Arbitrary `opacity-[.67]`. https://tailwindcss.com/docs/opacity
- `mix-blend-*` includes `mix-blend-plus-darker` and `mix-blend-plus-lighter`. Add `isolate` on the parent to stop blending with what is behind it. https://tailwindcss.com/docs/mix-blend-mode#isolating-blending
- `bg-blend-*` blends a background image with the background color. https://tailwindcss.com/docs/background-blend-mode#basic-example

## Masks

- Image mask: `mask-[url(/img/scribble.png)]`, `mask-none`. https://tailwindcss.com/docs/mask-image#using-an-image-mask
- Edge masks: `mask-t-from-50%`, `mask-b-to-80%`, axis forms `mask-x-from-70%`, `mask-y-to-90%`. https://tailwindcss.com/docs/mask-image#masking-edges
- Angled: `mask-linear-50`, `-mask-linear-50`, with `mask-linear-from-*` and `mask-linear-to-*`. https://tailwindcss.com/docs/mask-image#adding-an-angled-linear-mask
- Radial: `mask-radial-from-*`, `mask-radial-to-*`, center `mask-radial-at-top-left` or `mask-radial-at-[35%_35%]`, shape `mask-circle` or `mask-ellipse` (default), size `mask-radial-closest-side` and similar. https://tailwindcss.com/docs/mask-image#adding-a-radial-mask
- Conic: `mask-conic-<angle>`, `mask-conic-from-*`. https://tailwindcss.com/docs/mask-image#adding-a-conic-mask
- Stop values: a bare number follows the spacing scale, a percentage (`mask-b-from-50%`), a color, a variable or an arbitrary value. Masks transition from black to transparent by default. https://tailwindcss.com/docs/mask-image#masking-edges
- `mask-radial-at-*` moves the gradient center. `mask-position` moves the mask image. https://tailwindcss.com/docs/mask-image#setting-the-radial-position
- A circle takes one length (`mask-radial-[5rem]`), an ellipse 2 values (`mask-radial-[40%_80%]`). https://tailwindcss.com/docs/mask-image#setting-the-radial-size
- Masks stack because `mask-composite` defaults to intersect: `mask-b-from-50% mask-radial-[50%_90%] mask-radial-from-80%`. https://tailwindcss.com/docs/mask-image#combining-masks
- `mask-add`, `mask-subtract`, `mask-intersect`, `mask-exclude` change how they combine. https://tailwindcss.com/docs/mask-composite#basic-example
- `mask-alpha`, `mask-luminance`, `mask-match` set `mask-mode`. `mask-type-alpha` and `mask-type-luminance` are for SVG `<mask>`. Grayscale masks give the most predictable luminance result. https://tailwindcss.com/docs/mask-mode#basic-example
- Position `mask-top-left`, size `mask-cover`, `mask-size-[50px_50px]`, repeat `mask-no-repeat`, `mask-repeat-round`. https://tailwindcss.com/docs/mask-position, https://tailwindcss.com/docs/mask-size, https://tailwindcss.com/docs/mask-repeat

## Filters

- Filters compose: `blur-xs grayscale`. `filter-none` removes every filter, for example `md:filter-none`. https://tailwindcss.com/docs/filter#removing-filters
- Blur scale from `--blur-*`: `xs` 4px, `sm` 8px, `md` 12px, `lg` 16px, `xl` 24px, `2xl` 40px, `3xl` 64px. No bare `blur` is listed. https://tailwindcss.com/docs/filter-blur
- `brightness-<n>`, `contrast-<n>`, `saturate-<n>` are percentages, any number. Arbitrary values are raw CSS factors: `brightness-[1.75]`, `contrast-[.25]`. https://tailwindcss.com/docs/filter-brightness#using-a-custom-value
- `grayscale`, `invert`, `sepia` alone mean 100%. `grayscale-75` is 75%. https://tailwindcss.com/docs/filter-grayscale
- `hue-rotate-90` is degrees, negative `-hue-rotate-15`. https://tailwindcss.com/docs/filter-hue-rotate#using-negative-values
- Scale from `--drop-shadow-*`, faint by default (15% or less): `drop-shadow-xl/25`, color `drop-shadow-cyan-500/50`, `drop-shadow-(color:--x)`. https://tailwindcss.com/docs/filter-drop-shadow#changing-the-opacity
- SVG filter: `filter-[url('filters.svg#filter-id')]`. https://tailwindcss.com/docs/filter#using-a-custom-value

## Backdrop filters

- Same model with a `backdrop-` prefix: `backdrop-blur-*`, `backdrop-brightness-*`, `backdrop-opacity-*` and the others. https://tailwindcss.com/docs/backdrop-filter#basic-example
- Pair with a translucent background: `bg-white/30 backdrop-blur-xs`. https://tailwindcss.com/docs/backdrop-filter#basic-example
- `backdrop-opacity-*` sets the opacity of the backdrop filters, not of the element. https://tailwindcss.com/docs/backdrop-filter-opacity#basic-example
- The negative dash goes first: `-backdrop-hue-rotate-90`. https://tailwindcss.com/docs/backdrop-filter-hue-rotate#using-negative-values
- `backdrop-filter-none` removes all of them. https://tailwindcss.com/docs/backdrop-filter#removing-filters

## Old patterns

| v3 | v4 |
| :--- | :--- |
| `shadow-sm`, `shadow` | `shadow-xs`, `shadow-sm` https://tailwindcss.com/docs/upgrade-guide#updated-shadow-radius-and-blur-scales |
| `blur-sm`, `blur`, `drop-shadow`, `backdrop-blur` | `blur-xs`, `blur-sm`, `drop-shadow-sm`, `backdrop-blur-sm` https://tailwindcss.com/docs/upgrade-guide#updated-shadow-radius-and-blur-scales |
| `ring` (3px, blue) | `ring-3 ring-blue-500` https://tailwindcss.com/docs/upgrade-guide#default-ring-width-and-color |
| `ring-opacity-*` | opacity modifiers like `ring-black/50` https://tailwindcss.com/docs/upgrade-guide#removed-deprecated-utilities |
