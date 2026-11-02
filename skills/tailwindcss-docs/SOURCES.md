# Sources

- Technology: Tailwind CSS
- Version covered: v4.x, latest (the docs site is built against tailwindcss ^4.3.3 per its package.json)
- Documentation: https://tailwindcss.com/docs
- Docs repository: https://github.com/tailwindlabs/tailwindcss.com (branch `main`, `src/docs/*.mdx` and `src/app/(docs)/docs/installation/`)
- Fetched: 2026-11-02
- License: the documentation belongs to Tailwind Labs and is distributed under its own license. This skill summarizes it and links back to each page. It does not reproduce it.

Each page below was read from its source file in the docs repository. A page at `src/docs/<slug>.mdx` is published at `https://tailwindcss.com/docs/<slug>`.

## Installation

- https://tailwindcss.com/docs/installation/using-vite
- https://tailwindcss.com/docs/installation/using-postcss
- https://tailwindcss.com/docs/installation/tailwind-cli
- https://tailwindcss.com/docs/installation/play-cdn
- https://tailwindcss.com/docs/installation/framework-guides
- https://tailwindcss.com/docs/editor-setup
- https://tailwindcss.com/docs/compatibility

## Framework guides

Base: https://tailwindcss.com/docs/installation/framework-guides/

- nextjs, react-router, tanstack-start, sveltekit, nuxt, solidjs, astro, qwik, angular, emberjs, gatsby, meteor, parcel, rspack (tabs `rspack/react`, `rspack/vue`), laravel (tabs `laravel/vite`, `laravel/mix`), symfony, ruby-on-rails, phoenix, adonisjs

The guide list in the docs source registers Nuxt under the slug `nuxt`. Tab URLs are derived from the link logic of the index page.

## Upgrade guide

- https://tailwindcss.com/docs/upgrade-guide

## Core concepts

- https://tailwindcss.com/docs/styling-with-utility-classes
- https://tailwindcss.com/docs/hover-focus-and-other-states
- https://tailwindcss.com/docs/responsive-design
- https://tailwindcss.com/docs/dark-mode
- https://tailwindcss.com/docs/theme
- https://tailwindcss.com/docs/colors
- https://tailwindcss.com/docs/adding-custom-styles
- https://tailwindcss.com/docs/detecting-classes-in-source-files
- https://tailwindcss.com/docs/functions-and-directives
- https://tailwindcss.com/docs/preflight

## Utility pages

All at `https://tailwindcss.com/docs/<slug>`.

- Layout: aspect-ratio, columns, break-after, break-before, break-inside, box-decoration-break, box-sizing, display, float, clear, isolation, object-fit, object-position, overflow, overscroll-behavior, position, top-right-bottom-left, visibility, z-index
- Flexbox and grid: flex-basis, flex-direction, flex-wrap, flex, flex-grow, flex-shrink, order, grid-template-columns, grid-column, grid-template-rows, grid-row, grid-auto-flow, grid-auto-columns, grid-auto-rows, gap, justify-content, justify-items, justify-self, align-content, align-items, align-self, place-content, place-items, place-self
- Spacing and sizing: padding, margin, width, min-width, max-width, height, min-height, max-height, inline-size, min-inline-size, max-inline-size, block-size, min-block-size, max-block-size
- Typography: font-family, font-size, font-smoothing, font-style, font-weight, font-stretch, font-variant-numeric, font-feature-settings, letter-spacing, line-clamp, line-height, list-style-image, list-style-position, list-style-type, text-align, color, text-decoration-line, text-decoration-color, text-decoration-style, text-decoration-thickness, text-underline-offset, text-transform, text-overflow, text-wrap, text-indent, tab-size, vertical-align, white-space, word-break, overflow-wrap, hyphens, content
- Backgrounds and borders: background-attachment, background-clip, background-color, background-image, background-origin, background-position, background-repeat, background-size, border-radius, border-width, border-color, border-style, outline-width, outline-color, outline-style, outline-offset
- Effects: box-shadow, text-shadow, opacity, mix-blend-mode, background-blend-mode, mask-clip, mask-composite, mask-image, mask-mode, mask-origin, mask-position, mask-repeat, mask-size, mask-type
- Filters: filter, filter-blur, filter-brightness, filter-contrast, filter-drop-shadow, filter-grayscale, filter-hue-rotate, filter-invert, filter-saturate, filter-sepia, backdrop-filter, backdrop-filter-blur, backdrop-filter-brightness, backdrop-filter-contrast, backdrop-filter-grayscale, backdrop-filter-hue-rotate, backdrop-filter-invert, backdrop-filter-opacity, backdrop-filter-saturate, backdrop-filter-sepia
- Tables, transitions and transforms: border-collapse, border-spacing, table-layout, caption-side, transition-property, transition-behavior, transition-duration, transition-timing-function, transition-delay, animation, backface-visibility, perspective, perspective-origin, rotate, scale, skew, transform, transform-origin, transform-style, translate, zoom
- Interactivity, SVG and accessibility: accent-color, appearance, caret-color, color-scheme, cursor, field-sizing, pointer-events, resize, scroll-behavior, scrollbar-color, scrollbar-width, scrollbar-gutter, scroll-margin, scroll-padding, scroll-snap-align, scroll-snap-stop, scroll-snap-type, touch-action, user-select, will-change, fill, stroke, stroke-width, forced-color-adjust
