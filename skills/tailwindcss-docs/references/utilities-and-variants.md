# Utilities and variants

## Contents

- Arbitrary values and inline styles
- Conflicts, important and duplication
- State variants
- Parent, sibling and child selectors
- Media and feature variants
- Attribute variants
- Responsive design
- Container queries
- Dark mode
- Old patterns

## Arbitrary values and inline styles

- Arbitrary values: `bg-[#316ff6]`, `grid-cols-[24rem_2.5rem_minmax(0,1fr)]` (underscore is a space), `max-h-[calc(100dvh-(--spacing(6)))]`. https://tailwindcss.com/docs/styling-with-utility-classes#using-arbitrary-values and, for the underscore, https://tailwindcss.com/docs/adding-custom-styles#handling-whitespace
- CSS variable shorthand uses parentheses: `bg-(--bg-color) hover:bg-(--bg-color-hover)`. https://tailwindcss.com/docs/styling-with-utility-classes#when-to-use-inline-styles
- Arbitrary properties take variants: `[--scroll-offset:56px] lg:[--scroll-offset:44px]`, `hover:[mask-type:alpha]`. https://tailwindcss.com/docs/adding-custom-styles#arbitrary-properties
- Values from a database or API: set a CSS variable inline and read it with a utility. https://tailwindcss.com/docs/styling-with-utility-classes#when-to-use-inline-styles

```jsx
<button style={{ "--bg-color": buttonColor }} className="bg-(--bg-color)">
```


## Conflicts, important and duplication

- Of 2 utilities on the same property, the later one in the stylesheet wins, not in `class`: `class="grid flex"` gives `display: grid`. Never put 2 conflicting classes on one element. https://tailwindcss.com/docs/styling-with-utility-classes#conflicting-utility-classes
- Important: trailing `!`, `bg-red-500!`. For all utilities: `@import "tailwindcss" important;`. https://tailwindcss.com/docs/styling-with-utility-classes#using-the-important-modifier
- Duplication: prefer loops, multi-cursor editing, then components or template partials over a CSS class (custom CSS is fine when a partial feels heavy-handed). https://tailwindcss.com/docs/styling-with-utility-classes#managing-duplication

## State variants

- Form state: `required`, `invalid`, `disabled`, `enabled`, `checked`, `indeterminate`, `default`, `optional`, `valid`, `user-valid`, `user-invalid` (only after user interaction), `in-range`, `out-of-range`, `placeholder-shown`, `autofill`, `read-only`. https://tailwindcss.com/docs/hover-focus-and-other-states#required-and-disabled
- `nth-<number>`, `nth-last-<number>`, `nth-of-type-*`, and arbitrary `nth-[2n+1_of_li]`. https://tailwindcss.com/docs/hover-focus-and-other-states#first-last-odd-and-even
- `not-*` negates any variant: `hover:not-focus:bg-indigo-700`, `not-supports-[display:grid]:flex`. https://tailwindcss.com/docs/hover-focus-and-other-states#not
- `open` matches `[open]` on `<details>` and `<dialog>` and `:popover-open`. https://tailwindcss.com/docs/hover-focus-and-other-states#open-closed-state
- `starting:` maps to `@starting-style`, the state on first render or when leaving `display: none`: `opacity-0 starting:open:opacity-0` on a popover. https://tailwindcss.com/docs/hover-focus-and-other-states#starting-style
- `before:` and `after:` add `content: ''` automatically. A real element is usually simpler. https://tailwindcss.com/docs/hover-focus-and-other-states#before-and-after
- `selection:bg-pink-300` is inherited: put it on an ancestor for a site-wide color. https://tailwindcss.com/docs/hover-focus-and-other-states#selection
- Other pseudo-elements: `placeholder`, `file`, `marker`, `first-line`, `first-letter`, `backdrop`. https://tailwindcss.com/docs/hover-focus-and-other-states#pseudo-elements

## Parent, sibling and child selectors

- `group` and `group-hover:`, named groups `group/item` with `group-hover/item:`. Arbitrary `group-[.is-published]:block`. https://tailwindcss.com/docs/hover-focus-and-other-states#differentiating-nested-groups and https://tailwindcss.com/docs/hover-focus-and-other-states#arbitrary-groups
- `in-focus:opacity-100` reacts to any ancestor, no `group` class needed. Use `group` for finer control. https://tailwindcss.com/docs/hover-focus-and-other-states#implicit-groups
- `has-[:focus]`, `has-[img]`, plus `group-has-*` and `peer-has-*`. https://tailwindcss.com/docs/hover-focus-and-other-states#has
- `peer` only works on previous siblings: put the `peer` element before the one styled with `peer-invalid:`, `peer-focus:` or `peer-[.is-dirty]:`. https://tailwindcss.com/docs/hover-focus-and-other-states#styling-based-on-sibling-state
- `*:` targets direct children. A class on the child cannot override it: `<ul class="*:bg-sky-50"><li class="bg-red-50">` stays sky. Prefer classes on the children when you control them. https://tailwindcss.com/docs/hover-focus-and-other-states#styling-direct-children
- `**:` targets all descendants, narrowed by another variant: `**:data-avatar:size-12`. https://tailwindcss.com/docs/hover-focus-and-other-states#styling-all-descendants
- Arbitrary variants: `[&.is-dragging]:cursor-grabbing`, `[&_p]:mt-4`, `[&>[data-active]+span]:text-blue-600`, at-rules `[@supports(display:grid)]:grid`. https://tailwindcss.com/docs/hover-focus-and-other-states#using-arbitrary-variants
- A reused selector becomes a variant: `@custom-variant theme-midnight (&:where([data-theme="midnight"] *));`. https://tailwindcss.com/docs/hover-focus-and-other-states#registering-a-custom-variant

## Media and feature variants

- `hover` only applies when the primary input device supports hover (`@media (hover: hover)`); `group-hover` compiles the same way. Restore the old behavior with `@custom-variant hover (&:hover);`. https://tailwindcss.com/docs/upgrade-guide#hover-styles-on-mobile and https://tailwindcss.com/docs/styling-with-utility-classes#complex-selectors
- `motion-safe:` to opt in to motion, rather than undoing many styles with `motion-reduce:`. https://tailwindcss.com/docs/hover-focus-and-other-states#prefers-reduced-motion
- `contrast-more`, `forced-colors`, `inverted-colors`, `portrait`, `print`, `noscript` (`scripting: none`). https://tailwindcss.com/docs/hover-focus-and-other-states#forced-colors
- `pointer-coarse:` for larger touch targets (primary device). `any-pointer-*` checks any connected device. https://tailwindcss.com/docs/hover-focus-and-other-states#pointer-and-any-pointer
- `supports-[display:grid]:grid`, property-only shorthand `supports-backdrop-filter:`. https://tailwindcss.com/docs/hover-focus-and-other-states#supports

## Attribute variants

- `aria-*` boolean variants: `aria-busy`, `aria-checked`, `aria-disabled`, `aria-expanded`, `aria-hidden`, `aria-pressed`, `aria-readonly`, `aria-required`, `aria-selected`. Others: `aria-[sort=ascending]:`, or `@custom-variant aria-asc (&[aria-sort="ascending"]);`. https://tailwindcss.com/docs/hover-focus-and-other-states#aria-states
- `data-active:` checks presence (`<div data-active>`). `data-[size=large]:p-8` checks a value. https://tailwindcss.com/docs/hover-focus-and-other-states#data-attributes
- `rtl:` and `ltr:` only when the site supports both directions. https://tailwindcss.com/docs/hover-focus-and-other-states#rtl-support

## Responsive design

- Add `<meta name="viewport" content="width=device-width, initial-scale=1.0" />`. https://tailwindcss.com/docs/responsive-design#overview
- Breakpoints are min-width, `@media (width >= 40rem)`: sm 40rem, md 48rem, lg 64rem, xl 80rem, 2xl 96rem. https://tailwindcss.com/docs/responsive-design#overview
- Mobile first: `sm:text-center` does not target phones. Write `text-center sm:text-left`. https://tailwindcss.com/docs/responsive-design#targeting-mobile-screens
- Ranges with `max-*`: `md:max-xl:flex`, a single breakpoint `md:max-lg:flex`. https://tailwindcss.com/docs/responsive-design#targeting-a-breakpoint-range
- One-off breakpoints: `min-[320px]:text-center`, `max-[600px]:bg-sky-300`. https://tailwindcss.com/docs/responsive-design#using-arbitrary-values
- Custom breakpoints: `@theme { --breakpoint-xs: 30rem; --breakpoint-3xl: 120rem; }`, same unit as the defaults. Remove one with `--breakpoint-2xl: initial;`, all with `--breakpoint-*: initial;`. https://tailwindcss.com/docs/responsive-design#removing-default-breakpoints

## Container queries

- Mark the parent `@container`, style children with `@sm:`, `@md:`, mobile first. https://tailwindcss.com/docs/responsive-design#container-queries
- `@max-md:`, ranges `@sm:@max-md:flex-col`, named `@container/main` with `@sm/main:`. https://tailwindcss.com/docs/responsive-design#named-containers
- `@container` is inline-size only. Use `@container-size` for block units like `h-[50cqb]`. https://tailwindcss.com/docs/responsive-design#using-size-containers
- Sizes: `@3xs` 16rem, `@xs` 20rem, `@sm` 24rem, `@md` 28rem, `@lg` 32rem, `@xl` 36rem, `@2xl` 42rem up to `@7xl` 80rem. Add `--container-8xl: 96rem` for `@8xl:`, or use `@min-[475px]:`. https://tailwindcss.com/docs/responsive-design#container-size-reference

## Dark mode

- `dark:` follows `prefers-color-scheme` by default. https://tailwindcss.com/docs/dark-mode#overview
- Class toggle: `@custom-variant dark (&:where(.dark, .dark *));` with `<html class="dark">`. https://tailwindcss.com/docs/dark-mode#toggling-dark-mode-manually
- Data attribute: `@custom-variant dark (&:where([data-theme=dark], [data-theme=dark] *));`. https://tailwindcss.com/docs/dark-mode#using-a-data-attribute
- Light, dark and system: an inline `<head>` script toggles `dark` from `localStorage.theme` and `matchMedia`, to avoid a flash. https://tailwindcss.com/docs/dark-mode#with-system-theme-support

## Old patterns

| v3 | v4 |
| :--- | :--- |
| leading `!bg-red-500` | trailing `bg-red-500!` (the old way is deprecated but still supported) https://tailwindcss.com/docs/upgrade-guide#the-important-modifier |
| `bg-[--x]` | `bg-(--x)` https://tailwindcss.com/docs/upgrade-guide#variables-in-arbitrary-values |
| stacked variants applied right to left | applied left to right: `first:*:pt-0` becomes `*:first:pt-0` https://tailwindcss.com/docs/upgrade-guide#variant-stacking-order |
