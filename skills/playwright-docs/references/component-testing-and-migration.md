# Component testing and migration from other tools

## Contents

- Component testing model
- Gallery and config
- Story and test patterns
- Migrating from the experimental ct packages
- From Testing Library
- From Protractor
- From Puppeteer

## Component testing model

https://playwright.dev/docs/test-components#how-it-works

- A component test is a normal Playwright test against a story gallery page served by your own dev server. No dedicated runtime or bundler plugin. `test`, `expect` and the `mount` fixture come from `@playwright/test`.
  ```js
  import { test, expect } from '@playwright/test';

  test('click should expand', async ({ mount }) => {
    const component = await mount('components/Expandable/Stateful');
    await component.getByRole('button').click();
    await expect(component.getByTestId('expanded')).toHaveValue('true');
  });
  ```
  https://playwright.dev/docs/test-components#introduction
- A story is a named export in a `*.story.*` file that renders one scenario with its props, providers and mock data.
- The gallery is one page exposing `window.mount({ story, props })` and `window.unmount()`, rendering into `#root`. You own it.
- `mount` navigates to `baseURL`, calls `window.mount` and returns a Locator for the root.
- Setup lives in the story, in the browser. Assertions use what the page shows: DOM, URL, network.

## Gallery and config

- Playwright ships the setup as an agent skill: `npx playwright init-skills` installs `playwright-component-testing`.
  https://playwright.dev/docs/test-components#step-1-point-your-coding-agent-at-the-skill
- Gallery contract: under `playwright/gallery/`. An unknown story or a render error rejects, so `mount()` throws. The root is reused so `update(props)` keeps state. App-wide setup goes in the body of `window.mount`.
  https://playwright.dev/docs/test-components#step-1-point-your-coding-agent-at-the-skill
- Dedicated project, `baseURL` pointing AT the gallery:
  ```js
  projects: [{
    name: 'components',
    testDir: './tests/components',
    use: {
      ...devices['Desktop Chrome'],
      baseURL: 'http://localhost:5173/playwright/gallery/index.html',
      serviceWorkers: 'block',
      reuseContext: true,
    },
  }],
  webServer: {
    command: 'npm run dev',
    url: 'http://localhost:5173/playwright/gallery/index.html',
    reuseExistingServer: !process.env.CI,
  },
  ```
  `serviceWorkers: 'block'` stops cached responses from shadowing `page.route()` mocks. `reuseContext: true` reuses the context within a worker for speed.
  https://playwright.dev/docs/test-components#step-2-configure-playwright

## Story and test patterns

- One export per scenario (`Primary`, `Disabled`, `WithLongTitle`), next to the component. Prefer a new export over parameterizing.
  https://playwright.dev/docs/test-components#stories-as-a-methodology
- Story ids come from the path under `src/` minus `.story.*`, plus the export: `components/Button/Primary`. Any unique suffix works: `mount('Button/Primary')`.
  https://playwright.dev/docs/test-components#stories-as-a-methodology
- Never pass callbacks between Node and the browser. The story records outcomes into a hidden input and the test asserts with `toHaveValue()`.
  ```js
  export const Stateful = () => {
    const [expanded, setExpanded] = useState(false);
    return <>
      <Expandable expanded={expanded} setExpanded={setExpanded} title='Title'>Details</Expandable>
      <form hidden><input data-testid='expanded' readOnly value={String(expanded)} /></form>
    </>;
  };
  ```
  Use `String(...)` for scalars and `JSON.stringify(...)` for payloads, one input per value.
  https://playwright.dev/docs/test-components#record-state-for-assertions
- Plain serializable props as second argument, typed by the story:
  ```js
  const component = await mount<typeof WithTitle>('Button/WithTitle', { title: 'Hello' });
  await component.update({ value: 2 });
  ```
  https://playwright.dev/docs/test-components#per-test-props and https://playwright.dev/docs/test-components#prop-transitions-with-update
- Screenshot the returned root, not the page.
  ```js
  await expect(await mount('Button/Primary')).toHaveScreenshot('primary.png');
  ```
  https://playwright.dev/docs/test-components#multiple-states-and-visual-comparison
- `mount` navigates, so register `page.route()` BEFORE it.
  https://playwright.dev/docs/test-components#handling-network-requests
- Never reach into component instances or methods. Interact as a user and record effects in the DOM.
  https://playwright.dev/docs/test-components#how-do-i-access-the-components-methods-or-its-instance
- Story ids are strings: renaming a story breaks specs at runtime only.
  https://playwright.dev/docs/test-components#migration-from-the-experimental-packages

## Migrating from the experimental ct packages

`@playwright/experimental-ct-react`, `-react17` and `-vue` are no longer published. Stay on Playwright 1.62 until migrated, then upgrade.
https://playwright.dev/docs/test-components#introduction

| Old | New |
| --- | --- |
| `mount(<Button onClick={spy} />)` | stateful story + hidden input + `toHaveValue()` |
| per-test JSX, children, slots | one story export per composition |
| `component.update(<Button count={2} />)` | `component.update({ count: 2 })` |
| `beforeMount` / `afterMount` | body of gallery `window.mount`, or story decorators |
| `hooksConfig` | props read by the story: `mount('App/Routing', { route: '/dashboard' })` |
| `router` fixture, MSW in Node | `page.route()` in the test, or MSW `setupWorker` in a story |
| `playwright/index.html` | gallery `index.html` |
| `ctViteConfig`, `ctPort`, `ctTemplateDir` | removed, port lives in `webServer` and `baseURL` |

https://playwright.dev/docs/test-components#migration-from-the-experimental-packages

Migrate spec by spec beside the old CT project, then delete `playwright/index.html`, `playwright/index.ts`, `playwright/.cache` and the dependency.

## From Testing Library

https://playwright.dev/docs/testing-library#cheat-sheet

- `screen` becomes `page` or `component`, queries become locators, `waitFor` becomes web-first assertions.
- No get/find/query split: `await component.getByText('Click me').click()` always auto-waits.
- `getByLabelText` becomes `getByLabel`, `queryByPlaceholderText` becomes `getByPlaceholder`. `rerender` becomes `component.update(props)`.
- `waitForElementToBeRemoved` becomes `await expect(page.getByText('the mummy')).toBeHidden();`
  https://playwright.dev/docs/testing-library#replacing-waitfor
- `within(el)` becomes locator chaining.
  https://playwright.dev/docs/testing-library#replacing-within

## From Protractor

https://playwright.dev/docs/protractor#cheat-sheet

- `by.model('x')` becomes `page.locator('[ng-model="x"]')`, `by.repeater('x')` becomes `page.locator('[ng-repeat="x"]')`. `browser.getCurrentUrl()` becomes `page.url()`.
- `waitForAngular` is unneeded with auto-waiting. For edge cases the page provides an Angular 2+ polyfill based on `window.getAllAngularTestabilities()`.
  https://playwright.dev/docs/protractor#polyfilling-waitforangular

## From Puppeteer

https://playwright.dev/docs/puppeteer#cheat-sheet

- `createIncognitoBrowserContext()` becomes `browser.newContext()`. `setViewport` becomes `setViewportSize`.
- `waitForXPath(x)` becomes `waitForSelector(x)`. `waitForNetworkIdle()` becomes `waitForLoadState('networkidle')`, rarely useful with auto-waiting.
- `page.$eval` + generic expect becomes a web-first assertion: `await expect(page.locator('.hero__title')).toContainText('...')`.
  https://playwright.dev/docs/puppeteer#test-example
- `page.type` becomes `locator.fill`. `waitForFileChooser` + `uploadFile` becomes `setInputFiles`. Cookies move to the context: `context.cookies`, `addCookies`, `clearCookies`.
- Locators are strict: an operation throws if more than one element matches. `page.$` is discouraged, use locators.
  https://playwright.dev/docs/puppeteer#cheat-sheet
