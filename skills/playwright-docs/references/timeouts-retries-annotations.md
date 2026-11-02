# Timeouts, retries, annotations and parameterization

## Contents

- Timeouts
- Retries
- Serial mode
- Annotations and tags
- Parameterization

## Timeouts

| Timeout | Default | Config | Override |
| --- | --- | --- | --- |
| Test | 30_000 ms | `timeout: 60_000` | `test.setTimeout(120_000)`, `test.slow()` triples |
| Expect | 5_000 ms | `expect: { timeout: 10_000 }` | `toBeVisible({ timeout: 10_000 })` |
| Global | none | `globalTimeout: 3_600_000` | |
| Action | none | `use: { actionTimeout: 10_000 }` | `click({ timeout: 10_000 })` |
| Navigation | none | `use: { navigationTimeout: 30_000 }` | `goto('/', { timeout: 30_000 })` |
| beforeAll / afterAll | 30_000 ms | | `test.setTimeout(60_000)` inside the hook |
| Fixture | shares the test timeout | | `{ timeout: 60_000 }` on the fixture |

https://playwright.dev/docs/test-timeouts

- The test timeout covers the test body, fixture setup and `beforeEach`. A separate budget of the same size covers `afterEach` and fixture teardown.
  https://playwright.dev/docs/test-timeouts#test-timeout
- Extend from a hook: `testInfo.setTimeout(testInfo.timeout + 30_000);`
  https://playwright.dev/docs/test-timeouts#change-timeout-from-a-beforeeach-hook
- Action, navigation and fixture timeouts are low level and "you should not need to change these". For flaky tests, look for the cause elsewhere.
  https://playwright.dev/docs/test-timeouts#advanced-low-level-timeouts
- A slow fixture gets its own timeout instead of a bigger test timeout.
  ```js
  slowFixture: [async ({}, use) => { await use('hello'); }, { timeout: 60_000 }]
  ```
  https://playwright.dev/docs/test-timeouts#fixture-timeout
- Library mode differs: there most operations time out after 30 s. In the test runner they do not, only the test does.
  https://playwright.dev/docs/library#key-differences

## Retries

- No retries by default. `retries: 3` or `--retries=3`. Outcomes: passed, flaky (failed then passed), failed.
  https://playwright.dev/docs/test-retries#retries
- When ANY test fails, its worker process and browser are discarded. The next test or the retry starts in a fresh worker, and `beforeAll` runs again there.
  https://playwright.dev/docs/test-retries#failures
- `testInfo.retry` is non-zero on a retry, for example to clean server state first.
  https://playwright.dev/docs/test-retries#retries
- Group-level: `test.describe.configure({ retries: 2 });`
  https://playwright.dev/docs/test-retries#retries
- `retryStrategy: 'isolated'` runs all retries at the end, one by one in a single worker. Default `'immediate'` retries right away, next to other tests.
  ```js
  export default defineConfig({ retries: 2, retryStrategy: 'isolated' });
  ```
  https://playwright.dev/docs/release-notes#version-162
- Flaky tests exit 0 unless `failOnFlakyTests` / `--fail-on-flaky-tests` is set.
  https://playwright.dev/docs/release-notes#version-145

## Serial mode

- `test.describe.configure({ mode: 'serial' })`: after a failure later tests are skipped, and retries rerun the whole group. Not recommended: keep tests isolated.
  https://playwright.dev/docs/test-retries#serial-mode
- To reuse one page across tests, the documented pattern is serial mode plus a page created in `beforeAll` and closed in `afterAll`.
  ```ts
  let page: Page;
  test.beforeAll(async ({ browser }) => { page = await browser.newPage(); });
  test.afterAll(async () => { await page.close(); });
  ```
  https://playwright.dev/docs/test-retries#reuse-single-page-between-tests
- To serialize only tests that share a resource, prefer a test lock: `test('...', { lock: 'user-settings' }, async ({ page }) => {})`.
  https://playwright.dev/docs/test-parallel#test-locks

## Annotations and tags

- `test.skip` (not run), `test.fail` (runs and must fail, complains if it passes), `test.fixme` (not run, for crashing or slow tests), `test.slow` (triples the timeout). All accept a condition.
  https://playwright.dev/docs/test-annotations#introduction
- Inside a test: `test.skip(browserName === 'firefox', 'Still working on it');`
  https://playwright.dev/docs/test-annotations#conditionally-skip-a-test
- In a `describe`, the callback form also skips `beforeAll`:
  ```js
  test.skip(({ browserName }) => browserName !== 'chromium', 'Chromium only!');
  ```
  https://playwright.dev/docs/test-annotations#conditionally-skip-a-group-of-tests
- Put `test.fixme(isMobile, '...')` in `beforeEach` so the hook body does not run.
  https://playwright.dev/docs/test-annotations#use-fixme-in-beforeeach-hook
- Tags start with `@`. Default form: the details object.
  ```js
  test('test customer login', { tag: ['@fast', '@login'] }, async ({ page }) => {});
  test('test full report', { annotation: [{ type: 'issue', description: 'https://...' }] }, async ({ page }) => {});
  ```
  https://playwright.dev/docs/release-notes#version-142 and https://playwright.dev/docs/test-annotations#annotate-tests
- Filtering: `--grep @fast`, `--grep-invert @fast` (`-G`), OR with `"@fast|@slow"`, AND with lookaheads `"(?=.*@fast)(?=.*@slow)"`. In PowerShell: `--grep --% "@fast^|@slow"`.
  https://playwright.dev/docs/test-annotations#tag-tests
- `TestConfig.tag` adds a tag to every test of the run, useful to tell environments apart in merged reports.
  https://playwright.dev/docs/release-notes#version-157
- The HTML reporter hides annotations whose `type` starts with `_`. Runtime: `test.info().annotations.push({ type, description })`.
  https://playwright.dev/docs/test-annotations#annotate-tests and https://playwright.dev/docs/test-annotations#runtime-annotations

## Parameterization

- Test level: loop with `forEach` or `for`, and give each test a unique title.
  ```js
  [{ name: 'Alice', expected: 'Hello, Alice!' }].forEach(({ name, expected }) => {
    test(`testing with ${name}`, async ({ page }) => { /* ... */ });
  });
  ```
  https://playwright.dev/docs/test-parameterize#parameterized-tests
- Hooks go OUTSIDE the loop so they register once. For per-iteration hooks wrap each iteration in `test.describe(() => {...})`.
  https://playwright.dev/docs/test-parameterize#before-and-after-hooks
- Project level: declare an option fixture, set it per project, type the config.
  ```ts
  export const test = base.extend<TestOptions>({ person: ['John', { option: true }] });
  export default defineConfig<TestOptions>({
    projects: [{ name: 'alice', use: { person: 'Alice' } }],
  });
  ```
  https://playwright.dev/docs/test-parameterize#parameterized-projects
- Only fixtures declared `option: true` can be set from config. Overriding a plain fixture from config is an error.
  https://playwright.dev/docs/release-notes#version-118 and https://playwright.dev/docs/release-notes#version-160
- Secrets come from the environment, not source. Load a `.env` with `dotenv.config({ path: path.resolve(__dirname, '.env') })` at the top of the config.
  https://playwright.dev/docs/test-parameterize#env-files
- Tests can be generated from files read at load time, for example a CSV parsed with `csv-parse/sync`.
  https://playwright.dev/docs/test-parameterize#create-tests-via-a-csv-file
