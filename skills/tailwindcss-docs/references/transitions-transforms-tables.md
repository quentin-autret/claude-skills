# Transitions, animations, transforms and tables

## Contents

- Transitions
- Discrete transitions and entry styles
- Animations
- Transforms
- 3D
- Zoom
- Tables
- Old patterns

## Transitions

- `transition` is not `all`. It lists color, background, border, outline and decoration colors, fill, stroke, gradient stops, opacity, box-shadow, transform, translate, scale, rotate, filter, backdrop-filter, display, content-visibility, overlay and pointer-events. https://tailwindcss.com/docs/transition-property
- Every `transition-*` class except `transition-none` also sets the default duration (150ms, `--default-transition-duration`) and easing (`--default-transition-timing-function`). Add `duration-*` or `ease-*` only to override. https://tailwindcss.com/docs/transition-property
- `transition-transform` covers `transform`, `translate`, `scale` and `rotate`. `transition-colors` covers gradient stops. https://tailwindcss.com/docs/transition-property
- `duration-<n>` and `delay-<n>` are milliseconds (`duration-<number>`, `delay-<number>`). Lists: `duration-[1s,15s]`, `delay-[1s,250ms]`. https://tailwindcss.com/docs/transition-duration
- `ease-in`, `ease-out`, `ease-in-out` read `--ease-*`. `ease-linear` is `linear`. https://tailwindcss.com/docs/transition-timing-function#customizing-your-theme
- Reduced motion: `motion-reduce:transition-none` (https://tailwindcss.com/docs/transition-property#supporting-reduced-motion), `motion-reduce:duration-0` (https://tailwindcss.com/docs/transition-duration#supporting-reduced-motion).

## Discrete transitions and entry styles

- `transition-discrete` (`transition-behavior: allow-discrete`) lets discrete properties like `display` transition: https://tailwindcss.com/docs/transition-behavior#basic-example

```html
<div class="hidden transition-all transition-discrete not-peer-has-checked:opacity-0 peer-has-checked:block">
```

- `starting:` sets the first-render state (`@starting-style`): `opacity-0 starting:open:opacity-0` on a popover. https://tailwindcss.com/docs/hover-focus-and-other-states#starting-style

## Animations

- `animate-spin`, `animate-ping`, `animate-pulse`, `animate-bounce` read `--animate-*`. Arbitrary: `animate-[wiggle_1s_ease-in-out_infinite]`. https://tailwindcss.com/docs/animation
- Custom animation: `--animate-wiggle` and its `@keyframes` together inside `@theme`: https://tailwindcss.com/docs/animation#customizing-your-theme

```css
@theme {
  --animate-wiggle: wiggle 1s ease-in-out infinite;
  @keyframes wiggle {
    0%, 100% { transform: rotate(-3deg); }
    50% { transform: rotate(3deg); }
  }
}
```

- Opt in to motion: `motion-safe:animate-spin`. https://tailwindcss.com/docs/animation#supporting-reduced-motion

## Transforms

- `rotate-*`, `scale-*` and `translate-*` set the CSS `rotate`, `scale` and `translate` properties, not `transform`. `rotate-<number>` is degrees, `scale-<number>` is a percentage. https://tailwindcss.com/docs/rotate
- Reset them with `rotate-none`, `scale-none`, `translate-none`. `transform-none` does not reset them. https://tailwindcss.com/docs/upgrade-guide#individual-transform-properties
- In an arbitrary transition list, name the individual properties: `transition-[opacity,scale]`. https://tailwindcss.com/docs/upgrade-guide#individual-transform-properties
- `translate-<n>` follows the spacing scale (https://tailwindcss.com/docs/translate#using-the-spacing-scale). `translate-<fraction>` such as `translate-1/4` and `translate-full` are percentages of the element's size (https://tailwindcss.com/docs/translate#using-a-percentage).
- Negative scale mirrors: `-scale-x-75`. https://tailwindcss.com/docs/scale#using-negative-values
- `skew-*` still uses `transform`. https://tailwindcss.com/docs/skew
- The transform page lists no class that enables transforms. `transform-gpu` forces GPU rendering, `transform-cpu` undoes it, `transform-[matrix(1,2,3,4,5,6)]` sets a raw value. https://tailwindcss.com/docs/transform
- Origin uses the `origin-` prefix: `origin-top-left`, `origin-(<custom-property>)`, `origin-[<value>]`. https://tailwindcss.com/docs/transform-origin#using-a-custom-value

## 3D

- `transform-3d` (`transform-style: preserve-3d`) positions children in 3D space. `transform-flat` is `transform-style: flat`. https://tailwindcss.com/docs/transform-style#basic-example
- `rotate-x-50 -rotate-y-30 rotate-z-45`, `scale-z-*`, `scale-3d`, `translate-z-*` (needs `transform-3d` on the parent). https://tailwindcss.com/docs/translate#translating-on-the-z-axis https://tailwindcss.com/docs/rotate#rotating-in-3d-space https://tailwindcss.com/docs/scale
- Perspective goes on the parent. Presets: `perspective-dramatic` 100px, `-near` 300px, `-normal` 500px, `-midrange` 800px, `-distant` 1200px (each is `var(--perspective-*)`). `perspective-none` removes it. The page compares it to moving a camera closer to or further from an object. https://tailwindcss.com/docs/perspective#basic-example
- `perspective-origin-top-left`, `perspective-origin-[200%_150%]`. https://tailwindcss.com/docs/perspective-origin#using-a-custom-value

## Zoom

- `zoom-125` is `zoom: 125%`, works with `hover:zoom-125`. Arbitrary `zoom-[1.1]`. https://tailwindcss.com/docs/zoom#using-a-custom-value

## Tables

- `border-spacing-<n>` follows the spacing scale and controls the space between cell borders of tables with separate borders (`border-separate`). `border-spacing-x-*` and `-y-*` combine. https://tailwindcss.com/docs/border-spacing#basic-example
- `border-collapse` also collapses the border of `<table>` itself. https://tailwindcss.com/docs/border-collapse#collapsing-table-borders
- With `table-fixed`, first-row widths set column widths. Columns without an explicit width divide the remaining width evenly. https://tailwindcss.com/docs/table-layout#using-fixed-column-widths

## Old patterns

| v3 | v4 |
| :--- | :--- |
| `transform-none` to reset a scale | `scale-none` https://tailwindcss.com/docs/upgrade-guide#individual-transform-properties |
| `transition-[opacity,transform]` for scale | `transition-[opacity,scale]` https://tailwindcss.com/docs/upgrade-guide#individual-transform-properties |