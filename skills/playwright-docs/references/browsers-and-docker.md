# Browsers, Docker and special environments

Browser installation, channels, the Docker image, Chrome extensions, WebView2 and Selenium Grid in Playwright 1.64.1.

## Contents

- Browser binaries
- Choosing a Chromium flavor
- Firefox and WebKit
- Installing behind a proxy or mirror
- Docker
- Chrome extensions
- WebView2
- Selenium Grid

## Browser binaries

- Each Playwright version needs its own browser builds. After every upgrade run `npx playwright install --with-deps`.
  https://playwright.dev/docs/browsers#introduction and https://playwright.dev/docs/intro#updating-playwright
- Cache location: `~/.cache/ms-playwright` (Linux), `~/Library/Caches/ms-playwright` (macOS), `%USERPROFILE%\AppData\Local\ms-playwright` (Windows). `PLAYWRIGHT_BROWSERS_PATH` must be set at BOTH install and run time.
  https://playwright.dev/docs/browsers#managing-browser-binaries
- `PLAYWRIGHT_BROWSERS_PATH=0` installs into `node_modules/playwright-core/.local-browsers`. Chrome and Edge are unaffected.
  https://playwright.dev/docs/browsers#hermetic-install
- Stale browsers are garbage collected when no installation needs them. Keep them with `PLAYWRIGHT_SKIP_BROWSER_GC=1` or `install --no-remove`.
  https://playwright.dev/docs/browsers#stale-browser-removal
- `npx playwright install-deps --dry-run` on Linux exits non-zero when packages are missing, a non-interactive check.
  https://playwright.dev/docs/test-cli#install-browsers
- System requirements: Node.js 22.x, 24.x or 26.x. Windows 11+, macOS 14+, Debian 12/13, Ubuntu 22.04/24.04/26.04.
  https://playwright.dev/docs/intro#system-requirements

## Choosing a Chromium flavor

- Default: bundled Chromium. Headless runs use the separate "chromium headless shell".
  https://playwright.dev/docs/browsers#chromium-headless-shell
- Escape hatch for higher-fidelity headless: `channel: 'chromium'` (new headless mode, real Chrome). Install with `--no-shell`.
  ```js
  use: { ...devices['Desktop Chrome'], channel: 'chromium' }
  ```
  https://playwright.dev/docs/browsers#chromium-new-headless-mode
- Branded channels: `chrome`, `msedge`, plus `-beta`, `-dev`, `-canary`. They are not installed by default. `npx playwright install msedge` installs at the OS location and OVERRIDES the existing browser. Enterprise policies can break launch.
  https://playwright.dev/docs/browsers#google-chrome--microsoft-edge
- Use branded stable channels for regression against public browsers and for media codecs Chromium lacks.
  https://playwright.dev/docs/browsers#when-to-use-google-chrome--microsoft-edge-and-when-not-to
- Branded `chrome*` and `msedge*` channels run the new headless mode. Expect snapshot and PDF viewer differences from the headless shell.
  https://playwright.dev/docs/release-notes#version-149

## Firefox and WebKit

Both are Playwright-patched builds. Branded Firefox and Safari are not supported. For behavior closest to Safari (video playback) run WebKit on macOS. WebKit on Linux CI is the cheapest option.
https://playwright.dev/docs/browsers#webkit

## Installing behind a proxy or mirror

https://playwright.dev/docs/browsers#install-behind-a-firewall-or-a-proxy

- `HTTPS_PROXY=https://192.0.2.1 npx playwright install`.
- Custom CA (`self signed certificate in certificate chain`): set `NODE_EXTRA_CA_CERTS` before installing.
- `PLAYWRIGHT_DOWNLOAD_CONNECTION_TIMEOUT` is an idle timeout in ms, default 30 s.
- On Linux, `install-deps` behind a proxy must run as root with the variables: `sudo HTTPS_PROXY=... npx playwright install-deps`.
- Mirror: `PLAYWRIGHT_DOWNLOAD_HOST`, overridden per browser by `PLAYWRIGHT_CHROMIUM_DOWNLOAD_HOST`, `PLAYWRIGHT_FIREFOX_DOWNLOAD_HOST`, `PLAYWRIGHT_WEBKIT_DOWNLOAD_HOST`.
  https://playwright.dev/docs/browsers#download-from-artifact-repository

## Docker

- `mcr.microsoft.com/playwright:v<VERSION>-noble` contains browsers and OS packages but NOT the npm package. Install the Playwright package separately, and keep the tag version equal to the one in your tests when running remotely.
  https://playwright.dev/docs/docker#introduction and https://playwright.dev/docs/docker#image-tags
- Suffixes: `-noble` (Ubuntu 24.04), `-jammy` (22.04), `-resolute` (26.04).
  https://playwright.dev/docs/docker#image-tags
- Run with `--init` (no PID 1 zombies) and `--ipc=host` (Chromium can run out of memory and crash without it). Add `--cap-add=SYS_ADMIN` only for odd local launch errors.
  ```bash
  docker run -it --rm --ipc=host mcr.microsoft.com/playwright:v<VERSION>-noble /bin/bash
  ```
  https://playwright.dev/docs/docker#recommended-docker-configuration
- The default user is root, which disables the Chromium sandbox. Fine for your own E2E tests. For untrusted sites, use `--user pwuser` plus a seccomp profile.
  https://playwright.dev/docs/docker#run-the-image
- Alpine and other musl distributions are not supported.
  https://playwright.dev/docs/docker#alpine
- Remote server: `npx -y playwright@<VERSION> run-server --port 3000 --host 0.0.0.0` in the container, then `PW_TEST_CONNECT_WS_ENDPOINT=ws://127.0.0.1:3000/ npx playwright test`. Versions must match. Reach host services with `--add-host=hostmachine:host-gateway`.
  https://playwright.dev/docs/docker#remote-connection
- Own image: `FROM node:20-bookworm` then `RUN npx -y playwright@<VERSION> install --with-deps`.
  https://playwright.dev/docs/docker#build-your-own-image

## Chrome extensions

https://playwright.dev/docs/chrome-extensions

- Only Chromium with a persistent context. Chrome and Edge removed the side-loading flags, so use the bundled Chromium. `channel: 'chromium'` allows headless.
- Override `context` in a fixture:
  ```ts
  export const test = base.extend<{ context: BrowserContext; extensionId: string }>({ ... });
  export const expect = test.expect;
  ```
  launch with `chromium.launchPersistentContext('', { channel: 'chromium', args: [`--disable-extensions-except=${pathToExtension}`, `--load-extension=${pathToExtension}`] })`, close it after `use`, and read `extensionId` from `serviceWorker.url().split('/')[2]`.
  https://playwright.dev/docs/chrome-extensions#testing
- MV3 service workers suspend after about 30 s idle. The same `Worker` object stays valid with no new event. An `evaluate()` in flight at suspension throws "Service worker restarted".
  https://playwright.dev/docs/chrome-extensions#service-worker-idle-suspension-mv3
- Manifest v2 is no longer supported. `backgroundPage` is never emitted.
  https://playwright.dev/docs/release-notes#version-155 and https://playwright.dev/docs/release-notes#version-156

## WebView2

- Start the control with `--remote-debugging-port=<port>` (env `WEBVIEW2_ADDITIONAL_BROWSER_ARGUMENTS`), then `chromium.connectOverCDP('http://localhost:9222')` and take `browser.contexts()[0].pages()[0]`.
  https://playwright.dev/docs/webview2#overview
- All instances share ONE user data directory by default, so parallel tests interfere. Set `WEBVIEW2_USER_DATA_FOLDER` per worker, and a per-worker port such as `10000 + testInfo.workerIndex`.
  https://playwright.dev/docs/webview2#writing-and-running-tests

## Selenium Grid

Experimental, Chrome and Edge only, over CDP. `SELENIUM_REMOTE_URL=http://<hub>:4444 npx playwright test` with no code change. Extras through `SELENIUM_REMOTE_CAPABILITIES` and `SELENIUM_REMOTE_HEADERS`. Nodes need `SE_NODE_GRID_URL`.
https://playwright.dev/docs/selenium-grid#connecting-playwright-to-selenium-grid
