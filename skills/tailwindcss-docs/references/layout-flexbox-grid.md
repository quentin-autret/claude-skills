# Layout, flexbox and grid

## Contents

- Value syntax shared by these utilities
- Display, visibility, position
- Columns, aspect ratio, overflow
- Flex
- Grid
- Alignment and safe alignment
- Old patterns

## Value syntax shared by these utilities

- Numeric utilities take a `<number>` instead of a fixed list, for example `grid-cols-<n>`, `col-span-<n>`, `row-start-<n>`, `order-<n>`, `grow-<n>`, `flex-<n>`, `z-<n>`. https://tailwindcss.com/docs/grid-template-columns
- Spacing-based values compile to `calc(var(--spacing) * <n>)`: `gap-*`, `basis-<n>`, `auto-cols-<n>`, `top-*`, `inset-*`. https://tailwindcss.com/docs/gap
- Negative values take a leading dash: `-order-1`, `-col-start-1`, `-z-10`, `-top-4`. https://tailwindcss.com/docs/order#using-negative-values
- Variable shorthand uses parentheses, `grid-cols-(<custom-property>)`, with the property name in place of `<custom-property>`. Arbitrary values use brackets and underscores, `grid-cols-[200px_minmax(900px,_1fr)_100px]`. https://tailwindcss.com/docs/grid-template-columns#using-a-custom-value

## Display, visibility, position

- `flow-root` creates a block formatting context. `contents` makes the children act as children of the parent. https://tailwindcss.com/docs/display#flow-root
- `sr-only sm:not-sr-only`: visually hidden, kept for screen readers. https://tailwindcss.com/docs/display#screen-reader-only
- `hidden` (`display: none`, see https://tailwindcss.com/docs/display#hidden) removes the element from the document. `invisible` hides it and keeps its space. `collapse` hides table rows or columns without resizing the table. https://tailwindcss.com/docs/visibility#collapsing-elements
- `box-border` sets `box-sizing: border-box`, which Tailwind makes the default for all elements in its Preflight base styles. `box-content` sets `content-box`. https://tailwindcss.com/docs/box-sizing#including-borders-and-padding
- `isolate` explicitly creates a new stacking context. `isolation-auto` undoes it. https://tailwindcss.com/docs/isolation#basic-example
- Logical inset: `inset-s-*` (inline start), `inset-e-*`, `inset-bs-*` (block start), `inset-be-*`. `inset-x` and `inset-y` map to `inset-inline` and `inset-block`. https://tailwindcss.com/docs/top-right-bottom-left#using-logical-properties
- Logical floats: `float-start` and `float-end` map to left or right based on the text direction (`float: inline-start` / `inline-end`). `clear-start` and `clear-end` are `clear: inline-start` / `inline-end`. https://tailwindcss.com/docs/float#using-logical-properties
- `overscroll-contain` stops scroll chaining and keeps the bounce. `overscroll-none` removes both. https://tailwindcss.com/docs/overscroll-behavior#preventing-parent-overscrolling

## Columns, aspect ratio, overflow

- `columns-3` fixes the count. `columns-xs` (from the `--container-*` scale) sets an ideal width and lets the count adapt. Spacing between columns uses `gap-*`. https://tailwindcss.com/docs/columns#setting-by-number
- Inside columns: `break-inside-avoid-column`, `break-after-column`, `break-before-page`. https://tailwindcss.com/docs/break-inside
- Ratios are bare fractions: `aspect-3/2`. `aspect-video` reads `--aspect-video`. Add `--aspect-retro: 4 / 3` for `aspect-retro`. https://tailwindcss.com/docs/aspect-ratio#customizing-your-theme
- `object-[25%_75%]` with underscores. https://tailwindcss.com/docs/object-position#using-a-custom-value
- `overflow-auto` shows scrollbars only when needed. `overflow-scroll` always does, except on systems such as macOS that hide them. https://tailwindcss.com/docs/overflow#scrolling-in-all-directions

## Flex

- `flex-1` grows and shrinks ignoring the initial size. `flex-auto` takes it into account. `flex-initial` shrinks without growing. `flex-none` does neither. https://tailwindcss.com/docs/flex#basic-example
- `flex-<fraction>` gives `flex: calc(<fraction> * 100%)`. Arbitrary: `flex-[3_1_auto]`. https://tailwindcss.com/docs/flex
- Grow and shrink are `grow`, `grow-0`, `grow-3`, `shrink`, `shrink-0`. https://tailwindcss.com/docs/flex-grow
- `basis-1/3` is a percentage. `basis-3xs` to `basis-7xl` come from `--container-*`, not the spacing scale. https://tailwindcss.com/docs/flex-basis#using-the-container-scale
- `order-first` is `-9999`, `order-last` is `9999`. Order applies to flex and grid items. https://tailwindcss.com/docs/order
- `flex-nowrap` lets inflexible items overflow. https://tailwindcss.com/docs/flex-wrap#dont-wrap

## Grid

- `grid-cols-<n>` is `grid-template-columns: repeat(<n>, minmax(0, 1fr))`. `auto-cols-fr` is `grid-auto-columns: minmax(0, 1fr)`. https://tailwindcss.com/docs/grid-template-columns
- Subgrid: `grid-cols-subgrid`, `grid-rows-subgrid`. https://tailwindcss.com/docs/grid-template-columns#implementing-a-subgrid
- `col-span-full` is `1 / -1`. Combine `col-start-<n>` with `col-span-<n>`. `col-<n>` sets `grid-column: <n>`. Arbitrary `col-[16_/_span_16]`. https://tailwindcss.com/docs/grid-column
- Rows mirror columns: `row-span-*`, `row-start-*`, `row-[span_16_/_span_16]`. https://tailwindcss.com/docs/grid-row
- `grid grid-flow-col auto-cols-max` sizes implicit tracks. https://tailwindcss.com/docs/grid-auto-columns#basic-example
- `gap-x-*` sets `column-gap`, `gap-y-*` sets `row-gap`. Gap works for flex too. https://tailwindcss.com/docs/gap

## Alignment and safe alignment

- `-safe` variants compile to the `safe` keyword (`justify-center-safe` is `justify-content: safe center`). For the `justify-*` family the docs state that when there is not enough space, items align to the start of the container instead of the center or end. Variants in the tables: `justify-center-safe`, `justify-end-safe`, `items-center-safe`, `items-end-safe`, `self-center-safe`, `justify-items-center-safe`, `justify-self-end-safe`, `place-content-center-safe`, `place-items-end-safe`, `place-self-center-safe`. https://tailwindcss.com/docs/justify-content#center
- `items-baseline-last` and `self-baseline-last` align on the last baseline in the container. https://tailwindcss.com/docs/align-items#last-baseline
- `align-content` uses the `content-` prefix: `content-between`, `content-stretch`. It has no `-safe` form. https://tailwindcss.com/docs/align-content
- `justify-stretch` stretches only auto-sized items. `justify-normal` resets. https://tailwindcss.com/docs/justify-content#stretch
- `justify-items-*` and `justify-self-*` apply to grid items. https://tailwindcss.com/docs/justify-items

## Old patterns

| v3 | v4 |
| :--- | :--- |
| `flex-grow`, `flex-shrink-0` | `grow`, `shrink-0` https://tailwindcss.com/docs/upgrade-guide#removed-deprecated-utilities |
| variable in brackets, e.g. `grid-cols-[--x]` (the upgrade guide shows `bg-[--brand-color]`) | `grid-cols-(--x)` https://tailwindcss.com/docs/upgrade-guide#variables-in-arbitrary-values |
| commas in `grid-cols-[max-content,auto]` | underscores `grid-cols-[max-content_auto]` https://tailwindcss.com/docs/upgrade-guide#arbitrary-values-in-grid-and-object-position-utilities |
| `overflow-ellipsis` | `text-ellipsis` https://tailwindcss.com/docs/upgrade-guide#removed-deprecated-utilities |
