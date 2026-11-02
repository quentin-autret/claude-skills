# API testing and the clock

`APIRequestContext` for server calls, and `page.clock` for time control, in Playwright Test 1.64.1.

## API testing

- Configure `baseURL` and `extraHTTPHeaders` once in `use`. The built-in `request` fixture honors them, and `use.proxy` too.
  ```ts
  use: {
    baseURL: 'https://api.github.com',
    extraHTTPHeaders: { 'Authorization': `token ${process.env.API_TOKEN}` },
  }
  ```
  https://playwright.dev/docs/api-testing#configuration
- Assert on the response, then on the body.
  ```ts
  expect(response.ok()).toBeTruthy();
  expect(await issues.json()).toContainEqual(expect.objectContaining({ /* expected fields */ }));
  ```
  https://playwright.dev/docs/api-testing#writing-tests
- `await expect(response).toBeOK()` is the web-first form.
  https://playwright.dev/docs/test-assertions#auto-retrying-assertions
- Typed JSON: `request.get<User>('/api/users/42')`, then `response.json()` is typed.
  https://playwright.dev/docs/release-notes#version-163
- `params` accepts `URLSearchParams` or a string. `form` and `multipart` accept `FormData`. `maxRetries` on `fetch` retries on `ECONNRESET`. `newContext({ maxRedirects, failOnStatusCode })` tune a context.
  https://playwright.dev/docs/release-notes#version-147 and https://playwright.dev/docs/release-notes#version-148 and https://playwright.dev/docs/release-notes#version-144 and https://playwright.dev/docs/release-notes#version-146 and https://playwright.dev/docs/release-notes#version-152 and https://playwright.dev/docs/release-notes#version-151
- `APIRequestContext` has `addCookies`, `cookies` and `clearCookies`.
  https://playwright.dev/docs/release-notes#version-164

### Which request context: shared or isolated cookies

https://playwright.dev/docs/api-testing#context-request-vs-global-request

- `page.request` and `context.request` SHARE the browser context cookie store: they send its `Cookie` header and write back `Set-Cookie`.
  ```ts
  const response = await context.request.fetch(route.request()); // shares cookies with the context
  ```
- `request.newContext()` (imported from `@playwright/test`, or `playwright.request.newContext()`) has ISOLATED cookies and never touches the browser. Export them by hand with `storageState()`.
- A context you create must be disposed, or it leaks.
  ```ts
  let apiContext;
  test.beforeAll(async ({ playwright }) => {
    apiContext = await playwright.request.newContext({ baseURL: 'https://api.github.com' });
  });
  test.afterAll(async () => { await apiContext.dispose(); });
  ```
  https://playwright.dev/docs/api-testing#sending-api-requests-from-ui-tests
- Storage state is interchangeable: `apiContext.storageState({ path })`, then `browser.newContext({ storageState: 'state.json' })`.
  https://playwright.dev/docs/api-testing#reusing-authentication-state

### Uses beside pure API tests

- Prepare server state before a UI test and check post-conditions after it with the same `request` fixture.
  https://playwright.dev/docs/api-testing#sending-api-requests-from-ui-tests
- Poll an endpoint until it answers.
  ```ts
  await expect.poll(async () => {
    const response = await page.request.get('https://api.example.com');
    return response.status();
  }, { timeout: 10000 }).toBe(200);
  ```
  https://playwright.dev/docs/test-assertions#expectpoll

## Clock

https://playwright.dev/docs/clock

- Overridden by `page.clock`: `Date`, `Temporal.Now`, `setTimeout`, `clearTimeout`, `setInterval`, `clearInterval`, `requestAnimationFrame`, `cancelAnimationFrame`, `requestIdleCallback`, `cancelIdleCallback`, `performance`, `Event.timeStamp`.
- Default: `setFixedTime`. Escape hatch: `install` when you must pause, fast forward or tick. `setSystemTime` is for advanced cases only.
- `setFixedTime` fakes `Date.now()` and `new Date()` only. Timers keep running, but `Date.now` always returns the fixed value.
  ```ts
  await page.clock.setFixedTime(new Date('2024-02-02T10:00:00'));
  await page.goto('http://localhost:3333');
  ```
  https://playwright.dev/docs/clock#test-with-predefined-time
- `install` MUST be called before any other clock call. A native `setInterval` created before `install` then cleared after gives undefined behavior.
- Install before load, then jump:
  ```ts
  await page.clock.install({ time: new Date('2024-02-02T08:00:00') });
  await page.goto('http://localhost:3333');
  await page.clock.pauseAt(new Date('2024-02-02T10:00:00'));
  await page.clock.fastForward('30:00');
  ```
  https://playwright.dev/docs/clock#consistent-time-and-timers
- `fastForward` is like closing and reopening a laptop lid: each due timer fires ONCE. Ticks are milliseconds or a string: `'08'` (seconds), `'01:00'` (mm:ss), `'02:34:10'` (hh:mm:ss). `install()` with no time starts at the current time.
  https://playwright.dev/docs/clock#test-inactivity-monitoring
- `pauseAt` then `runFor(2000)` ticks through time, firing every timer and animation frame on the way.
  https://playwright.dev/docs/clock#tick-through-time-manually-firing-all-the-timers-consistently
