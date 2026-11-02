# Testing: IPC mocks, WebDriver end-to-end, CI

## Contents

- Strategy
- Mocking the IPC in unit tests
- End-to-end with WebdriverIO
- Manual tauri-driver
- CI on GitHub Actions

## Strategy

- Unit and integration tests run on a mock runtime that never starts the native webview; end-to-end tests use WebDriver. https://v2.tauri.app/develop/tests/
- Default for E2E: WebdriverIO with `@wdio/tauri-service`, which covers Windows, Linux and macOS. Raw `tauri-driver` covers only Windows and Linux (no macOS WebDriver client). https://v2.tauri.app/develop/tests/ https://v2.tauri.app/develop/tests/webdriver/
- Rust commands become testable on the mock runtime when they are generic over `R: Runtime`. https://v2.tauri.app/develop/calling-rust/

## Mocking the IPC in unit tests

- `mockIPC` from `@tauri-apps/api/mocks` intercepts every `invoke`. Call `clearMocks()` after each test. https://v2.tauri.app/develop/tests/mocking/
- jsdom lacks WebCrypto: polyfill `window.crypto.getRandomValues` with Node's `randomFillSync` in `beforeAll`. https://v2.tauri.app/develop/tests/mocking/

```ts
import { mockIPC, clearMocks } from '@tauri-apps/api/mocks';
import { invoke } from '@tauri-apps/api/core';
afterEach(() => { clearMocks(); });

test('invoke add', async () => {
  mockIPC((cmd, args) => {
    if (cmd === 'add') return (args.a as number) + (args.b as number);
  });
  const spy = vi.spyOn(window.__TAURI_INTERNALS__, 'invoke');
  expect(invoke('add', { a: 12, b: 15 })).resolves.toBe(27);
  expect(spy).toHaveBeenCalled();
});
```

https://v2.tauri.app/develop/tests/mocking/

- Events (since 2.7.0): `mockIPC(() => {}, { shouldMockEvents: true })`. `emitTo` and `emit_filter` are not mocked. https://v2.tauri.app/develop/tests/mocking/
- `mockWindows('main', 'second', 'third')` fakes which windows exist, the first label being the current one. Window properties still need `mockIPC`. https://v2.tauri.app/develop/tests/mocking/
- Shell or sidecar output: read the handler id from `args.message.onEventFn`, call `window['_' + id]` with `{ event: 'Stdout', payload }`, and emit `Terminated` LAST or the promise never resolves. https://v2.tauri.app/develop/tests/mocking/

## End-to-end with WebdriverIO

- Scaffold with `npm create wdio@latest ./`, choose Desktop Testing, then Tauri. https://v2.tauri.app/develop/tests/webdriver/
- Driver providers: `embedded` (default, WebDriver server inside the app through `tauri-plugin-wdio-webdriver`, the only free macOS route), `external` (native drivers via `tauri-driver`, Windows/Linux), `crabnebula` (cross-platform fork, paid key for macOS). `tauri-plugin-wdio` adds `browser.tauri.execute()`, IPC mocking (`browser.tauri.mock()`) and log capture. https://v2.tauri.app/develop/tests/webdriver/ https://v2.tauri.app/develop/tests/mocking/

```typescript
services: [['tauri', { appBinaryPath: './src-tauri/target/release/my-tauri-app', driverProvider: 'embedded' }]]
```

https://v2.tauri.app/develop/tests/webdriver/

- Browser mode runs the frontend in Chrome against the Vite dev server and intercepts `invoke()`: no binary, driver or plugin. https://v2.tauri.app/develop/tests/webdriver/

## Manual tauri-driver

- `cargo install tauri-driver --locked`; its version is independent of the other Tauri versions. Mobile needs Appium 2. https://v2.tauri.app/develop/tests/webdriver/manual-setup/ https://v2.tauri.app/develop/tests/webdriver/ci/
- Linux needs `WebKitWebDriver` (Debian package `webkit2gtk-driver`). https://v2.tauri.app/develop/tests/webdriver/manual-setup/
- Windows: `msedgedriver.exe` must match the installed Edge version exactly, or tests hang on connect. Put it on `PATH` or pass `--native-driver`; `msedgedriver-tool` fetches the right one. Automate this in CI. https://v2.tauri.app/develop/tests/webdriver/manual-setup/
- Build first with `tauri build --debug --no-bundle`, and give that step a long timeout (the Selenium example uses 2 minutes). https://v2.tauri.app/develop/tests/webdriver/example/selenium/ https://v2.tauri.app/develop/tests/webdriver/example/webdriverio/
- Capabilities: `tauri:options: { application }`, browser name `wry`, server `http://127.0.0.1:4444/`. https://v2.tauri.app/develop/tests/webdriver/example/selenium/

```javascript
capabilities.set('tauri:options', { application });
capabilities.setBrowserName('wry');
```

https://v2.tauri.app/develop/tests/webdriver/example/selenium/

- Kill the spawned `tauri-driver` on exit and on SIGINT, SIGTERM, SIGHUP, SIGBREAK: WDIO `afterSession` does not run when the session fails to start. https://v2.tauri.app/develop/tests/webdriver/example/webdriverio/
- WDIO specs path: use `./test/specs/**/*.js` (the page's sample config shows a wrong path). https://v2.tauri.app/develop/tests/webdriver/example/webdriverio/

## CI on GitHub Actions

- Linux needs a virtual display: install `webkit2gtk-driver` and `xvfb` (plus `libwebkit2gtk-4.1-dev`, `libayatana-appindicator3-dev`) and run tests under `xvfb-run`. Windows runs as is. https://v2.tauri.app/develop/tests/webdriver/ci/
- Run `cargo test` before WebDriver tests so a broken app fails fast; cache with `Swatinem/rust-cache@v2` and `workspaces: src-tauri`. https://v2.tauri.app/develop/tests/webdriver/ci/
- With the isolation pattern, headless CI may stall on key generation for lack of entropy: install `haveged`. https://v2.tauri.app/concept/inter-process-communication/isolation/
