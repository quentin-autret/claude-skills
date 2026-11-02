# Interactivity, scrolling, SVG and accessibility

## Contents

- Forms and native controls
- Pointer, cursor, selection
- Scrollbars
- Scroll snap, margin and padding
- will-change
- SVG fill and stroke
- Forced colors and screen readers

## Forms and native controls

- `scheme-light dark:scheme-dark` on `html` makes native controls, date inputs and scrollbars follow dark mode. Other values: `scheme-normal`, `scheme-light-dark`, `scheme-only-dark`, `scheme-only-light`. https://tailwindcss.com/docs/color-scheme#applying-in-dark-mode
- `field-sizing-content` sizes a textarea or input to its content. `field-sizing-fixed` keeps a fixed size. https://tailwindcss.com/docs/field-sizing#sizing-based-on-content Browser support for `field-sizing: content` is limited. https://tailwindcss.com/docs/compatibility#browser-support
- `appearance-none` for custom form controls, `appearance-auto` to bring native controls back, for example in some accessibility modes. https://tailwindcss.com/docs/appearance#restoring-default-appearance
- `accent-pink-500` recolors checkboxes and radios. An opacity modifier on it only works in Firefox. https://tailwindcss.com/docs/accent-color#changing-the-opacity
- `caret-pink-500` colors the text cursor. https://tailwindcss.com/docs/caret-color
- `resize` alone is `resize: both`. `resize-x`, `resize-y`, `resize-none`. https://tailwindcss.com/docs/resize

## Pointer, cursor, selection

- `pointer-events-none` lets events pass through to elements beneath, while child elements still receive them. https://tailwindcss.com/docs/pointer-events#ignoring-pointer-events
- Arbitrary cursor: `cursor-[url(hand.cur),_pointer]`. https://tailwindcss.com/docs/cursor#using-a-custom-value
- `select-all` selects all text on click. `select-none` also applies to children. https://tailwindcss.com/docs/user-select
- `touch-manipulation`, `touch-pan-x`, `touch-pinch-zoom` and the other `touch-*` values. https://tailwindcss.com/docs/touch-action

## Scrollbars

- `scrollbar-thumb-<color>` and `scrollbar-track-<color>` share one `scrollbar-color` declaration. Set both: an unset one is transparent. https://tailwindcss.com/docs/scrollbar-color#setting-the-scrollbar-color

```html
<div class="overflow-auto scrollbar-thumb-slate-900/60 scrollbar-track-slate-100 hover:scrollbar-thumb-sky-500">
```

- `scrollbar-thin`, `scrollbar-none` (still scrolls), `scrollbar-auto`. No arbitrary widths. https://tailwindcss.com/docs/scrollbar-width#hiding-scrollbars
- `scrollbar-gutter-stable` reserves the gutter even without overflow. `scrollbar-gutter-both` mirrors it on both edges. https://tailwindcss.com/docs/scrollbar-gutter
- `scroll-smooth` only affects scrolling triggered by the browser. https://tailwindcss.com/docs/scroll-behavior#using-smooth-scrolling

## Scroll snap, margin and padding

- `snap-mandatory` and `snap-proximity` only set strictness. Pair them with an axis: `snap-x snap-mandatory`. https://tailwindcss.com/docs/scroll-snap-type
- Children need an alignment (`snap-start`, `snap-center`, `snap-end`), or nothing snaps. https://tailwindcss.com/docs/scroll-snap-type#horizontal-scroll-snapping
- `snap-always` with `snap-mandatory` forces a stop on each item: https://tailwindcss.com/docs/scroll-snap-stop

```html
<div class="snap-x snap-mandatory overflow-x-auto">
  <div class="snap-center snap-always">...</div>
</div>
```

- `snap-align-none` removes the child alignment. `snap-none` removes the container snapping. https://tailwindcss.com/docs/scroll-snap-align
- `scroll-mt-<n>` and `scroll-p-<n>` follow the spacing scale, with negative `-scroll-mt-4`. Logical forms: `scroll-ms`, `scroll-me`, `scroll-mbs`, `scroll-mbe`, `scroll-ps`, `scroll-pbs`. https://tailwindcss.com/docs/scroll-margin#using-logical-properties

## will-change

- A last resort for a known performance problem. Apply `will-change-transform` just before the change, then `will-change-auto` after. Overuse slows the page. https://tailwindcss.com/docs/will-change#optimizing-with-will-change

## SVG fill and stroke

- `fill-current` and `stroke-current` follow the text color, so an icon inside `hover:text-white` changes with it. https://tailwindcss.com/docs/fill#using-the-current-color
- `stroke-<n>` is the stroke width. A variable width needs the hint `stroke-(length:--my-stroke-width)`. `stroke-(--x)` is a color. https://tailwindcss.com/docs/stroke-width#using-a-custom-value
- `fill-none`, `stroke-none`. https://tailwindcss.com/docs/fill

## Forced colors and screen readers

- `forced-colors:` styles forced-colors mode. `forced-color-adjust-none` opts an element out when the limited palette hurts usability, for example color swatches. https://tailwindcss.com/docs/forced-color-adjust#opting-out-of-forced-colors
- Swatch picker pattern: `sr-only` radio inputs and label text, visual swatches with `aria-hidden`. https://tailwindcss.com/docs/forced-color-adjust#opting-out-of-forced-colors
- `sr-only` uses `clip-path: inset(50%)`. Undo with `not-sr-only`. https://tailwindcss.com/docs/display#screen-reader-only
- Hide a focus outline with `outline-hidden`, which stays visible in forced-colors mode, and add your own focus style. https://tailwindcss.com/docs/outline-style#hiding-an-outline
