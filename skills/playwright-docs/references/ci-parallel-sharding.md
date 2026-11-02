# CI, parallelism and sharding

## Contents

- CI baseline
- GitHub Actions specifics
- Other CI providers
- Parallelism model
- Test locks and shared state
- Sharding and merged reports

## CI baseline

- Install with `npm ci`, then `npx playwright install --with-deps` (browsers plus OS packages). Browsers are NOT downloaded by `npm install`.
  https://playwright.dev/docs/ci#introduction and https://playwright.dev/docs/release-notes#version-138
- Install only what runs: `npx playwright install chromium --with-deps`. Headless-only runs without `channel` can add `--only-shell`.
  https://playwright.dev/docs/best-practices#optimize-browser-downloads-on-ci and https://playwright.dev/docs/browsers#chromium-headless-shell
- Caching browser binaries is not recommended: restore takes about as long as download, and Linux OS dependencies cannot be cached. If you cache anyway, key on the Playwright version.
  https://playwright.dev/docs/ci#caching-browsers
- Default `workers: process.env.CI ? 1 : undefined` for stable, reproducible runs. On strong self-hosted runners parallel is fine. For more throughput, shard rather than add workers.
  https://playwright.dev/docs/ci#workers
- ALWAYS set `globalTimeout`. With no upper bound, a hung suite is killed by the CI runner and NO report is written. Keep any job-level timeout well above it.
  ```js
  export default defineConfig({ globalTimeout: 60 * 60 * 1000 });
  ```
  https://playwright.dev/docs/ci#global-timeout
- `forbidOnly: !!process.env.CI` fails the run when a `test.only` was left in.
  https://playwright.dev/docs/test-configuration#basic-configuration
- `maxFailures: process.env.CI ? 10 : undefined` stops early and skips the rest.
  https://playwright.dev/docs/test-parallel#limit-failures-and-fail-fast
- Headed on Linux CI needs Xvfb: `xvfb-run npx playwright test`. The Docker image already has it.
  https://playwright.dev/docs/ci#running-headed
- `DEBUG=pw:browser npx playwright test` diagnoses `Failed to launch browser`.
  https://playwright.dev/docs/ci#debugging-browser-launches
- Treat traces, HTML reports and logs as secrets: they can hold test credentials and tokens. Upload only to trusted stores.
  https://playwright.dev/docs/ci-intro#properly-handling-secrets

## GitHub Actions specifics

- The workflow the docs generate (`npm init playwright@latest`). It sets no job-level `timeout-minutes`, the timeout lives in `globalTimeout` instead.
  ```yml
  name: Playwright Tests
  on:
    push:
      branches: [ main, master ]
    pull_request:
      branches: [ main, master ]
  jobs:
    test:
      runs-on: ubuntu-latest
      steps:
      - uses: actions/checkout@v6
      - uses: actions/setup-node@v6
        with:
          node-version: lts/*
      - run: npm ci
      - run: npx playwright install --with-deps
      - run: npx playwright test
      - uses: actions/upload-artifact@v4
        if: ${{ !cancelled() }}
        with:
          name: playwright-report
          path: playwright-report/
          retention-days: 30
  ```
  https://playwright.dev/docs/ci-intro#setting-up-github-actions
- Upload the report with `if: ${{ !cancelled() }}`, as the docs workflow does: the step runs after failed tests but is skipped when the run is cancelled.
  https://playwright.dev/docs/ci-intro#setting-up-github-actions
- Container job: `container.image: mcr.microsoft.com/playwright:v<VERSION>-noble` with `options: --user 1001`. No `playwright install` step is needed. A consistent environment helps screenshot and visual tests.
  https://playwright.dev/docs/ci#via-containers
- Fail fast on PRs, then always run the full suite. `--only-changed` follows the import graph, a heuristic that can miss tests.
  ```yml
  - uses: actions/checkout@v6
    with:
      fetch-depth: 0
  - run: npx playwright test --only-changed=origin/$GITHUB_BASE_REF
    if: github.event_name == 'pull_request'
  ```
  https://playwright.dev/docs/ci#fail-fast
- After a deployment: trigger on `deployment_status`, guard with `if: github.event.deployment_status.state == 'success'`, pass `PLAYWRIGHT_TEST_BASE_URL: ${{ github.event.deployment_status.target_url }}`.
  https://playwright.dev/docs/ci#on-deployment
- The `github` reporter adds failure annotations. Avoid it with a matrix strategy, where annotations multiply.
  https://playwright.dev/docs/test-reporters#github-actions-annotations

## Other CI providers

- CircleCI: the shard index is 0-based. `SHARD="$((${CIRCLE_NODE_INDEX}+1))"; npx playwright test --shard=${SHARD}/${CIRCLE_NODE_TOTAL}`. The `docker` executor's `medium` class has 2 cores, more workers than that cause timeouts.
  https://playwright.dev/docs/ci#circleci
- GitLab: `parallel: 7` with `--shard=$CI_NODE_INDEX/$CI_NODE_TOTAL`.
  https://playwright.dev/docs/ci#sharding
- Azure Pipelines: JUnit reporter with `outputFile: 'test-results/e2e-junit-results.xml'`, `PublishTestResults@2` with `condition: succeededOrFailed()`, and `CI: 'true'`.
  https://playwright.dev/docs/ci#uploading-playwright-report-folder-with-azure-pipelines
- Google Cloud Build needs `CI=true` in env.
  https://playwright.dev/docs/ci#google-cloud-build

## Parallelism model

- Files run in parallel across workers. Tests inside a file run in order in one worker. Each worker has its own browser and workers cannot communicate. The default worker count is 50% of logical cores.
  https://playwright.dev/docs/test-parallel#worker-processes and https://playwright.dev/docs/test-cli#all-options
- A worker is shut down after any test failure.
  https://playwright.dev/docs/test-parallel#worker-processes
- `fullyParallel: true` (config or project) parallelizes every test. `test.describe.configure({ mode: 'parallel' })` does it for one file or group, `mode: 'default'` opts back out. Parallel tests each run their own `beforeAll`.
  https://playwright.dev/docs/test-parallel#parallelize-tests-in-a-single-file
- Order across files is not guaranteed in parallel. With `workers: 1` files run alphabetically. Never define tests in a helper file imported by several specs.
  https://playwright.dev/docs/test-parallel#control-test-order
- `--shuffle [seed]` randomizes order to expose hidden inter-test dependencies. The seed is printed.
  https://playwright.dev/docs/release-notes#version-164
- `workerIndex` is unique and changes when a worker restarts. `parallelIndex` stays in `0..workers-1` and survives restarts. Env: `TEST_WORKER_INDEX`, `TEST_PARALLEL_INDEX`.
  https://playwright.dev/docs/test-parallel#worker-index-and-parallel-index

## Test locks and shared state

- Default for a shared resource: a named lock. Tests holding the same lock never run concurrently across files, workers and projects, everything else stays parallel.
  ```js
  test('update user settings', { lock: 'user-settings' }, async ({ page }) => {});
  test('reset the database', { lock: ['database', 'external-api'] }, async () => {});
  test.describe.configure({ lock: 'user-settings' });
  ```
  https://playwright.dev/docs/test-parallel#test-locks
- A describe group takes locks through its details argument. Repeated `describe.configure({ lock })` calls ADD locks, they never replace earlier ones, and nested groups are included.
  ```js
  test.describe('settings', { lock: 'user-settings' }, () => {});
  ```
  https://playwright.dev/docs/api/class-test#test-describe-configure-lock and https://playwright.dev/docs/api/class-test#test-describe-2
- All locks of a test are acquired before it starts and released when it finishes. A test with several locks waits until all are free.
  https://playwright.dev/docs/test-parallel#test-locks
- In default and serial modes, tests of a file run together in order, so a lock on any test is held for the whole file. A file mixing locked and unlocked tests therefore holds the lock for all of them, so keep locked tests in their own file.
  https://playwright.dev/docs/test-parallel#test-locks
- Unique backend data: derive ids from `testInfo.testId` and write files to `testInfo.outputPath('export.csv')`.
  https://playwright.dev/docs/test-parallel#avoiding-shared-state-in-parallel-tests

## Sharding and merged reports

- `npx playwright test --shard=1/4` per job. Only parallelizable units are split: files by default.
  https://playwright.dev/docs/test-sharding#sharding-tests-between-multiple-machines
- Use `fullyParallel: true` for balanced shards (test granularity). Without it, uneven files give uneven shards. Statically skipped tests are not counted.
  https://playwright.dev/docs/test-sharding#balancing-shards
- Merge: `reporter: process.env.CI ? 'blob' : 'html'`. Each shard uploads `blob-report` as `blob-report-<index>`. A merge job with `needs` and `if: ${{ !cancelled() }}` downloads `pattern: blob-report-*` with `merge-multiple: true`, then:
  ```bash
  npx playwright merge-reports --reporter html ./all-blob-reports
  ```
  Use `fail-fast: false` in the matrix.
  https://playwright.dev/docs/test-sharding#github-actions-example
- The blob reporter DELETES existing content of its output directory before writing.
  https://playwright.dev/docs/test-reporters#blob-reporter
- Same suite on several environments: set `tag: process.env.CI_ENVIRONMENT_NAME` so merged reports tell them apart. Blobs from different OSes need a merge config (`--config merge.config.ts`) naming the tests root.
  https://playwright.dev/docs/test-sharding#merging-reports-from-multiple-environments and https://playwright.dev/docs/test-sharding#merge-reports-cli
- Per-shard rerun of failures: `--last-failed --last-failed-file=.cache/last-run-shard-1.json`.
  https://playwright.dev/docs/running-tests#run-last-failed-tests
