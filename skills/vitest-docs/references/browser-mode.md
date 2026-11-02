# Browser mode

## Contents

- When and setup
- Several browsers
- Playwright provider
- Locators and assertions
- Interactions
- Component tests
- Mocking in the browser
- Visual and ARIA snapshots
- Commands and trace view

## When and setup

- Browser mode runs tests in a real browser, where jsdom and happy-dom only simulate one. It complements end-to-end runners, it does not replace them. https://vitest.dev/guide/browser/why
- Scaffold with `npx vitest init browser`. A provider is required, called as a factory. `@vitest/browser-playwright` is recommended (it supports parallel runs). The `preview` provider simulates events, has no headless mode and is not for CI. https://vitest.dev/guide/browser

```ts
import { playwright } from '@vitest/browser-playwright'
export default defineConfig({
  test: { browser: { provider: playwright(), enabled: true,
    instances: [{ browser: 'chromium' }] } },
})
```
https://vitest.dev/guide/browser

- Add your framework's Vite plugin. Mix Node and browser tests with `test.projects` and separate `include` globs (`*.browser.test.ts`). https://vitest.dev/guide/browser
- `--browser` without a `browser` block in the config fails. https://vitest.dev/guide/browser
- `browser.headless` defaults to `true` in CI. https://vitest.dev/guide/browser
- Browser mode serves on port 23315 by default, set with the top-level `api` option. `browser.api` is deprecated and has no effect. https://vitest.dev/guide/migration

## Several browsers

- Prefer `browser.instances` to separate projects: instances share one Vite server and its cache. https://vitest.dev/guide/browser/multiple-setups
- Two instances of the same browser need distinct `name`s. Filter with `--project=<name>`. https://vitest.dev/guide/browser/multiple-setups
- Instances inherit the root config: root and instance `setupFiles` both apply. Provider options on an instance replace the parent ones. https://vitest.dev/config/browser/instances , https://vitest.dev/config/browser/provider

## Playwright provider

- One page and one browser context per test file, not per test: isolation stops at the file. https://vitest.dev/config/browser/playwright
- `launchOptions.headless` is ignored (use `browser.headless`). Set the viewport with `browser.viewport`, not `contextOptions`. https://vitest.dev/config/browser/playwright
- `actionTimeout` has no default. Pass `{ timeout }` per call when needed. https://vitest.dev/config/browser/playwright
- The `page` from `vitest/browser` is not Playwright's page. Reach the real one through commands. https://vitest.dev/api/browser/context

## Locators and assertions

- Assert DOM state with `await expect.element(locator)`: it retries until pass or timeout. Plain `expect` fails at once. https://vitest.dev/api/browser/assertions
- Locator methods are async and must be awaited (tests fail otherwise). For types, add `/// <reference types="vitest/browser" />`. https://vitest.dev/api/browser/locators , https://vitest.dev/api/browser/assertions
- Prefer `getByLabelText` over `getByPlaceholder`. There is no `page.locator()`, use the `getBy*` methods. https://vitest.dev/api/browser/locators , https://vitest.dev/guide/browser/component-testing
- Chain `.getBy*`, `.filter` instead of `within(...)`. Keep locators and interact later. https://vitest.dev/api/browser/locators

```ts
const deleteButton = page.getByRole('row').filter({ hasText: 'Vitest' })
  .getByRole('button', { name: /delete/i })
await deleteButton.click()
await expect.element(deleteButton).toBeEnabled()
```
https://vitest.dev/api/browser/locators

- Text locators are exact and case-sensitive by default (`browser.locators.exact`). `getByRole` `name` is documented as a case-insensitive substring unless `exact: true` (the same page also says `exact` follows `browser.locators.exact`, so pass `exact` explicitly when it matters). `getByRole` skips hidden elements unless `includeHidden: true`. https://vitest.dev/api/browser/locators
- Strict: an action on a locator matching several elements fails, and `.or()` can match both. Use `.first()` when any match will do. Prefer chained locators to `.nth()`. https://vitest.dev/api/browser/locators , https://vitest.dev/api/browser/assertions
- `toHaveTextContent` is exact. Use `toMatchTextContent` for partial or RegExp matches. https://vitest.dev/guide/migration
- `toBeVisible`: `opacity: 0` counts as visible, zero size and `display: none` do not. https://vitest.dev/api/browser/assertions

## Interactions

- Use `userEvent` from `vitest/browser` (CDP or WebDriver input), not `@testing-library/user-event` directly. https://vitest.dev/guide/browser
- Prefer `fill` to `type`. `fill` ignores keyboard syntax. https://vitest.dev/api/browser/interactivity
- The default `userEvent` instance is created once and shared (use `userEvent.setup()` for a fresh one). Vitest resets unreleased keyboard state before each test, but not pointer position or hover state: `await userEvent.unhover(document.body)` in `beforeEach`. https://vitest.dev/api/browser/interactivity
- `alert`, `confirm` and `print` block the page. Vitest installs default mocks so tests do not hang, but mock them yourself. https://vitest.dev/guide/browser

## Component tests

- Render with `vitest-browser-react`, `-vue` or `-svelte`. `render` is async: `const screen = await render(<C />)`. Query through locators, not `container`. https://vitest.dev/api/browser/react , https://vitest.dev/guide/migration
- Test behavior seen by the user (props in, events and renders out, loading and error states), not state variables or CSS classes. https://vitest.dev/guide/browser/component-testing
- Moving from Jest plus Testing Library: change the `render` import, use `await expect.element()`, use `vitest/browser` interactions. https://vitest.dev/guide/browser/component-testing

## Mocking in the browser

- Module namespaces are sealed: `vi.spyOn(module, 'fn')` throws. Use `vi.mock('./module.js', { spy: true })` then `vi.mocked(fn).mockImplementation(...)`. https://vitest.dev/guide/browser
- `vi.mock(path)` without factory automocks: exports return `undefined`. https://vitest.dev/guide/migration
- Exported variables cannot be mocked. Export a setter. https://vitest.dev/guide/browser
- Mock requests with MSW `setupWorker` from `msw/browser`: `worker.start()`, `resetHandlers()`, `stop()`. https://vitest.dev/guide/browser/component-testing

## Visual and ARIA snapshots

- `await expect(locator).toMatchScreenshot()`. The first run writes a baseline and fails on purpose. https://vitest.dev/guide/browser/visual-regression-testing
- Keep visual tests in their own project (`*.vrt.test.ts`, headless, fixed viewport). Run them in a standardized environment (Docker or CI). Obsolete screenshots are not removed. https://vitest.dev/guide/browser/visual-regression-testing
- No default pixel tolerance. Prefer `allowedMismatchedPixelRatio` and set `comparatorName: 'pixelmatch'` for typed options. https://vitest.dev/api/browser/assertions
- Capture elements, not pages. Mask dynamic content and `await document.fonts.ready`. https://vitest.dev/guide/browser/visual-regression-testing
- `await expect.element(loc).toMatchAriaSnapshot()` asserts the accessibility tree. Children match as a subsequence unless `- /children: equal`. Hand-written regexes survive `--update` and need doubled backslashes in template literals. https://vitest.dev/guide/browser/aria-snapshots

## Commands and trace view

- Custom `browser.commands` run in Node with full access and none of the `server.fs` protections. Validate browser input. https://vitest.dev/api/browser/commands
- A locator passed to a command arrives as `SerializedLocator`: destructure `{ selector }` (type `SerializedLocator`, which also has a readable `locator` string). https://vitest.dev/guide/migration
- `browser.traceView` defaults to `'failure-only'`. Add steps with `page.mark('name')`. Snapshots miss iframe content, and canvas pixels unless `recordCanvas` is on. https://vitest.dev/guide/browser/trace-view
