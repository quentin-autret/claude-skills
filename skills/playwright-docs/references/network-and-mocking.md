# Network interception and mocking

Routes, URL globs, HAR replay, WebSocket mocks, browser API mocks and service workers in Playwright Test 1.64.1.

## Contents

- Routes
- Glob URL patterns
- What a route cannot see or change
- Waiting for traffic
- HAR record and replay
- WebSockets
- Mocking browser APIs
- Service workers

## Routes

- No config needed. `context.route` also covers popups and opened links. `page.route` covers one page.
  https://playwright.dev/docs/network#network-mocking
- Register the route BEFORE `page.goto`. Answer JSON with `route.fulfill({ json })`.
  ```ts
  await page.route('*/**/api/v1/fruits', async route => {
    const json = [{ name: 'Strawberry', id: 21 }];
    await route.fulfill({ json });
  });
  ```
  The trace of such a test shows the API was never called.
  https://playwright.dev/docs/mock#mock-api-requests
- Patch a real response: `route.fetch()`, then pass `response` to `fulfill` so only overridden fields change.
  ```ts
  const response = await route.fetch();
  const json = await response.json();
  json.push({ name: 'Loquat', id: 100 });
  await route.fulfill({ response, json });
  ```
  https://playwright.dev/docs/mock#modify-api-responses
- To force one header while keeping the others, spread `response.headers()` into the `headers` override.
  ```ts
  await route.fulfill({ response, body, headers: { ...response.headers(), 'content-type': 'text/html' } });
  ```
  https://playwright.dev/docs/network#modify-responses
- Remove a request header: copy `route.request().headers()`, delete the key, `route.continue({ headers })`.
  https://playwright.dev/docs/network#modify-requests
- Block by type: `route.request().resourceType() === 'image' ? route.abort() : route.continue()`.
  https://playwright.dev/docs/network#abort-requests
- `page.unrouteAll()` / `context.unrouteAll()` remove routes, optionally waiting for running handlers.
  https://playwright.dev/docs/release-notes#version-141
- `fulfill` and similar route calls are not shown as steps in reports or traces. Check the network tab.
  https://playwright.dev/docs/release-notes#version-148

## Glob URL patterns

https://playwright.dev/docs/network#glob-url-patterns

- `*` matches anything except `/`. `**` matches across `/`. `{a,b}` lists options. `\` escapes.
- `?` matches only a literal question mark. `[]` classes are not supported. Use a RegExp for those.
  https://playwright.dev/docs/release-notes#version-152
- A glob is matched against the full URL, so the query string is part of what it must match. With `?` literal, a pattern like `**/api/items?page=1` works, but for any other query shape use a RegExp or a predicate, as the docs advise for complex matching.
  https://playwright.dev/docs/network#glob-url-patterns
- The glob must match the WHOLE URL, protocol included. `https://example.com/*.js` misses `https://example.com/path/file.js`, `**/*.js` matches both.
- With `baseURL` set, relative patterns resolve against it through `URL()`.
  https://playwright.dev/docs/test-webserver#adding-a-baseurl

## What a route cannot see or change

https://playwright.dev/docs/network#how-request-interception-works

- The handler runs before the browser network stack. Headers it adds later (`Cookie`, `Host`, `Accept-Encoding`, `Content-Length`, `Sec-Fetch-*`) are not reliably in `request.headers()` and cannot be overridden.
  https://playwright.dev/docs/network#headers-owned-by-the-network-stack
- A `cookie` header passed to `route.continue()` is ignored in favor of the cookie store. Set cookies with `context.addCookies`.
  https://playwright.dev/docs/release-notes#version-152
- To read exact wire headers, observe without routing: with no route installed `request.allHeaders()` is complete. `response.allHeaders()` includes `Set-Cookie` for HttpOnly cookies.
  https://playwright.dev/docs/network#headers-owned-by-the-network-stack
- A request and its redirects are one unit: the handler runs once, for the original. `response.request()` is the last hop, `request.redirectedFrom()` walks back.
  https://playwright.dev/docs/network#redirects
- Fulfilling with a 3xx gives no second interception. Chromium and Firefox follow it silently, WebKit rejects the call. Serve the final content directly, or pass `url` to `route.continue()` / `route.fallback()`.
  https://playwright.dev/docs/network#redirects
- If routed requests seem missing, a service worker is serving them. Set `serviceWorkers: 'block'`. Mock Service Worker (MSW) adds its own service worker that takes over requests, which makes them invisible to `context.route` and `page.route`. Pick one mocking system: for network testing plus mocking, use the built-in routes.
  https://playwright.dev/docs/network#missing-network-events-and-service-workers

## Waiting for traffic

Promise first, action second.
```ts
const responsePromise = page.waitForResponse('**/api/fetch_data');
await page.getByText('Update').click();
const response = await responsePromise;
```
Accepts a glob, a RegExp or a predicate.
https://playwright.dev/docs/network#network-events

## HAR record and replay

https://playwright.dev/docs/mock#mocking-with-har-files

- Default workflow: `page.routeFromHAR(path, { url, update: true })` records real traffic, commit the HAR, then remove `update` (or set it to `false`) to replay. `url` is a glob limiting which requests the HAR serves. `context.routeFromHAR` and the `recordHar` context option are alternatives.
  https://playwright.dev/docs/mock#recording-a-har-file
- Without `url`, EVERY request is served from the HAR. In replay, a request with no match is aborted.
  https://playwright.dev/docs/mock#replaying-from-har
- Matching is strict on URL and method, plus payload for POST. Among several matches the one with most matching headers wins.
  https://playwright.dev/docs/mock#replaying-from-har
- CLI alternative: `npx playwright open --save-har=example.har --save-har-glob="**/api/**" https://example.com`. A `.zip` name stores payloads as separate entries.
  https://playwright.dev/docs/mock#recording-har-with-cli
- Edit payloads by hand in the hashed `.txt` files. Rerunning with `update: true` overwrites edits.
  https://playwright.dev/docs/mock#modifying-a-har-file
- HAR and traces include WebSocket traffic. `context.tracing.startHar('trace.har')` works with `await using`.
  https://playwright.dev/docs/release-notes#version-161 and https://playwright.dev/docs/release-notes#version-160

## WebSockets

```ts
await page.routeWebSocket('wss://example.com/ws', ws => {
  ws.onMessage(message => {
    if (message === 'request')
      ws.send('response');
  });
});
```
To sit between page and server, call `ws.connectToServer()` and forward with `server.send(...)`.
https://playwright.dev/docs/mock#mock-websockets

Inspection only: `page.on('websocket', ws => ...)` with `framesent`, `framereceived`, `close`.
https://playwright.dev/docs/network#websockets

## Mocking browser APIs

- Install mocks with `page.addInitScript` so they exist before page scripts run.
  ```ts
  await page.addInitScript(() => {
    window.navigator.getBattery = async () => mockBattery;
  });
  ```
  https://playwright.dev/docs/mock-browser-apis#creating-mocks
- Assigning a read-only property (`navigator.cookieEnabled = true`) has no effect. If the property is configurable, use `Object.defineProperty(Object.getPrototypeOf(navigator), 'cookieEnabled', { value: false })`.
  https://playwright.dev/docs/mock-browser-apis#mocking-read-only-apis
- Record calls with `page.exposeFunction('logCall', msg => log.push(msg))`, then `expect(log).toEqual([...])`.
  https://playwright.dev/docs/mock-browser-apis#verifying-api-calls
- Drive a mock mid-test by keeping it on `window` and calling `page.evaluate(() => window.mockBattery._setLevel(0.275))`.
  https://playwright.dev/docs/mock-browser-apis#updating-mock

## Service workers

https://playwright.dev/docs/service-workers

- Chromium-based browsers only. For ordinary mocking, block them: `use: { serviceWorkers: 'block' }`. It is a context option: `'block'` stops all service worker registration, default `'allow'`.
  https://playwright.dev/docs/api/class-browser#browser-new-context-option-service-workers
  https://playwright.dev/docs/service-workers#how-to-disable-service-workers
- The `serviceworker` event fires BEFORE the worker controls the page. Wait for activation before `worker.evaluate`.
  https://playwright.dev/docs/service-workers#accessing-service-workers-and-waiting-for-activation
- Worker-made requests are reported on the context and are routable by `context.route`. When a fetch handler exists, the page-owned request is NOT routable, only the worker's one.
- `request.frame()` / `response.frame()` THROW when `request.serviceWorker()` is set. Check it first.
  ```ts
  await context.route('**', async route => {
    if (route.request().serviceWorker())
      await route.fulfill({ contentType: 'text/plain', status: 200, body: 'from sw' });
    else
      await route.continue();
  });
  ```
  https://playwright.dev/docs/service-workers#network-events-and-routing
- Requests for an updated service worker main script cannot be routed.
  https://playwright.dev/docs/service-workers#known-limitations
