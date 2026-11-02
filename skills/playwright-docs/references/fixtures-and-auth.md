# Fixtures, isolation, page objects and authentication

## Contents

- Isolation model
- Fixtures over hooks
- Fixture scopes, options and order
- Global hooks and global setup
- Page objects
- Authentication with a setup project
- Multiple roles and per-worker accounts
- Custom selector engines

## Isolation model

- Each test gets a fresh BrowserContext (like a new browser profile) with a `page` in it. Starting fresh beats cleanup, which is forgettable and cannot undo some state.
  https://playwright.dev/docs/browser-contexts#two-ways-of-test-isolation
- Built-in fixtures: `page`, `context` (per test), `browser` (shared), `browserName`, `request` (an isolated APIRequestContext per test).
  https://playwright.dev/docs/test-fixtures#built-in-fixtures
- Several users: a `browser.newContext()` each.
  ```js
  test('admin and user', async ({ browser }) => {
    const adminContext = await browser.newContext();
    const userContext = await browser.newContext();
    const adminPage = await adminContext.newPage();
    const userPage = await userContext.newPage();
  });
  ```
  https://playwright.dev/docs/browser-contexts#multiple-contexts-in-a-single-test
- `test.beforeAll` / `afterAll` run once per WORKER, not per run.
  https://playwright.dev/docs/writing-tests#using-test-hooks

## Fixtures over hooks

A fixture is the default: it keeps setup and teardown together, is lazy (runs when a test asks) and composes. Turn a before/after hook pair that creates and removes one thing into a fixture.
https://playwright.dev/docs/test-fixtures#with-fixtures

```js
export const test = base.extend<MyFixtures>({
  todoPage: async ({ page }, use) => {
    const todoPage = new TodoPage(page);
    await todoPage.goto();
    await use(todoPage);
    await todoPage.removeAll();
  },
});
export { expect } from '@playwright/test';
```
https://playwright.dev/docs/test-fixtures#creating-a-fixture

- Tests import `test` from the fixtures file, not `@playwright/test`.
  https://playwright.dev/docs/test-fixtures#adding-global-beforeeachaftereach-hooks
- Fixture names: letters, digits and underscores, starting with a letter or underscore.
  https://playwright.dev/docs/test-fixtures#creating-a-fixture
- Override a built-in fixture by redeclaring it.
  ```js
  export const test = base.extend({
    page: async ({ baseURL, page }, use) => {
      await page.goto(baseURL);
      await use(page);
    },
  });
  ```
  https://playwright.dev/docs/test-fixtures#overriding-fixtures
- Fixture setup and teardown count toward the test timeout. A slow fixture takes its own `{ timeout }`.
  https://playwright.dev/docs/test-fixtures#fixture-timeout
- Combine with `mergeTests(dbTest, a11yTest)`.
  https://playwright.dev/docs/test-fixtures#combine-custom-fixtures-from-multiple-modules
- `{ box: true }` hides a fixture from UI mode, traces and reports. `box: 'self'` hides only it and keeps its inner steps. `{ title: '...' }` renames it.
  https://playwright.dev/docs/test-fixtures#box-fixtures

## Fixture scopes, options and order

- Worker scope uses the tuple form, typed in the SECOND generic: `base.extend<{}, { account: Account }>`. Each worker fixture gets its own timeout, equal to the default test timeout.
  ```js
  account: [async ({ browser }, use, workerInfo) => { /* ... */ await use({ username, password }); }, { scope: 'worker' }],
  ```
  https://playwright.dev/docs/test-fixtures#worker-scoped-fixtures
- `{ auto: true }` runs a fixture for every test (or worker) unlisted. `testInfo.outputPath('logs.txt')` gives a unique path, and `testInfo.status !== testInfo.expectedStatus` detects failure in teardown.
  https://playwright.dev/docs/test-fixtures#automatic-fixtures
- Option fixtures: `defaultItem: ['Something nice', { option: true }]`, set per project via `use: { defaultItem: 'Buy milk' }` and `defineConfig<MyOptions>`.
  https://playwright.dev/docs/test-fixtures#fixtures-options
- An array-valued option must be wrapped, or it is read as the tuple form:
  ```js
  test.use({ persons: [actualPersons, { scope: 'test' }] });
  ```
  https://playwright.dev/docs/test-fixtures#fixtures-options
- Order: dependencies set up first, torn down last. Unused non-auto fixtures never run. Automatic worker fixtures precede everything, `beforeAll` included. Automatic test fixtures precede `beforeEach`.
  https://playwright.dev/docs/test-fixtures#execution-order

## Global hooks and global setup

- `test.beforeEach` covers only its file or `describe`. For every test, use an auto fixture in the shared fixtures file. For once per worker, `{ scope: 'worker', auto: true }`.
  https://playwright.dev/docs/test-fixtures#adding-global-beforeeachaftereach-hooks and https://playwright.dev/docs/test-fixtures#adding-global-beforeallafterall-hooks
- One-time setup defaults to a setup project with `dependencies`: it shows in reports and traces, uses fixtures and honors `headless` and `testIdAttribute`. `globalSetup` does none of that.
  ```ts
  { name: 'setup db', testMatch: /global\.setup\.ts/, teardown: 'cleanup db' },
  { name: 'cleanup db', testMatch: /global\.teardown\.ts/ },
  { name: 'chromium', use: { ...devices['Desktop Chrome'] }, dependencies: ['setup db'] },
  ```
  https://playwright.dev/docs/test-global-setup-teardown#option-1-project-dependencies
- Escape hatch `globalSetup`: one exported function taking `FullConfig`, which may return the teardown function. Data goes to tests via `process.env`. Read `baseURL` from `config.projects[0].use`.
  https://playwright.dev/docs/test-global-setup-teardown#option-2-configure-globalsetup-and-globalteardown
- To trace a failing `globalSetup`, start `context.tracing`, and in a `catch` stop it with a `path`, then rethrow.
  https://playwright.dev/docs/test-global-setup-teardown#capturing-trace-of-failures-during-global-setup

## Page objects

A class with `page` and `Locator` fields built in the constructor plus action methods, handed over by a fixture.
```ts
export class PlaywrightDevPage {
  readonly page: Page;
  readonly getStartedLink: Locator;
  constructor(page: Page) {
    this.page = page;
    this.getStartedLink = page.locator('a', { hasText: 'Get started' });
  }
  async goto() { await this.page.goto('https://playwright.dev'); }
}
```
https://playwright.dev/docs/pom#implementation

## Authentication with a setup project

- Store state under `playwright/.auth`, gitignored. It can impersonate the account, so never commit it.
  https://playwright.dev/docs/auth#core-concepts
- Log in once in a setup project, save `storageState`, reuse it in dependents.
  ```js
  // tests/auth.setup.ts
  setup('authenticate', async ({ page }) => {
    // login steps
    await page.waitForURL('https://github.com/');
    await expect(page.getByRole('button', { name: 'View profile and more' })).toBeVisible();
    await page.context().storageState({ path: authFile });
  });
  // playwright.config.ts
  projects: [
    { name: 'setup', testMatch: /.*\.setup\.ts/ },
    { name: 'chromium',
      use: { ...devices['Desktop Chrome'], storageState: 'playwright/.auth/user.json' },
      dependencies: ['setup'] },
  ]
  ```
  https://playwright.dev/docs/auth#basic-shared-account-in-all-tests
- Login sets cookies across redirects, so wait for the final URL or a post-login element BEFORE saving.
  https://playwright.dev/docs/auth#basic-shared-account-in-all-tests
- Reused state covers cookies, local storage, IndexedDB and passkeys, not session storage. IndexedDB (such as Firebase tokens) needs `storageState({ path, indexedDB: true })`. Virtual WebAuthn passkeys come through the `credentials` option of `storageState` (added in 1.62).
  https://playwright.dev/docs/auth#session-storage, https://playwright.dev/docs/release-notes#version-151 and https://playwright.dev/docs/release-notes#version-162
- Delete expired state files. Under `outputDir` it is cleaned before each run.
  https://playwright.dev/docs/auth#basic-shared-account-in-all-tests
- UI mode skips the setup project. Enable it in the filters, run it, then disable it.
  https://playwright.dev/docs/auth#authenticating-in-ui-mode
- Opt out per file: `test.use({ storageState: { cookies: [], origins: [] } });`
  https://playwright.dev/docs/auth#avoid-authentication-in-some-tests
- API login: `request.post(...)`, then `request.storageState({ path: authFile })`.
  https://playwright.dev/docs/auth#authenticate-with-api-request
- `context.setStorageState(...)` resets cookies, local storage and IndexedDB without a new context.
  https://playwright.dev/docs/release-notes#version-159

## Multiple roles and per-worker accounts

- Roles: one setup test per role (`admin.json`, `user.json`), then `test.use({ storageState: 'playwright/.auth/admin.json' })` per file or `describe`.
  https://playwright.dev/docs/auth#multiple-signed-in-roles
- For parallel tests that mutate server state, use one account per worker keyed on `parallelIndex`, logging in with `storageState: undefined` to start clean.
  ```js
  export const test = baseTest.extend<{}, { workerStorageState: string }>({
    storageState: ({ workerStorageState }, use) => use(workerStorageState),
    workerStorageState: [async ({ browser }, use) => {
      const id = test.info().parallelIndex;
      const page = await browser.newPage({ storageState: undefined });
      // log in, save under test.info().project.outputDir, use(fileName)
    }, { scope: 'worker' }],
  });
  ```
  https://playwright.dev/docs/auth#moderate-one-account-per-parallel-worker

## Custom selector engines

Register with `selectors.register('tag', engine)` before any page exists, once per worker via a `{ scope: 'worker', auto: true }` fixture. `{ contentScript: true }` isolates the engine from page globals.
https://playwright.dev/docs/extensibility#custom-selector-engines
