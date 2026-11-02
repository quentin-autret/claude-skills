---
name: playwright-docs
description: Pitfalls, surprising defaults and recommended patterns of Playwright Test for JavaScript and TypeScript, distilled from the official docs. Use when writing, reviewing, debugging or configuring Playwright tests (@playwright/test, playwright.config.ts, npx playwright test), including flaky tests, locators (getByRole, filter, strict mode), web-first assertions (expect, toHaveText, toBeVisible, expect.poll, toPass), fixtures, storageState authentication, project dependencies, timeouts, retries, network mocking (page.route, routeFromHAR, routeWebSocket), API testing, clock, aria snapshots, toHaveScreenshot, axe accessibility scans, traces, UI mode, reporters, CI, Docker, sharding, test locks, component testing with stories and mount, codegen, Playwright MCP, playwright-cli and migration from Puppeteer, Protractor, Testing Library or older Playwright versions.
metadata:
  version: "1.64.1"
  source: https://playwright.dev/docs/intro
  fetched: "2026-11-02"
---

# Playwright Test (JS/TS) from the official docs

## Overview

This skill holds what the Playwright docs say that is easy to get wrong: auto-waiting rules, strictness, timeouts that do not stack the way one expects, mocking blind spots, snapshot update modes, CI settings and removed APIs. Generic testing advice is left out. Every claim carries the docs URL it comes from, so it can be checked.

## Version covered

Playwright Test 1.64.1 for Node.js and TypeScript (`@playwright/test`). Python, Java and .NET content is excluded. When the installed version is older, check the version table in `references/version-changes.md` before applying a newer API.

## How to navigate

1. Read the quick start below. It covers the rules that matter in most tests.
2. Open the one reference file whose domain matches the task, from the table at the end.
3. Before reviewing old code or upgrading, read "Old patterns" below and `references/version-changes.md`.

## Quick start

### Locators

- Default: `page.getByRole(role, { name })`. Then `getByLabel`, `getByText` (non-interactive elements only), `getByTestId` as the contract-based fallback. CSS or XPath only when nothing user-facing works.
  https://playwright.dev/docs/locators#quick-guide
- Locators are strict: an action on a locator matching 2 elements throws. Narrow with `filter({ hasText })`, `filter({ has })` or `inner.within(outer)` instead of reaching for `first()` / `nth()`.
  https://playwright.dev/docs/locators#strictness
- The locator inside `filter({ has })` is resolved from the outer match, never from the page root.
  https://playwright.dev/docs/locators#filter-by-childdescendant
- Text matching is substring and case-insensitive by default, and always whitespace-normalized, even with `exact: true`.
  https://playwright.dev/docs/locators#locate-by-text

### Actions and waiting

- Actions auto-wait for actionability. Add no `waitForTimeout`, no manual load waits before a click.
  https://playwright.dev/docs/actionability#introduction and https://playwright.dev/docs/navigations#when-is-the-page-loaded
- `fill()` is the default for text. `pressSequentially()` only for key-by-key handling.
  https://playwright.dev/docs/input#type-characters
- Events (popup, download, filechooser, response): create the wait promise BEFORE the click, await it after.
  ```js
  const downloadPromise = page.waitForEvent('download');
  await page.getByText('Download file').click();
  const download = await downloadPromise;
  ```
  https://playwright.dev/docs/downloads#introduction
- A `dialog` listener that neither accepts nor dismisses hangs the page. With no listener dialogs are auto-dismissed.
  https://playwright.dev/docs/dialogs#alert-confirm-prompt-dialogs

### Assertions

- Always `await expect(locator).toX()`. Never `expect(await locator.isVisible()).toBe(true)`, which checks once.
  https://playwright.dev/docs/best-practices#use-web-first-assertions
- Assertion timeout is 5 s, separate from the 30 s test timeout.
  https://playwright.dev/docs/test-timeouts
- No matching matcher: default to `expect.poll`. `expect(...).toPass()` has NO timeout by default, so pass one.
  https://playwright.dev/docs/test-assertions#expecttopass
- Use Playwright's `expect`, not Jest's.
  https://playwright.dev/docs/test-assertions#compatibility-with-expect-library

### Structure and config

- Runner options at the top level of `defineConfig`, browser and context options inside `use`.
  https://playwright.dev/docs/test-configuration#introduction
- Fixtures over before/after hook pairs. Tests import `test` from the fixtures file.
  https://playwright.dev/docs/test-fixtures#with-fixtures
- `beforeAll` runs once per worker, and a worker restarts after every failure, so `beforeAll` can run several times.
  https://playwright.dev/docs/test-retries#failures
- Login once in a setup project with `dependencies`, save `storageState` after waiting for the final URL, gitignore `playwright/.auth`.
  https://playwright.dev/docs/auth#basic-shared-account-in-all-tests
- Spread a device first, then override `viewport`, because a device also defines its own `viewport`.
  https://playwright.dev/docs/emulation#viewport
- Playwright does not type check. Run `tsc --noEmit` on CI and lint with `@typescript-eslint/no-floating-promises`.
  https://playwright.dev/docs/test-typescript#introduction and https://playwright.dev/docs/best-practices#lint-your-tests

### Network

- Register `page.route` BEFORE `page.goto`. A glob must match the full URL, and `?` matches only a literal question mark.
  https://playwright.dev/docs/mock#mock-api-requests and https://playwright.dev/docs/network#glob-url-patterns
- Routes miss requests served by a service worker. Set `serviceWorkers: 'block'`.
  https://playwright.dev/docs/network#missing-network-events-and-service-workers
- `page.request` shares the browser cookies. `request.newContext()` is isolated and must be disposed.
  https://playwright.dev/docs/api-testing#context-request-vs-global-request

### CI

- `forbidOnly: !!process.env.CI`, `retries: process.env.CI ? 2 : 0`, `workers: process.env.CI ? 1 : undefined`, `trace: 'on-first-retry'`, and always a `globalTimeout`, otherwise a hung run produces no report.
  https://playwright.dev/docs/test-configuration#basic-configuration and https://playwright.dev/docs/ci#global-timeout
- `npx playwright install --with-deps` after `npm ci` and after every upgrade. Do not cache browsers.
  https://playwright.dev/docs/ci#caching-browsers
- Shard with `fullyParallel: true` and merge blob reports.
  https://playwright.dev/docs/test-sharding#balancing-shards
- Serialize tests that share a resource with a lock, not serial mode: `test('x', { lock: 'db' }, ...)`.
  https://playwright.dev/docs/test-parallel#test-locks

### Snapshots

- Without `-u`, a missing snapshot is written AND the test fails. `--update-snapshots=missing` writes and passes. `-u` means `changed`.
  https://playwright.dev/docs/aria-snapshots#updating-snapshots-with-playwrighttest-and-the---update-snapshots-flag
- Screenshot baselines are per browser and OS. Generate them where they are compared.
  https://playwright.dev/docs/test-snapshots#introduction

## Old patterns

| Old | Current | Source |
| --- | --- | --- |
| `locator.type()`, `page.type()` | `locator.fill()`, or `pressSequentially()` | https://playwright.dev/docs/release-notes#version-138 |
| `page.$()`, `ElementHandle`, `page.$eval` + generic expect | Locator + web-first assertion | https://playwright.dev/docs/handles#locator-vs-elementhandle |
| `expect(await locator.isVisible()).toBe(true)` | `await expect(locator).toBeVisible()` | https://playwright.dev/docs/best-practices#dont-use-manual-assertions |
| `:visible` pseudo-class | `locator.visible()` | https://playwright.dev/docs/release-notes#version-163 |
| `text=Log in`, `id=`, `data-testid=` selectors | `getByText`, `getByTestId` | https://playwright.dev/docs/other-locators#legacy-text-locator |
| `:right-of()`, `:near()` layout selectors | user-visible locators | https://playwright.dev/docs/other-locators#css-matching-elements-based-on-layout |
| `_react`, `_vue` selectors | standard locators | https://playwright.dev/docs/release-notes#version-158 |
| `page.accessibility` | `@axe-core/playwright` | https://playwright.dev/docs/release-notes#version-157 |
| `globalSetup` for login or seeding | setup project with `dependencies` | https://playwright.dev/docs/test-global-setup-teardown#introduction |
| `test.describe.configure({ mode: 'serial' })` for a shared resource | test locks | https://playwright.dev/docs/test-parallel#test-locks |
| `webServer.port` | `webServer.url` | https://playwright.dev/docs/test-webserver#configuring-a-web-server |
| `videosPath`, `videoSize` | `recordVideo`, or `use.video` in the runner | https://playwright.dev/docs/release-notes#version-160 |
| `devtools: true` launch option | `args: ['--auto-open-devtools-for-tabs']` | https://playwright.dev/docs/release-notes#version-158 |
| `-gv` flag | `--grep-invert` or `-G` | https://playwright.dev/docs/release-notes#version-154 |
| `npx playwright open` to record | `npx playwright codegen` | https://playwright.dev/docs/release-notes#version-154 |
| `updateSnapshots: 'all'` to refresh changed ones | `'changed'` (`'all'` rewrites everything) | https://playwright.dev/docs/release-notes#version-150 |
| `waitUntil: 'networkidle'` | auto-waiting and web-first assertions | https://playwright.dev/docs/puppeteer#automation-example |
| `@playwright/experimental-ct-*` with JSX `mount(<C />)` | stories + `mount('Story/Id', props)` from `@playwright/test` | https://playwright.dev/docs/test-components#migration-from-the-experimental-packages |
| browsers downloaded by `npm install` | `npx playwright install --with-deps` | https://playwright.dev/docs/release-notes#version-138 |

## Reference files

| File | Read it for |
| --- | --- |
| [references/locators.md](references/locators.md) | locator priority, text matching, strictness, `filter`, `within`, `and` / `or`, `visible()`, shadow DOM, frames, legacy selectors |
| [references/actions-and-events.md](references/actions-and-events.md) | actionability checks per action, `force`, input, drag and drop, dialogs, downloads, uploads, popups, navigation, hydration, `evaluate`, init scripts, handles |
| [references/assertions.md](references/assertions.md) | web-first versus generic matchers, assertion timeouts, `expect.poll`, `toPass`, soft assertions, custom matchers, `test.step` |
| [references/snapshots-and-accessibility.md](references/snapshots-and-accessibility.md) | snapshot update modes, aria snapshots, `toHaveScreenshot` tolerances and naming, non-image snapshots, axe scans |
| [references/configuration.md](references/configuration.md) | config structure, `use` scopes and defaults, projects and dependencies, devices and emulation, TypeScript handling, `webServer` |
| [references/timeouts-retries-annotations.md](references/timeouts-retries-annotations.md) | every timeout and its default, retries and worker restarts, serial mode, skip / fail / fixme, tags, parameterized tests and projects |
| [references/fixtures-and-auth.md](references/fixtures-and-auth.md) | isolation, custom fixtures, scopes, options, execution order, global hooks, setup projects, page objects, `storageState` auth, roles, per-worker accounts |
| [references/network-and-mocking.md](references/network-and-mocking.md) | `route`, glob rules, headers and redirects a route cannot touch, HAR record and replay, WebSocket mocks, browser API mocks, service workers |
| [references/api-testing-and-clock.md](references/api-testing-and-clock.md) | `request` fixture, shared versus isolated cookies, disposing contexts, `page.clock` methods and ordering |
| [references/ci-parallel-sharding.md](references/ci-parallel-sharding.md) | CI baseline, GitHub Actions and other providers, workers, `fullyParallel`, test locks, sharding, blob report merging |
| [references/browsers-and-docker.md](references/browsers-and-docker.md) | browser install and cache, Chromium channels, proxies and mirrors, Docker image, Chrome extensions, WebView2, Selenium Grid |
| [references/debugging-and-reporting.md](references/debugging-and-reporting.md) | CLI flags, `--debug`, Inspector, trace and video modes, trace viewer, UI mode, reporters |
| [references/component-testing-and-migration.md](references/component-testing-and-migration.md) | stories and gallery component testing, migrating from experimental ct packages, Testing Library, Protractor, Puppeteer |
| [references/codegen-and-agent-tooling.md](references/codegen-and-agent-tooling.md) | codegen, Library versus Test, release channels, Test Agents, playwright-cli, Playwright MCP |
| [references/version-changes.md](references/version-changes.md) | breaking behavior changes, removed and deprecated APIs with replacements, platform support changes |

Sources and fetch details are listed in [SOURCES.md](SOURCES.md).
