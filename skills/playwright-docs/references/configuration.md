# Configuration

`playwright.config.ts`, `use` options, projects, emulation, TypeScript handling and `webServer` in Playwright Test 1.64.1.

## Contents

- Config file structure
- `use` options and their scopes
- Projects and dependencies
- Emulation and devices
- TypeScript
- webServer

## Config file structure

Runner options go at the TOP LEVEL of `defineConfig`. `use` holds browser and context options only.
https://playwright.dev/docs/test-configuration#introduction

```js
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: 'tests',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  globalTimeout: 60 * 60 * 1000,
  reporter: 'html',
  use: {
    baseURL: 'http://localhost:3000',
    trace: 'on-first-retry',
  },
  projects: [
    { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
  ],
  webServer: {
    command: 'npm run start',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
  },
});
```
https://playwright.dev/docs/test-configuration#basic-configuration

- There is no default `globalTimeout`. Set one so a stuck run fails with a report.
  https://playwright.dev/docs/test-timeouts#global-timeout
- `workers` accepts a percentage such as `'50%'`. `workers: 0` or a negative value is rejected.
  https://playwright.dev/docs/test-configuration#basic-configuration and https://playwright.dev/docs/release-notes#version-160
- Default test file pattern: `**/*.@(spec|test).?(c|m)[jt]s?(x)`. Override with `testMatch` / `testIgnore`, top level or per project.
  https://playwright.dev/docs/test-configuration#filtering-tests
- `outputDir` (default `test-results`) receives screenshots, videos and traces.
  https://playwright.dev/docs/test-configuration#advanced-configuration
- `failOnFlakyTests: true` makes a run with flaky tests exit non-zero. By default a failure recovered on retry exits 0.
  https://playwright.dev/docs/release-notes#version-145 and https://playwright.dev/docs/release-notes#version-152

## `use` options and their scopes

- Narrowest scope wins: config `use`, then project `use`, then `test.use({...})` in a file or `describe`.
  ```js
  test.use({ locale: 'fr-FR' });
  ```
  https://playwright.dev/docs/test-use-options#configuration-scopes
- Options without a direct key go in `use.launchOptions`, `use.contextOptions` or `use.connectOptions`.
  ```js
  use: { launchOptions: { slowMo: 50 } }
  ```
  https://playwright.dev/docs/test-use-options#more-browser-and-context-options
- Contexts you create yourself (`browser.newContext()`, `launchPersistentContext`) INHERIT the `use` options. Explicit arguments win.
  https://playwright.dev/docs/test-use-options#explicit-context-creation-and-option-inheritance
- `test.use({ baseURL: undefined })` resets to the config value, it does not unset. To unset:
  ```js
  test.use({ baseURL: [async ({}, use) => use(undefined), { scope: 'test' }] });
  ```
  https://playwright.dev/docs/test-use-options#reset-an-option
- Defaults: `headless` true, `acceptDownloads` true, `bypassCSP` false, `browserName` chromium, `actionTimeout` 0 (no limit). Recording (`screenshot`, `video`, `trace`) is off.
  https://playwright.dev/docs/test-use-options#other-options and https://playwright.dev/docs/test-use-options#recording-options
- `bypassCSP: true` helps when the CSP includes the production origin.
  https://playwright.dev/docs/test-use-options#other-options
- `httpCredentials`, `proxy`, `clientCertificates: [{ origin, certPath, keyPath, passphrase }]` and `extraHTTPHeaders` live in `use`.
  https://playwright.dev/docs/network#http-authentication and https://playwright.dev/docs/release-notes#version-146

## Projects and dependencies

- All projects run by default. `--project=firefox` runs one, `--project` accepts `*` wildcards.
  https://playwright.dev/docs/test-projects#run-projects
- `default: false` keeps a project out of plain runs. `--project="*"` runs everything (use on CI).
  ```js
  { name: 'firefox', use: devices['Desktop Firefox'], default: false }
  ```
  https://playwright.dev/docs/release-notes#version-164
- Per-project `retries`, `timeout`, `testMatch`, `testIgnore`, `workers`, `use` are allowed.
  ```js
  { name: 'Smoke', testMatch: /.*smoke.spec.ts/, retries: 0 },
  { name: 'Default', testIgnore: /.*smoke.spec.ts/, retries: 2 },
  ```
  https://playwright.dev/docs/test-projects#splitting-tests-into-projects
- `dependencies: ['setup']`: dependents start only after ALL dependency tests pass. A failed dependency means dependents do not run. `teardown: 'cleanup db'` on the setup project runs after all dependents.
  https://playwright.dev/docs/test-projects#dependencies and https://playwright.dev/docs/test-projects#teardown
- Filters (`--grep`, `--shard`, `file:line`, `test.only`) select primary tests only. Their dependency projects still run in full. `--no-deps` skips dependencies and teardowns.
  https://playwright.dev/docs/test-projects#test-filtering
- Default for shared setup: a setup project, not `globalSetup`.
  https://playwright.dev/docs/test-global-setup-teardown#introduction

## Emulation and devices

- Spread the device FIRST, then override, because the device also defines `viewport`, `isMobile` and `userAgent`.
  ```js
  use: { ...devices['Desktop Chrome'], viewport: { width: 1280, height: 720 } }
  ```
  https://playwright.dev/docs/emulation#viewport
- Device descriptors carry a fixed user agent (Desktop Chrome reports Windows). For the host's own UA, set `userAgent: undefined` after the spread.
  https://playwright.dev/docs/emulation#devices
- Descriptors forward `screen` too. Opt out with `screen: undefined`.
  https://playwright.dev/docs/release-notes#version-164
- Branded browser: put `channel: 'chrome'` or `'msedge'` after the spread.
  https://playwright.dev/docs/test-projects#configure-projects-for-multiple-browsers
- `locale` and `timezoneId` affect the browser only, not the Node test process. Set `TZ` for the runner.
  https://playwright.dev/docs/emulation#locale--timezone
- `geolocation` also needs `permissions: ['geolocation']`. Per-origin grants: `context.grantPermissions(['notifications'], { origin })`.
  https://playwright.dev/docs/emulation#geolocation and https://playwright.dev/docs/emulation#permissions
- `offline: true` cuts navigations, fetch, XHR and WebSocket, but NOT established WebRTC connections.
  https://playwright.dev/docs/emulation#offline
- `colorScheme` is `'light'`, `'dark'` or `'no-preference'`. Print media: `page.emulateMedia({ media: 'print' })`. `javaScriptEnabled: false` emulates a no-JS user.
  https://playwright.dev/docs/emulation#color-scheme-and-media
- The headless clipboard is isolated from the host OS.
  https://playwright.dev/docs/release-notes#version-162

## TypeScript

- Playwright transpiles but does NOT type check, and runs despite non-critical TS errors. Run `npx tsc -p tsconfig.json --noEmit` on CI before the tests.
  https://playwright.dev/docs/test-typescript#introduction
- Honored tsconfig options only: `allowJs`, `baseUrl`, `paths`, `jsx`, `jsxFactory`, `jsxFragmentFactory`, `jsxImportSource`, `references`, `extends`.
  https://playwright.dev/docs/test-typescript#tsconfigjson
- Each imported file uses its closest `tsconfig.json`. `--tsconfig=tsconfig.test.json` forces one file for all imported files, not only test files. The `tsconfig` config option does the same for test files, reporters and so on, but is not used to load the config file itself or the files it imports.
  https://playwright.dev/docs/test-typescript#tsconfig-resolution
- JSX follows tsconfig. When `jsx` is unset, `preserve` or `react-native`, the automatic runtime from `react/jsx-runtime` is used.
  https://playwright.dev/docs/test-typescript#tsconfig-jsx-options
- Very recent TS syntax: compile with `tsc` into `tests-out` and run `playwright test -c tests-out`.
  https://playwright.dev/docs/test-typescript#manually-compile-tests-with-typescript
- Do not install both `playwright` and `@playwright/test`. Import `chromium`, `firefox`, `webkit` from `@playwright/test`.
  https://playwright.dev/docs/release-notes#version-134

## webServer

- Defaults: `timeout` 60000 ms, `stdout` `'ignore'`, `stderr` `'pipe'`, `cwd` the config directory, `env` inherits `process.env` plus `PLAYWRIGHT_TEST=1`.
  https://playwright.dev/docs/test-webserver#configuring-a-web-server
- `reuseExistingServer: false` THROWS if something already listens. Use `!process.env.CI`.
  https://playwright.dev/docs/test-webserver#configuring-a-web-server
- `url` counts as ready on 2xx, 3xx, 400, 401, 402 or 403. `port` is deprecated, use `url`.
  https://playwright.dev/docs/test-webserver#configuring-a-web-server
- `wait` waits for log output. Named groups become env variables.
  ```js
  webServer: { command: 'npm run start', wait: { stdout: /Listening on port (?<my_server_port>\d+)/ } }
  ```
  gives `process.env.MY_SERVER_PORT`. With both `url` and `wait`, the first met wins.
  https://playwright.dev/docs/test-webserver#configuring-a-web-server
- Shutdown is a SIGKILL of the process group by default. `gracefulShutdown: { signal: 'SIGTERM', timeout: 500 }` changes it (ignored on Windows, needed for Docker containers).
  https://playwright.dev/docs/test-webserver#configuring-a-web-server
- Set `use.baseURL` too: `page.goto('./login')`, `page.route`, `waitForURL`, `waitForRequest`, `waitForResponse` resolve against it with the `URL()` constructor.
  https://playwright.dev/docs/test-webserver#adding-a-baseurl
- Several servers: `webServer` as an array, each with its own `name`, `url`, `timeout`.
  https://playwright.dev/docs/test-webserver#multiple-web-servers
