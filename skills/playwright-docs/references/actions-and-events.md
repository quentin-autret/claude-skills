# Actions, events and navigation

Auto-waiting, input, dialogs, downloads, popups, navigation and in-page evaluation in Playwright Test 1.64.1.

## Contents

- Actionability checks per action
- Input actions
- Waiting for events: the promise-first pattern
- Dialogs
- Downloads and uploads
- Pages and popups
- Navigation, hydration and BFCache
- evaluate, init scripts and handles

## Actionability checks per action

Before an action Playwright waits for its checks. If they do not pass within the timeout the action fails with `TimeoutError`.
https://playwright.dev/docs/actionability#introduction

| Action | Checks |
| --- | --- |
| `click`, `dblclick`, `check`, `uncheck`, `setChecked`, `tap` | Visible, Stable, Receives Events, Enabled |
| `hover`, `dragTo` | Visible, Stable, Receives Events |
| `fill`, `clear` | Visible, Enabled, Editable |
| `selectOption` | Visible, Enabled |
| `screenshot` | Visible, Stable |
| `selectText` | Visible |
| `scrollIntoViewIfNeeded` | Stable |
| `focus`, `blur`, `press`, `pressSequentially`, `setInputFiles`, `dispatchEvent` | none |

https://playwright.dev/docs/actionability#introduction

Edges of the definitions (https://playwright.dev/docs/actionability#visible):

- Visible: non-empty bounding box and no `visibility:hidden`. `display:none` and zero size are not visible. `opacity:0` IS visible.
- Elements inside a hidden iframe count as hidden for actions, `isVisible` and `toBeVisible`. https://playwright.dev/docs/release-notes#version-164
- Stable: same bounding box for 2 consecutive animation frames.
- Enabled fails on `[disabled]` button, select, input, textarea, option and optgroup (also when inside `fieldset[disabled]`), and descendants of `[aria-disabled=true]`.
- Receives Events: the element is the hit target at the action point. An overlay there fails the check.

`force: true` skips the non-essential checks. A forced click on a covered element is indistinguishable from a real bug, so use it only when the overlay is intended.
https://playwright.dev/docs/input#forcing-the-click

Overlays that appear at random times (cookie banners) are handled by `page.addLocatorHandler`. After the handler runs, Playwright waits until the overlay is hidden (opt out with `noWaitAfter`).
```js
await page.addLocatorHandler(
  page.getByRole('heading', { name: 'Hej! You are in control of your cookies.' }),
  async () => { await page.getByRole('button', { name: 'Accept all' }).click(); });
```
https://playwright.dev/docs/release-notes#version-142 and https://playwright.dev/docs/release-notes#version-144

## Input actions

- Default: `fill()`. It works on `<input>`, `<textarea>` and `[contenteditable]`. Date and time inputs take ISO-like strings.
  ```js
  await page.getByLabel('Birth date').fill('2020-02-02');
  await page.getByLabel('Local time').fill('2020-03-02T05:15');
  ```
  https://playwright.dev/docs/input#text-input
- Escape hatch: `pressSequentially()` only when the page has key-by-key handling.
  https://playwright.dev/docs/input#type-characters
- `selectOption('blue')` matches value or label. `{ label: 'Blue' }` forces label. An array selects several.
  https://playwright.dev/docs/input#select-options
- `click({ modifiers: ['ControlOrMeta'] })` gives Ctrl on Windows and Linux, Meta on macOS.
  https://playwright.dev/docs/input#mouse-click
- Keys follow `KeyboardEvent.key`. A single character is case-sensitive: `Shift+A` gives `A`, `Shift-a` gives `a` as if CapsLock were toggled.
  https://playwright.dev/docs/input#keys-and-shortcuts
- `dispatchEvent('click')` dispatches the event without real user conditions.
  https://playwright.dev/docs/input#programmatic-click
- Drag and drop: `locator.dragTo(target)`. When the page relies on `dragover`, a manual drag needs at least 2 mouse moves: hover the target twice before `mouse.up()`.
  https://playwright.dev/docs/input#dragging-manually
- `locator.drop({ files | data })` simulates an external drag-and-drop onto an upload zone.
  https://playwright.dev/docs/release-notes#version-160
- Scrolling is automatic before actions. For infinite lists `scrollIntoViewIfNeeded()` on the last element is "the most reliable way". Actions also accept `scroll: 'none'` to disable auto-scroll.
  https://playwright.dev/docs/input#scrolling and https://playwright.dev/docs/release-notes#version-162
- Touch: `dispatchEvent('touchstart', { touches, changedTouches, targetTouches })` does not set `Event.isTrusted`, so disable any `isTrusted` check in the app under test.
  https://playwright.dev/docs/touch-events
- Most actions and web-first assertions accept `signal: AbortSignal`. A signal does NOT disable the default timeout, pass `timeout: 0` for that.
  https://playwright.dev/docs/release-notes#version-162

## Waiting for events: the promise-first pattern

Create the wait promise BEFORE the triggering action, do not await it yet, then await it after. This applies to `waitForEvent('popup' | 'download' | 'filechooser')`, `waitForRequest`, `waitForResponse` and `context.waitForEvent('page')`.
```js
const popupPromise = page.waitForEvent('popup');
await page.getByText('open the popup').click();
const popup = await popupPromise;
```
https://playwright.dev/docs/events#waiting-for-event

`page.on` / `page.once` / `page.off` handle events that happen at random times.
https://playwright.dev/docs/events#addingremoving-event-listener

## Dialogs

- With no `dialog` listener, `alert`, `confirm`, `prompt` and `beforeunload` are auto-dismissed.
- Register the listener before the action. Once a listener exists it MUST call `accept()` or `dismiss()`, otherwise the action hangs.
  ```js
  page.on('dialog', dialog => console.log(dialog.message()));
  await page.getByRole('button').click(); // Will hang here
  ```
  https://playwright.dev/docs/dialogs#alert-confirm-prompt-dialogs
- `beforeunload` fires only with `page.close({ runBeforeUnload: true })`, and then `close` does not wait for the page to close.
  https://playwright.dev/docs/dialogs#beforeunload-dialog
- Print dialogs: stub `window.print` after load and before the click.
  ```js
  await page.evaluate('(() => {window.waitForPrintDialog = new Promise(f => window.print = f);})()');
  await page.getByText('Print it!').click();
  await page.waitForFunction('window.waitForPrintDialog');
  ```
  https://playwright.dev/docs/dialogs#print-dialogs

## Downloads and uploads

- Downloads land in a temp folder that is DELETED when the browser context closes. Persist with `download.saveAs(...)`.
  ```js
  const downloadPromise = page.waitForEvent('download');
  await page.getByText('Download file').click();
  const download = await downloadPromise;
  await download.saveAs('/path/to/save/at/' + download.suggestedFilename());
  ```
  https://playwright.dev/docs/downloads#introduction
- `download.path()` and `createReadStream()` throw for failed or cancelled downloads.
  https://playwright.dev/docs/release-notes#version-140
- `setInputFiles(path)` needs an `input type=file`. Relative paths resolve against the current working directory. `[]` clears. A `{ name, mimeType, buffer }` payload and directories are accepted.
  https://playwright.dev/docs/input#upload-files
- No input in hand: wait for `filechooser` first, then click, then `fileChooser.setFiles(...)`.
  https://playwright.dev/docs/input#upload-files

## Pages and popups

- Every page in a context behaves as focused. No bring-to-front is needed. Pages inherit context emulation and routes.
  https://playwright.dev/docs/pages#multiple-pages
- New tab from `target="_blank"`: `context.waitForEvent('page')`. Unknown trigger: `context.on('page', ...)` then `waitForLoadState()`.
  https://playwright.dev/docs/pages#handling-new-pages
- The page `popup` event fires in addition to the context `page` event, only for popups opened by that page.
  https://playwright.dev/docs/pages#handling-popups

## Navigation, hydration and BFCache

- `page.goto` waits for `load`. There is no universal "page is loaded" moment. Actions auto-wait, so add no manual load waits before clicking.
  https://playwright.dev/docs/navigations#when-is-the-page-loaded
- A click that triggers several navigations: wait for the final URL.
  ```js
  await page.getByText('Click me').click();
  await page.waitForURL('**/login');
  ```
  https://playwright.dev/docs/navigations#waiting-for-navigation
- Hydration: if a click seems to do nothing or typed text disappears, suspect a control enabled before its listeners attached. Reproduce with DevTools "Slow 3G". The fix belongs in the app: keep controls disabled until hydration ends.
  https://playwright.dev/docs/navigations#hydration
- BFCache is disabled in all browsers by default. Testing BFCache restores is not supported: a restore skips `commit`, `domcontentloaded` and `load`, so `goBack()` into one times out.
  https://playwright.dev/docs/navigations#backforward-cache-bfcache

## evaluate, init scripts and handles

- `page.evaluate` runs in the browser. Node variables are invisible there. Pass ONE argument, an object for several values.
  ```js
  const data = 'some data';
  await page.evaluate(data => { window.myApp.use(data); }, data);
  await page.evaluate(({ button1, button2 }) => button1.textContent + button2.textContent, { button1, button2 });
  ```
  https://playwright.dev/docs/evaluating#evaluation-argument
- `evaluate` awaits returned promises.
  https://playwright.dev/docs/evaluating#introduction
- `page.addInitScript` / `context.addInitScript` run before any page script. Use them for deterministic globals.
  ```js
  await page.addInitScript(value => { Math.random = () => value; }, 42);
  ```
  https://playwright.dev/docs/evaluating#init-scripts
- ElementHandle is discouraged. A handle points at one element and goes stale after a re-render. Use Locator. If a handle is unavoidable, get it with `page.waitForSelector(...)` and `dispose()` it.
  https://playwright.dev/docs/handles#locator-vs-elementhandle and https://playwright.dev/docs/handles#handle-lifecycle
