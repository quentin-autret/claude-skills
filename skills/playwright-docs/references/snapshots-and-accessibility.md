# Snapshots, visual comparison and accessibility scans

Aria snapshots, screenshot comparison, snapshot update modes and axe scans in Playwright Test 1.64.1.

## Contents

- Snapshot update modes
- Aria snapshots
- Visual comparisons
- Non-image snapshots and plain screenshots
- Accessibility scans with axe

## Snapshot update modes

`--update-snapshots` (`-u`) and the `updateSnapshots` config option take one mode.
https://playwright.dev/docs/aria-snapshots#updating-snapshots-with-playwrighttest-and-the---update-snapshots-flag

| Mode | Effect |
| --- | --- |
| `default` (no flag) | writes missing snapshots, and the tests that wrote them FAIL |
| `missing` | writes missing snapshots, and the run passes |
| `changed` (`-u` with no value) | rewrites snapshots that do not match |
| `all` | rewrites every snapshot, unchanged ones included |
| `none` | never writes |

https://playwright.dev/docs/test-cli#all-options and https://playwright.dev/docs/release-notes#version-164

- `--update-snapshots=missing` lets one CI run create new snapshots and verify existing ones.
  https://playwright.dev/docs/release-notes#version-164
- Inline aria templates are not edited in place by default. `--update-source-method` picks `patch` (default, a unified diff for `git apply`), `3way` (conflict markers) or `overwrite`. Separate `.aria.yml` files are written directly.
  ```bash
  npx playwright test --update-snapshots --update-source-method=3way
  ```
  https://playwright.dev/docs/aria-snapshots#snapshot-patch-files
- `--ignore-snapshots` skips snapshot expectations.
  https://playwright.dev/docs/test-cli#all-options

## Aria snapshots

`toMatchAriaSnapshot` matches the accessibility tree, as YAML, against a template. The template is a constraint, not a full dump. It works on a page and on a locator.
```js
await expect(page.locator('body')).toMatchAriaSnapshot(`
  - banner:
    - heading /Playwright enables reliable/ [level=1]
`);
```
https://playwright.dev/docs/release-notes#version-149 and https://playwright.dev/docs/aria-snapshots#overview

- Node syntax: `- role "name" [attribute=value]`. A quoted name is exact, `/regex/` is a pattern. Attributes: `checked`, `disabled`, `expanded`, `invalid`, `level`, `pressed`, `selected`.
  https://playwright.dev/docs/aria-snapshots#aria-snapshots
- Matching is case-sensitive, whitespace-normalized and order-sensitive. A single-node template matches at any depth. Several top-level nodes must be siblings, in order.
  https://playwright.dev/docs/aria-snapshots#snapshot-matching
- Partial matching: omit name or attributes to ignore them (`- button`). Omit list items to match a subset.
  https://playwright.dev/docs/aria-snapshots#partial-matching
- Children mode defaults to `contain`: listed children present in order, extras allowed. `/children: equal` requires exact direct children, `deep-equal` also nested ones. Under `equal`, an empty child list means no children allowed.
  ```yaml
  - list:
    - /children: equal
    - listitem: Feature A
    - listitem: Feature B
  ```
  https://playwright.dev/docs/aria-snapshots#strict-matching
- Global children mode:
  ```ts
  export default defineConfig({ expect: { toMatchAriaSnapshot: { children: 'equal' } } });
  ```
  https://playwright.dev/docs/aria-snapshots#setting-children-mode-globally
- Links: `- /url: /https://example.com/.*/`. Dynamic text: `- heading /Issues \d+/`.
  https://playwright.dev/docs/aria-snapshots#matching-with-regular-expressions
- `<p>Line 1<br>Line 2</p>` renders as `- paragraph: Line 1 Line 2`. `aria-invalid` renders `[invalid]`, `[invalid=grammar]` or `[invalid=spelling]`.
  https://playwright.dev/docs/aria-snapshots#accessibility-tree-examples
- An empty template generates the snapshot even without `-u`: `await expect(locator).toMatchAriaSnapshot('');`
  https://playwright.dev/docs/aria-snapshots#empty-template-for-snapshot-generation
- Separate file: `toMatchAriaSnapshot({ name: 'main.aria.yml' })`, stored in `<spec>-snapshots/`. Only ONE file is kept across browsers. Change the location with `expect.toMatchAriaSnapshot.pathTemplate`.
  https://playwright.dev/docs/aria-snapshots#snapshots-as-separate-files
- `page.ariaSnapshot()` / `locator.ariaSnapshot()` return the YAML.
  https://playwright.dev/docs/aria-snapshots#using-pageariasnapshot-and-locatorariasnapshot
- Use targeted assertions for logic and computed values, snapshots for broad structure. Accepting updates without reading them hides bugs.
  https://playwright.dev/docs/aria-snapshots#assertion-testing-vs-snapshot-testing

## Visual comparisons

- `await expect(page).toHaveScreenshot();` The first run writes the baseline and FAILS. Playwright shoots until 2 consecutive screenshots match. Commit the baseline.
  https://playwright.dev/docs/test-snapshots#generating-screenshots
- Rendering varies with OS, browser version, settings, hardware, power source and headless mode. Generate and compare baselines in the same environment, for example a version-pinned container.
  https://playwright.dev/docs/test-snapshots#introduction and https://playwright.dev/docs/ci#via-containers
- Baselines are per browser (or project) and platform: `example-test-1-chromium-darwin.png`. Name explicitly with `toHaveScreenshot('landing.png')`. Path segments must stay inside `<spec>-snapshots` or it throws. Change the layout with `snapshotPathTemplate` (tokens include `{testFileBaseName}`).
  https://playwright.dev/docs/test-snapshots#generating-screenshots and https://playwright.dev/docs/release-notes#version-160
- WebP (lossless): name ending in `.webp`, or for all unnamed ones:
  ```ts
  export default defineConfig({ expect: { toHaveScreenshot: { type: 'webp' } } });
  ```
  https://playwright.dev/docs/test-snapshots#generating-screenshots
- Hover effects are captured. Move the mouse away first: `await page.mouse.move(-1, -1);`
  https://playwright.dev/docs/test-snapshots#hover-effects
- Tolerance: `threshold` (per-pixel color distance, 0 strict to 1 lax, default 0.2) and `maxDiffPixels` (unset by default) or `maxDiffPixelRatio` (0 to 1). Share them in config `expect.toHaveScreenshot`.
  ```js
  await expect(page).toHaveScreenshot({ maxDiffPixels: 100, threshold: 0.1 });
  ```
  https://playwright.dev/docs/test-snapshots#maxdiffpixels-and-threshold
- Hide volatile parts with `stylePath`, a stylesheet applied during capture.
  ```js
  await expect(page).toHaveScreenshot({ stylePath: path.join(__dirname, 'screenshot.css') });
  ```
  https://playwright.dev/docs/test-snapshots#stylepath
- Device descriptors forward `screen`, which changes `window.screen` and media queries. Opt out with `screen: undefined` after the spread. Expect baseline churn when upgrading.
  https://playwright.dev/docs/release-notes#version-164

## Non-image snapshots and plain screenshots

- `expect(value).toMatchSnapshot('hero.txt')` detects text or binary content. Commit `<spec>-snapshots` and review its diffs.
  https://playwright.dev/docs/test-snapshots#non-image-snapshots
- `page.screenshot()` without `path` returns a buffer. `locator.screenshot()` shoots one element. `quality` on `.webp` defaults to 100, which is lossless.
  https://playwright.dev/docs/screenshots#capture-into-buffer and https://playwright.dev/docs/release-notes#version-162

## Accessibility scans with axe

Package: `@axe-core/playwright`.
```ts
import AxeBuilder from '@axe-core/playwright';
const results = await new AxeBuilder({ page }).analyze();
expect(results.violations).toEqual([]);
```
https://playwright.dev/docs/accessibility-testing#scanning-an-entire-page

- `analyze()` scans the current state. Reveal the UI and `waitFor()` it first, otherwise axe misses elements. `include(selector)` narrows the scan.
  ```js
  await page.locator('#navigation-menu-flyout').waitFor();
  const results = await new AxeBuilder({ page }).include('#navigation-menu-flyout').analyze();
  ```
  https://playwright.dev/docs/accessibility-testing#configuring-axe-to-scan-a-specific-part-of-a-page
- By default axe also runs "best practice" rules no WCAG criterion requires. Restrict with `.withTags(['wcag2a', 'wcag2aa', 'wcag21a', 'wcag21aa'])`.
  https://playwright.dev/docs/accessibility-testing#scanning-for-wcag-violations
- Known issues: `disableRules(['rule-id'])` suppresses one rule with many preexisting violations (rule ids come from the `id` of the violations). `exclude(selector)` disables ALL rules on the element and its descendants, so avoid it on large components.
  https://playwright.dev/docs/accessibility-testing#handling-known-issues
- Never snapshot the whole `violations` array (it holds HTML and breaks on unrelated changes). Snapshot a fingerprint of rule id plus targets.
  https://playwright.dev/docs/accessibility-testing#using-snapshots-to-allow-specific-known-issues
- Share configuration through a fixture returning a `makeAxeBuilder` function, then add `include()` per test.
  https://playwright.dev/docs/accessibility-testing#using-a-test-fixture-for-common-axe-configuration
- Automated scans catch only part of WCAG issues. Combine with manual review.
  https://playwright.dev/docs/accessibility-testing#introduction
- `page.accessibility` is removed. Use axe.
  https://playwright.dev/docs/release-notes#version-157
