# Breaking changes and removed APIs

Behavior changes that break or silently alter existing tests, from Playwright 1.18 up to 1.64. Every item comes from https://playwright.dev/docs/release-notes, anchored as `#version-1NN`.

## Contents

- Changes that alter test results
- Removed or deprecated APIs
- Platform and packaging changes

## Changes that alter test results

| Version | Change | What to do | Source |
| --- | --- | --- | --- |
| 1.64 | Device descriptors forward `screen`, so `window.screen` and media queries change | `screen: undefined` after the spread to opt out | https://playwright.dev/docs/release-notes#version-164 |
| 1.64 | Elements inside a hidden iframe count as hidden | update visibility assertions | https://playwright.dev/docs/release-notes#version-164 |
| 1.64 | `--update-snapshots=missing` passes the run. The unnamed mode is now `default` and still fails | use `missing` on CI to create and verify in one run | https://playwright.dev/docs/release-notes#version-164 |
| 1.64 | JSX in tests follows `tsconfig.json` | set `jsx` options in tsconfig | https://playwright.dev/docs/release-notes#version-164 |
| 1.64 | Videos encoded in VP9, not VP8 | check video consumers | https://playwright.dev/docs/release-notes#version-164 |
| 1.62 | Headless clipboard isolated from the OS | do not rely on the host clipboard | https://playwright.dev/docs/release-notes#version-162 |
| 1.60 | Config overriding a non-option fixture errors. `workers: 0` is rejected | declare `{ option: true }` | https://playwright.dev/docs/release-notes#version-160 |
| 1.59 | JUnit emits `<error>` for some former `<failure>` | update CI parsers | https://playwright.dev/docs/release-notes#version-159 |
| 1.52 | Route globs drop `?` and `[]` | use a RegExp | https://playwright.dev/docs/release-notes#version-152 |
| 1.52 | `route.continue` ignores a `Cookie` header override | `context.addCookies` | https://playwright.dev/docs/release-notes#version-152 |
| 1.50 | `updateSnapshots: 'all'` rewrites unchanged snapshots too | `'changed'` for the old behavior | https://playwright.dev/docs/release-notes#version-150 |
| 1.50 | `toBeEditable` / `isEditable` throw on non-editable element types | target the input itself | https://playwright.dev/docs/release-notes#version-150 |
| 1.49 | `chrome` and `msedge` channels use the new headless mode | rerun and update snapshots. `channel: 'chromium'` opts in for Chromium | https://playwright.dev/docs/release-notes#version-149 |
| 1.45 | Recovered flaky failures exit 0 | `failOnFlakyTests` or `--fail-on-flaky-tests` | https://playwright.dev/docs/release-notes#version-145 |
| 1.44 | After an `addLocatorHandler` handler, Playwright waits for the overlay to hide | `noWaitAfter: true` to opt out | https://playwright.dev/docs/release-notes#version-144 |
| 1.40 | `download.path()` throws for failed or cancelled downloads | handle the error | https://playwright.dev/docs/release-notes#version-140 |
| 1.38 | `npm install` no longer downloads browsers | `npx playwright install --with-deps` after `npm ci` | https://playwright.dev/docs/release-notes#version-138 |
| 1.34 | `playwright test` breaks if both `playwright` and `@playwright/test` are installed | import browsers from `@playwright/test` | https://playwright.dev/docs/release-notes#version-134 |
| 1.23 | webServer ready on 2xx, 3xx and 400-403 | nothing, know it | https://playwright.dev/docs/release-notes#version-123 |
| 1.18 | Custom config options must be `option: true` fixtures. `acceptDownloads` defaults to true | declare options as fixtures | https://playwright.dev/docs/release-notes#version-118 |

## Removed or deprecated APIs

| Old | Replacement | Status | Source |
| --- | --- | --- | --- |
| `locator.type()`, `page.type()`, `frame.type()` | `locator.fill()`, or `pressSequentially()` for key-by-key handling | deprecated 1.38 | https://playwright.dev/docs/release-notes#version-138 |
| `:visible` pseudo-class | `locator.visible()` | replaced 1.63 | https://playwright.dev/docs/release-notes#version-163 |
| `page.accessibility` | `@axe-core/playwright` | removed 1.57 | https://playwright.dev/docs/release-notes#version-157 |
| `_react`, `_vue` selectors, `:light` suffix | standard locators | removed 1.58 | https://playwright.dev/docs/release-notes#version-158 |
| `devtools` launch option | `args: ['--auto-open-devtools-for-tabs']` | removed 1.58 | https://playwright.dev/docs/release-notes#version-158 |
| `videosPath`, `videoSize` context options | `recordVideo` | removed 1.60 | https://playwright.dev/docs/release-notes#version-160 |
| `Locator.ariaRef()` | `Locator.ariaSnapshot` | removed 1.60 | https://playwright.dev/docs/release-notes#version-160 |
| `logger` option of `connect` / `connectOverCDP` | tracing | removed 1.60 | https://playwright.dev/docs/release-notes#version-160 |
| `handle` option of `exposeBinding` | none | removed 1.60 | https://playwright.dev/docs/release-notes#version-160 |
| `ConsoleMessage.location().lineNumber` / `columnNumber` | `line` / `column` | deprecated 1.60 | https://playwright.dev/docs/release-notes#version-160 |
| `Screencast.showActions({ fontSize })` | `style` option | deprecated 1.64 | https://playwright.dev/docs/release-notes#version-164 |
| `BrowserContext.backgroundPage` event | none, never emitted | deprecated 1.56 | https://playwright.dev/docs/release-notes#version-156 |
| `selectOption({ noWaitAfter })` | none | deprecated 1.47 | https://playwright.dev/docs/release-notes#version-147 |
| `-gv` CLI flag | `--grep-invert` or `-G` | removed 1.54 | https://playwright.dev/docs/release-notes#version-154 |
| `npx playwright open` as recorder | `npx playwright codegen` | 1.54 | https://playwright.dev/docs/release-notes#version-154 |
| `webServer.port` | `webServer.url` | deprecated | https://playwright.dev/docs/test-webserver#configuring-a-web-server |
| `@playwright/experimental-ct-react`, `-react17`, `-vue` | `mount` fixture with stories | unpublished | https://playwright.dev/docs/test-components#introduction |
| `@playwright/experimental-ct-svelte` | none | removed 1.59 | https://playwright.dev/docs/release-notes#version-159 |
| `@playwright/experimental-ct-vue2`, `-solid` | none | no updates since 1.49 | https://playwright.dev/docs/release-notes#version-149 |

## Platform and packaging changes

- Ubuntu 20.04 unsupported (1.63). Debian 11 dropped (1.62). Linux arm64 moved to Chrome for Testing (1.63).
  https://playwright.dev/docs/release-notes#version-163 and https://playwright.dev/docs/release-notes#version-162
- Chromium runs on Chrome for Testing builds since 1.57.  https://playwright.dev/docs/release-notes#version-157
- WebKit dropped macOS 13 (1.58) and macOS 14 (1.59).
  https://playwright.dev/docs/release-notes#version-158 and https://playwright.dev/docs/release-notes#version-159
- Chromium extensions manifest v2 dropped (1.55).
  https://playwright.dev/docs/release-notes#version-155
- Node.js 16 removed and 18 deprecated (1.54).
  https://playwright.dev/docs/release-notes#version-154
- Docker image without suffix is Ubuntu 24.04 Noble since 1.47. Use `-jammy` for 22.04.
  https://playwright.dev/docs/release-notes#version-147
- Docker image no longer contains Python (1.21).
  https://playwright.dev/docs/release-notes#version-121
