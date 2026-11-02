# Debugging, traces, videos and reporters

## Contents

- CLI flags worth knowing
- Debug mode and the Inspector
- Trace and video modes
- Trace viewer
- UI mode
- Reporters

## CLI flags worth knowing

https://playwright.dev/docs/test-cli#all-options

- Positional arguments are regexes on the full file path. `file.spec.ts:42` runs the test at a line. `-g` filters titles, `-G` inverts.
  https://playwright.dev/docs/test-cli
- Defaults: workers 50% of logical cores, test timeout 30 s, no retries, reporter `list` locally and `dot` on CI, output `test-results`.
- `--last-failed` reruns the previous failures (stored in `<outputDir>/.last-run.json`).
  https://playwright.dev/docs/running-tests#run-last-failed-tests
- `--repeat-each <N>` and `--shuffle [seed]` hunt flakiness. `--fail-on-flaky-tests` fails a run that only passed on retry.
- `--max-failures <N>` / `-x` stop early. `--global-timeout <ms>` caps the run.
- `--list` collects without running. `--pass-with-no-tests`, `--no-deps` (skip project dependencies), `--ignore-snapshots`.
- `--reporter` REPLACES configured reporters. `--add-reporter` appends one.
- `--trace <mode>` forces tracing for one run.
- `--test-list <file>` / `--test-list-invert <file>` take the `--list` output format. Line and column numbers are ignored.
  https://playwright.dev/docs/test-cli#test-list

## Debug mode and the Inspector

- `--debug` equals `PWDEBUG=1` plus `--timeout=0 --max-failures=1 --headed --workers=1`. `--debug=cli` debugs over the CLI.
  ```bash
  npx playwright test example.spec.ts:10 --project=webkit --debug
  ```
  https://playwright.dev/docs/test-cli#common-options and https://playwright.dev/docs/debug#run-in-debug-mode
- `await page.pause()` stops at that point under `--debug`.
  https://playwright.dev/docs/debug#run-a-test-from-a-specific-breakpoint
- The Inspector shows actionability logs. An action stuck as pending is waiting on a check (visible, enabled, stable).
  https://playwright.dev/docs/debug#actionability-logs
- `PWDEBUG=console` plus `page.pause()` exposes `playwright.$`, `$$`, `inspect`, `locator`, `selector($0)` in DevTools.
  https://playwright.dev/docs/debug#browser-developer-tools
- `DEBUG=pw:api` prints verbose API logs. Opening the WebKit Inspector halts scripts and resets user agent and device emulation.
  https://playwright.dev/docs/debug#verbose-api-logs
- The VS Code extension gives breakpoints, live locator highlighting and "Record at cursor". Project dependencies and global setup can be run by hand from its sidebar.
  https://playwright.dev/docs/debug#vs-code-debugger and https://playwright.dev/docs/getting-started-vscode#advanced-features

## Trace and video modes

Recording is off by default. Default for CI: `trace: 'on-first-retry'` with `retries: process.env.CI ? 2 : 0`. Never `trace: 'on'` for every test, it is performance heavy. Without retries use `'retain-on-failure'`.
https://playwright.dev/docs/trace-viewer-intro#recording-a-trace and https://playwright.dev/docs/trace-viewer#tracing-on-ci

What each mode keeps, with `retries: 2` (https://playwright.dev/docs/test-use-options#trace-modes):

| Mode | Fail then pass | Fail every run |
| --- | --- | --- |
| `on` | run 1 + retry | all 3 runs |
| `retain-on-failure` | run 1 | all 3 runs |
| `retain-on-first-failure` | run 1 | run 1 |
| `retain-on-failure-and-retries` | run 1 + retry | all 3 runs |
| `on-first-retry` | retry 1 | retry 1 |
| `on-all-retries` | retry 1 | retries 1 and 2 |

- `video` accepts the same modes as `trace`. `screenshot` takes `'off'`, `'on'`, `'only-on-failure'`.
  https://playwright.dev/docs/test-use-options#recording-options and https://playwright.dev/docs/release-notes#version-161
- Videos are saved when the context closes. `page.video().path()` is available only after close. Default size fits the viewport into 800x800.
  ```ts
  video: { mode: 'on-first-retry', size: { width: 640, height: 480 } }
  ```
  https://playwright.dev/docs/videos#record-video
- Trace snapshots can be tuned: `trace: { mode: 'on', snapshots: { dom: true, aria: true, screen: true } }`.
  https://playwright.dev/docs/release-notes#version-163
- Canvas content is hidden in traces by default. Enable "Display canvas content" in the viewer.
  https://playwright.dev/docs/release-notes#version-150
- Outside the runner: `context.tracing.start({ screenshots: true, snapshots: true })` before creating the page, `context.tracing.stop({ path: 'trace.zip' })` at the end.
  https://playwright.dev/docs/trace-viewer#tracing-on-ci

## Trace viewer

- `npx playwright show-trace path/to/trace.zip`, or a URL. trace.playwright.dev loads the file in the browser and sends no data out.
  https://playwright.dev/docs/trace-viewer#opening-trace-viewer
- Tabs: actions with Before / Action / After DOM snapshots (the Action snapshot marks the click point), Source, Call, Log, Errors, Console, Network, Attachments with an image diff slider.
  https://playwright.dev/docs/trace-viewer#trace-viewer-features
- Prefer traces over videos and screenshots to debug CI failures.
  https://playwright.dev/docs/best-practices#debugging-on-ci

## UI mode

- `npx playwright test --ui`: watch mode, time-travel traces of every test, locator picker, filters by project, tag and status.
  https://playwright.dev/docs/test-ui-mode#opening-ui-mode
- Setup projects do NOT run automatically in UI mode. Run them first.
  https://playwright.dev/docs/test-ui-mode#filtering-tests
- `--ui-host=0.0.0.0 --ui-port=8080` (containers, Codespaces) exposes traces, passwords and secrets to the network.
  https://playwright.dev/docs/test-ui-mode#docker--github-codespaces

## Reporters

- Several at once: `reporter: [['list'], ['json', { outputFile: 'test-results.json' }]]`.
  https://playwright.dev/docs/test-reporters#multiple-reporters
- HTML opens automatically only on failure (`open: 'on-failure'`). On CI set `open: 'never'`. Output in `playwright-report`.
  ```js
  reporter: [['html', { open: 'never' }]]
  ```
  https://playwright.dev/docs/test-reporters#html-reporter
- Opening the report's `index.html` from disk does not work. Run `npx playwright show-report <folder>`, which also accepts the `.zip`.
  https://playwright.dev/docs/ci-intro#viewing-the-html-report
- `doNotInlineAssets` writes JS and CSS as separate files for hosting under a strict CSP.
  https://playwright.dev/docs/test-reporters#html-reporter
- JSON and JUnit print to stdout unless `outputFile` or `PLAYWRIGHT_JSON_OUTPUT_NAME` / `PLAYWRIGHT_JUNIT_OUTPUT_NAME` is set.
  https://playwright.dev/docs/test-reporters#json-reporter
- JUnit distinguishes `<error>` from `<failure>`. Check CI parsers.
  https://playwright.dev/docs/release-notes#version-159
- `list` has `printSteps` and `printFailuresInline`. `line` suits large suites. `omitTags` exists on list, line, dot, github and junit.
  https://playwright.dev/docs/test-reporters#list-reporter and https://playwright.dev/docs/release-notes#version-163
- `--reporter=perfetto` writes `test-results/perfetto.json`, one lane per worker, for ui.perfetto.dev.
  https://playwright.dev/docs/test-reporters#perfetto-reporter
- Custom reporter: a class implementing `Reporter` from `@playwright/test/reporter` (`onBegin`, `onTestBegin`, `onTestEnd`, `onEnd`), registered as `reporter: './my-reporter.ts'`.
  https://playwright.dev/docs/test-reporters#custom-reporters
