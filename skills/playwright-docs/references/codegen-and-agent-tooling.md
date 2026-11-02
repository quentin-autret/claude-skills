# Codegen, Library mode and agent tooling

Code generation, the Library versus Test split, release channels, Test Agents, playwright-cli and Playwright MCP, as documented for 1.64.1.

## Contents

- Codegen
- Library versus Test
- Release channels
- Test Agents
- playwright-cli
- Playwright MCP

## Codegen

- `npx playwright codegen [url]` opens a browser and the Inspector. It prefers role, text and test id locators and refines a locator until it is unique. The output is a draft to improve by hand.
  https://playwright.dev/docs/codegen-intro#recording-a-test and https://playwright.dev/docs/codegen#running-codegen
- The toolbar records assertions: visibility, text, value.
  https://playwright.dev/docs/codegen#recording-a-test
- The default codegen viewport is fixed. Emulate with flags.
  ```bash
  npx playwright codegen --viewport-size="800,600" playwright.dev
  npx playwright codegen --timezone="Europe/Rome" --geolocation="41.890221,12.492348" --lang="it-IT" bing.com/maps
  ```
  https://playwright.dev/docs/codegen#emulation
- `--save-storage=auth.json` / `--load-storage=auth.json` keep cookies, local storage and IndexedDB. The file is sensitive: gitignore or delete it.
  https://playwright.dev/docs/codegen#preserve-authenticated-state
- `--http-credentials="user:pass"` sends the credentials to ANY origin that asks and writes them into the generated code.
  https://playwright.dev/docs/codegen#authenticate-with-http-credentials
- `--user-data-dir` cannot point at the default Chrome profile (blocked for automation since Chrome 136). Use a dedicated directory.
  https://playwright.dev/docs/codegen#use-existing-userdatadir
- Custom setup such as routes: call `page.pause()` in a headed script to get the recorder.
  ```js
  const browser = await chromium.launch({ headless: false });
  const context = await browser.newContext();
  await context.route('**/*', route => route.continue());
  const page = await context.newPage();
  await page.pause();
  ```
  https://playwright.dev/docs/codegen#record-using-custom-setup
- `npx playwright open` no longer records. Use `codegen`.
  https://playwright.dev/docs/release-notes#version-154

## Library versus Test

- For end-to-end tests use `@playwright/test` (`npm init playwright@latest`), not the `playwright` library (`npm i -D playwright`).
  https://playwright.dev/docs/library
- Library has no fixtures, no web-first assertions, and needs explicit `launch`, `newContext`, `context.close()`, `browser.close()`. Its operations time out after 30 s by default. In the runner most operations have no timeout of their own, only the test does.
  https://playwright.dev/docs/library#key-differences
- `assert(await page.title() === 'Example Domain')` is not web-first. In the runner use `await expect(page).toHaveTitle(...)`.
  https://playwright.dev/docs/library#library-example
- `PLAYWRIGHT_SKIP_BROWSER_DOWNLOAD=1` skips downloads when browsers are managed elsewhere. `PLAYWRIGHT_DOWNLOAD_HOST` points at an internal mirror.
  https://playwright.dev/docs/library#browser-downloads
- `await using page = await context.newPage();` disposes pages, routes and init scripts at scope end.
  https://playwright.dev/docs/release-notes#version-159

## Release channels

npm dist tags: `latest` (stable), `next` (daily canary from `main`), `beta` (each commit after a release branch is cut). `npm install -D @playwright/test@next`.
https://playwright.dev/docs/canary-releases

## Test Agents

- 3 agents: planner (explores and writes a Markdown plan in `specs/`), generator (turns the plan into tests in `tests/`, checking locators live), healer (replays failures and patches locators, waits or data, and may skip a test it thinks is broken).
  https://playwright.dev/docs/test-agents
- `npx playwright init-agents --loop=vscode|claude|codex|opencode`. Regenerate the definitions after every Playwright upgrade.
  https://playwright.dev/docs/test-agents#getting-started
- The planner runs a seed test (`tests/seed.spec.ts`) for setup, fixtures and hooks, and copies its style.
  ```js
  import { test, expect } from './fixtures';
  test('seed', async ({ page }) => {
    // this test uses custom fixtures from ./fixtures
  });
  ```
  https://playwright.dev/docs/test-agents#planner
- Generator output "may include initial errors". Generated files start with `// spec:` and `// seed:` comments.
  https://playwright.dev/docs/test-agents#generator
- `--run-agents <missing|all|none>` (default `none`) generates code for `page.perform`.
  https://playwright.dev/docs/test-cli#all-options

## playwright-cli

- For coding agents that want token-efficient browser control without big tool schemas. MCP suits loops that need persistent state and reasoning over page structure.
  https://playwright.dev/docs/getting-started-cli#playwright-cli-vs-playwright-mcp
- `npm install -g @playwright/cli@latest`, or locally then `npx playwright cli --help`. `playwright-cli install --skills` installs agent skills.
  https://playwright.dev/docs/getting-started-cli#installation
- `playwright-cli snapshot` returns refs like `e15`, then `playwright-cli click e15`. Role selectors also work: `"role=button[name=Submit]"`.
  https://playwright.dev/docs/getting-started-cli#targeting-elements
- Headless by default. The profile lives in memory unless `--persistent`. Named sessions with `-s=name`. A headless session closes after 1 hour idle (`--idle-timeout=<ms>`, `0` disables).
  https://playwright.dev/docs/getting-started-cli#sessions
- WebMCP tool names, schemas and results come from the page: treat them as untrusted input.
  https://playwright.dev/docs/getting-started-cli#webmcp

## Playwright MCP

- Config: `{"mcpServers":{"playwright":{"command":"npx","args":["@playwright/mcp@latest"]}}}`.
  https://playwright.dev/docs/getting-started-mcp#installation
- It works on accessibility snapshots with `ref=eN` references, no vision model.
  https://playwright.dev/docs/getting-started-mcp#accessibility-snapshots
- MCP runs HEADED by default (pass `--headless`), the opposite of the CLI.
  https://playwright.dev/docs/getting-started-mcp#headed-mode
- Profile: persistent per workspace by default, `--isolated` for a fresh one (initial state via `--storage-state`), `--extension` for existing tabs.
  https://playwright.dev/docs/getting-started-mcp#user-profile
- A headless browser closes after 1 hour without tool calls. With `--isolated` its cookies are then lost.
  https://playwright.dev/docs/getting-started-mcp#idle-timeout
- `browser_run_code_unsafe` is RCE-equivalent. Enable it only for trusted clients.
  https://playwright.dev/docs/getting-started-mcp#running-playwright-code
- Display-less hosts: `npx @playwright/mcp@latest --port 8931`, client URL `http://localhost:8931/mcp`. Set `PLAYWRIGHT_MCP_PING_TIMEOUT_MS` when a proxy drops the 5 s heartbeat.
  https://playwright.dev/docs/getting-started-mcp#standalone-server
