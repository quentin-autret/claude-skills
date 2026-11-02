# Assertions

Web-first assertions, polling, soft assertions and custom matchers in Playwright Test 1.64.1.

## Contents

- Web-first versus generic matchers
- Timeouts
- When no built-in matcher fits
- Soft assertions and messages
- Custom matchers
- Steps

## Web-first versus generic matchers

- Web-first matchers on `expect(locator)`, `expect(page)` and `expect(response)` re-fetch and retry until the condition holds. They MUST be awaited.
  ```js
  await expect(page.getByTestId('status')).toHaveText('Submitted');
  ```
  https://playwright.dev/docs/test-assertions#introduction
- Generic matchers (`toBe`, `toEqual`, `toContain`, `toBeTruthy`) are synchronous, take no `await` and never retry. On a value read from the page they make tests flaky.
  https://playwright.dev/docs/test-assertions#non-retrying-assertions
- Wrong and right:
  ```js
  // checks once, returns immediately
  expect(await locator.isVisible()).toBe(true);
  // waits and retries
  await expect(locator).toBeVisible();
  ```
  https://playwright.dev/docs/best-practices#use-web-first-assertions
- Do not import Jest's `expect`. It is not integrated with the runner. Use the one from `@playwright/test`.
  https://playwright.dev/docs/test-assertions#compatibility-with-expect-library
- A missing `await` is the most common bug. Lint with `@typescript-eslint/no-floating-promises`.
  https://playwright.dev/docs/best-practices#lint-your-tests
- Some doc pages show `expect(...).toBeChecked()` without `await`. Do not copy that: web-first assertions are always awaited.
  https://playwright.dev/docs/input#checkboxes-and-radio-buttons

Less known retrying matchers: `toBeAttached`, `toBeInViewport`, `toContainClass`, `toHaveAccessibleName`, `toHaveAccessibleDescription`, `toHaveAccessibleErrorMessage`, `toHaveRole`, `toHaveValues` (multi-select), `toHaveJSProperty`, `toMatchAriaSnapshot`, `expect(response).toBeOK()`.
https://playwright.dev/docs/test-assertions#auto-retrying-assertions and https://playwright.dev/docs/release-notes#version-150 (`toHaveAccessibleErrorMessage`)

- `toHaveText(..., { useInnerText: true })` compares `innerText` instead of `textContent`.
  https://playwright.dev/docs/protractor#example
- `toHaveCSS(..., { pseudo })` reads a pseudo-element. `toHaveURL` accepts a predicate and `{ ignoreCase }`.
  https://playwright.dev/docs/release-notes#version-160, https://playwright.dev/docs/release-notes#version-151 (predicate) and https://playwright.dev/docs/release-notes#version-144 (ignoreCase)
- `toBeEditable` throws when the target is not an input, select or other editable element.
  https://playwright.dev/docs/release-notes#version-150

## Timeouts

- Default assertion timeout: 5 seconds, unrelated to the test timeout. Change per call (`{ timeout: 10_000 }`) or in config `expect: { timeout }`.
  https://playwright.dev/docs/test-assertions#introduction and https://playwright.dev/docs/test-timeouts#expect-timeout
- `expect.configure` creates a preconfigured instance.
  ```js
  const slowExpect = expect.configure({ timeout: 10000 });
  const softExpect = expect.configure({ soft: true });
  ```
  https://playwright.dev/docs/test-assertions#expectconfigure

## When no built-in matcher fits

Default: `expect.poll` around a value-returning function. Timeout 5 s, intervals `[100, 250, 500, 1000]`.
```js
await expect.poll(async () => {
  const response = await page.request.get('https://api.example.com');
  return response.status();
}, { timeout: 10000 }).toBe(200);
```
https://playwright.dev/docs/test-assertions#expectpoll

Escape hatch: `expect(...).toPass()` retries a whole block. Its default timeout is 0 (no timeout) and it ignores the custom expect timeout, so always pass `timeout`.
```js
await expect(async () => {
  const response = await page.request.get('https://api.example.com');
  expect(response.status()).toBe(200);
}).toPass({ intervals: [1_000, 2_000, 10_000], timeout: 60_000 });
```
https://playwright.dev/docs/test-assertions#expecttopass

Mixing an API check and a locator assertion in one retried block:
```js
await expect(async () => {
  const response = await page.request.get('https://api.example.com/status');
  expect(response.status()).toBe(200);
  await expect(page.getByRole('status')).toHaveText('Ready');
}).toPass({ timeout: 60_000 });
```
https://playwright.dev/docs/test-assertions#expecttopass

`expect.toPass` timeout and intervals can be set globally or per project in config, under `expect: { toPass: { timeout, intervals } }`.
https://playwright.dev/docs/api/class-testconfig#test-config-expect
https://playwright.dev/docs/release-notes#version-143 and https://playwright.dev/docs/release-notes#version-144
https://playwright.dev/docs/release-notes#version-143 and https://playwright.dev/docs/release-notes#version-144

## Soft assertions and messages

- `expect.soft` keeps the test running but marks it failed. Bail out by checking `test.info().errors`. Works only with the Playwright runner.
  ```js
  await expect.soft(page.getByTestId('status')).toHaveText('Success');
  expect(test.info().errors).toHaveLength(0);
  ```
  https://playwright.dev/docs/test-assertions#soft-assertions
- `expect.soft.poll(...)` is supported.
  https://playwright.dev/docs/release-notes#version-161
- A custom message is the second argument of `expect`, shown in reporters.
  ```js
  await expect(page.getByText('Name'), 'should be logged in').toBeVisible();
  ```
  https://playwright.dev/docs/test-assertions#custom-expect-message

## Custom matchers

- `baseExpect.extend({...})` returns `{ message, pass, name, expected, actual }`. `message` is a callback. Honor `this.isNot`. Read the timeout as `options?.timeout ?? this.timeout`.
  ```ts
  import { expect as baseExpect } from '@playwright/test';
  export const expect = baseExpect.extend({
    async toHaveAmount(locator, expected, options) { /* ... */ },
  });
  ```
  https://playwright.dev/docs/test-assertions#add-custom-matchers-using-expectextend and https://playwright.dev/docs/release-notes#version-145
- Export the extended `expect` from the same fixtures file as `test`. Combine modules with `mergeExpects` and `mergeTests`.
  ```ts
  export const expect = mergeExpects(dbExpect, a11yExpect);
  export const test = mergeTests(dbTest, a11yTest);
  ```
  https://playwright.dev/docs/test-assertions#combine-custom-matchers-from-multiple-modules
- Asymmetric matchers nest inside others: `expect.any`, `anything`, `arrayContaining`, `arrayOf`, `closeTo`, `objectContaining`, `stringContaining`, `stringMatching`.
  https://playwright.dev/docs/test-assertions#asymmetric-matchers

## Steps

- `test.step(title, body, { box: true })` points errors at the step call site. `{ timeout }` gives a per-step timeout. `test.step.skip(...)` skips one.
  https://playwright.dev/docs/release-notes#version-139 and https://playwright.dev/docs/release-notes#version-150
- The step body receives a `TestStepInfo`: `step.skip(cond, reason)` and `step.attach(...)`.
  https://playwright.dev/docs/release-notes#version-151
- `test.abort(message)` fails the current test from a fixture, hook or route handler.
  https://playwright.dev/docs/release-notes#version-160
