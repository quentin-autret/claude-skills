# Semgrep in CI

## Contents

- 2 setups
- Installing in CI
- GitHub Actions
- GitLab CI/CD
- Other providers
- Full and diff-aware scans
- Blocking and errors
- Environment variables
- Tokens
- Monorepos
- Git errors in PR scans
- Old patterns

Abbreviations used below: ENV = https://docs.semgrep.dev/semgrep-ci/ci-environment-variables , SAMPLES = https://docs.semgrep.dev/semgrep-ci/sample-ci-configs

## 2 setups

- AppSec Platform: the job runs `semgrep ci` with `SEMGREP_APP_TOKEN`. It adds Supply Chain, Secrets, PR and MR comments and finding history. https://docs.semgrep.dev/deployment/add-semgrep-to-ci https://docs.semgrep.dev/deployment/oss-deployment
- Semgrep CE stand-alone: the job runs `semgrep scan --config ...`. SAST only, no usage limits. For a strictly open source setup, run your own rules or rules whose `metadata.license` fits. https://docs.semgrep.dev/deployment/oss-deployment
- `semgrep ci` with a token uses the organization policies, private rules included. `semgrep --config auto` only uses Registry rules. `--config p/default` comes closest locally. https://docs.semgrep.dev/kb/semgrep-ci/ci-vs-cli
- Run the same version, installed the same way, locally and in CI. https://docs.semgrep.dev/kb/semgrep-ci/ci-vs-cli

## Installing in CI

- The `semgrep/semgrep` Docker image is the recommended method. Otherwise `pipx install semgrep` or `uv tool install semgrep`. https://docs.semgrep.dev/deployment/add-semgrep-to-other-ci-providers
- The image is Alpine 3.22 with Python 3.11, `bash`, `jq` and `curl`. `jq` and `curl` may be removed later, so run `apk add bash jq curl` if a script needs them. https://docs.semgrep.dev/semgrep-ci/packages-in-semgrep-docker
- `semgrep/semgrep:latest-nonroot` breaks with GitHub's `container: image:` (checkout permissions). Run it through `docker run` after `sudo chown -R 1000:1000 .`. https://docs.semgrep.dev/kb/semgrep-ci/using-nonroot-docker-image-with-gha

## GitHub Actions

```yaml
name: Semgrep
on:
  pull_request: {}
  workflow_dispatch: {}
  push:
    branches: [main, master]
    paths: [.github/workflows/semgrep.yml]
  schedule:
    - cron: '20 17 * * *'   # pick a random time
permissions:
  contents: read
jobs:
  semgrep:
    name: semgrep/ci
    runs-on: ubuntu-latest
    container:
      image: semgrep/semgrep
    if: (github.actor != 'dependabot[bot]')
    steps:
      - uses: actions/checkout@v6
      - run: semgrep ci
        env:
          SEMGREP_APP_TOKEN: ${{ secrets.SEMGREP_APP_TOKEN }}
```

Source: (SAMPLES)

- Skip Dependabot PRs to avoid permission issues. (SAMPLES)
- `push` with both `branches` and `paths` runs only when both filters match. (SAMPLES)
- CE variant: same triggers but `push` filters on branches only (no `paths`), no token, last step `semgrep scan --config auto`. (SAMPLES)
- Reusable workflow: a central `.github/workflows/semgrep.yml` with `on: workflow_call`, called by `uses: ORG/REPO/.github/workflows/semgrep.yml@main` with `secrets: inherit`. Each caller still needs its own workflow file and its own `SEMGREP_APP_TOKEN` secret. https://docs.semgrep.dev/kb/semgrep-ci/github-reusable-workflows-semgrep

## GitLab CI/CD

```yaml
semgrep:
  image: semgrep/semgrep
  script: semgrep ci
  rules:
    - if: $CI_PIPELINE_SOURCE == "web"
    - if: $CI_MERGE_REQUEST_IID
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
  variables:
    SEMGREP_APP_TOKEN: $SEMGREP_APP_TOKEN
```

Source: (SAMPLES)

- A diff-aware scan starts automatically when `$CI_MERGE_REQUEST_IID` is set. https://docs.semgrep.dev/kb/semgrep-ci/trigger-diff-scans-env-var
- Set `GIT_DEPTH` to 20 or 50, a shallow clone breaks the merge-base lookup. https://docs.semgrep.dev/kb/semgrep-ci/git-command-errors
- CE variant: `script: semgrep scan --config auto .`. (SAMPLES)

## Other providers

- Outside GitHub Actions and GitLab, diff-aware scans need `SEMGREP_BASELINE_REF` (or `SEMGREP_BASELINE_COMMIT`) and a fetched baseline ref. For Jenkins and providers without Semgrep guidance, create a separate CI job for it. The Bitbucket and Azure samples set it inside the PR path instead. https://docs.semgrep.dev/deployment/customize-ci-jobs https://docs.semgrep.dev/kb/semgrep-ci/trigger-diff-scans-env-var
- Bitbucket: `export SEMGREP_BASELINE_REF="origin/main"` and `git fetch origin "+refs/heads/*:refs/remotes/origin/*"` before `semgrep ci`. `BITBUCKET_TOKEN` (a Bitbucket Cloud personal access token) is optional, worth trying when PR comments are missing or code links do not display. Out of memory: lower `-j` or add `size: 2x`. (SAMPLES) (ENV) https://docs.semgrep.dev/kb/semgrep-ci/trigger-diff-scans-env-var
- Azure Pipelines: `fetchDepth: 20`, `persistCredentials: true`, `SEMGREP_PR_ID=$(System.PullRequest.PullRequestId)`, `SEMGREP_BASELINE_REF='origin/master'` after `git fetch origin master:origin/master`, plus a build validation branch policy. TFVC is not supported. (SAMPLES)
- Jenkins: a separate diff job with `SEMGREP_BASELINE_REF = "main"` and `SEMGREP_PR_ID = env.CHANGE_ID`. (SAMPLES)
- Only GitHub Actions and GitLab feed their native security dashboards. For GitHub, write `semgrep ci --sarif > semgrep.sarif`, then upload it with `github/codeql-action/upload-sarif@v3` under `if: always()` and `security-events: write`. (SAMPLES) https://docs.semgrep.dev/kb/semgrep-ci/upload-ci-findings-to-github

## Full and diff-aware scans

- Run full scans on the default branch on a schedule, diff-aware scans on PRs and MRs. Full scans on every branch duplicate each finding per branch. https://docs.semgrep.dev/kb/semgrep-ci/why-duplicate-findings
- Cross-file analysis never runs in a diff-aware scan. https://docs.semgrep.dev/deployment/customize-ci-jobs
- `SEMGREP_BASELINE_REF` has no effect on GitHub Actions and GitLab, and cannot turn their PR scans into full scans. https://docs.semgrep.dev/deployment/add-semgrep-to-ci
- Full scans propagate triage to matching findings on other branches, diff-aware scans do not. https://docs.semgrep.dev/semgrep-ci/findings-ci

## Blocking and errors

- On the platform, only rules in Block mode (Policies page) fail the job. Other modes: Monitor, Comment, Disabled. https://docs.semgrep.dev/semgrep-ci/configuring-blocking-and-errors-in-ci
- Without the platform, every finding blocks and `semgrep ci` exits 1. `semgrep scan` exits 0 unless `--error`. https://docs.semgrep.dev/kb/semgrep-ci/ci-vs-cli
- On an internal error, `semgrep ci` sends a crash report and exits 0, so a broken scan can pass silently. Use `semgrep ci --no-suppress-errors` to fail on errors. `semgrep ci || true` never fails. https://docs.semgrep.dev/semgrep-ci/configuring-blocking-and-errors-in-ci
- Enforcing the merge block after exit code 1 depends on your CI provider. https://docs.semgrep.dev/semgrep-ci/configuring-blocking-and-errors-in-ci

## Environment variables

All from (ENV) unless a URL is given.

| Variable | Use |
| :-- | :-- |
| `SEMGREP_APP_TOKEN` | Platform login. Never set it together with `SEMGREP_RULES` |
| `SEMGREP_RULES` | Rule sources, space separated in a shell, newline separated in a YAML `>-` block (https://docs.semgrep.dev/deployment/oss-deployment) |
| `SEMGREP_BASELINE_COMMIT` | Baseline hash, ideally `git merge-base main feature-branch` |
| `SEMGREP_TIMEOUT` | Seconds per rule and file, default 5. 0 removes the limit, not recommended (https://docs.semgrep.dev/deployment/customize-ci-jobs) |
| `SEMGREP_GIT_COMMAND_TIMEOUT` | Default 300 s per git command |
| `SEMGREP_GHA_MIN_FETCH_DEPTH` | Raise it only when PR scans show untouched findings |
| `SEMGREP_REPO_DISPLAY_NAME` | Project name on the platform (1.61.1+) |
| `SEMGREP_PR_ID` | PR or MR number, enables PR or MR comments. Required for hyperlinks on Azure Pipelines (https://docs.semgrep.dev/deployment/add-semgrep-to-other-ci-providers) |
| `SEMGREP_BRANCH`, `SEMGREP_COMMIT`, `SEMGREP_REPO_NAME`, `SEMGREP_REPO_URL` | Override autodetected values to fix broken links and missing PR comments |
| `SEMGREP_JOB_URL` | Same troubleshooting set, for providers other than GitHub Actions and GitLab (https://docs.semgrep.dev/deployment/add-semgrep-to-other-ci-providers) |
| `SEMGREP_APP_URL` | Single-tenant platform URL |

- Get the base job working first, then add variables. https://docs.semgrep.dev/deployment/add-semgrep-to-other-ci-providers

## Tokens

- CI needs a token with the Agent (CI) scope. Web API tokens cannot send findings. Scopes are exclusive, and a Read-only token can never be widened. https://docs.semgrep.dev/deployment/tokens
- The value is shown once. API tokens stay valid until revoked, even after the creator leaves. `semgrep logout` removes the local token without revoking it. https://docs.semgrep.dev/deployment/tokens

## Monorepos

- A repository is 1 project. `semgrep ci --subdir src/moduleA/` makes a project per folder. https://docs.semgrep.dev/kb/semgrep-ci/scan-monorepo-in-parts
- Give each module its own `SEMGREP_REPO_DISPLAY_NAME`. Scans of different modules must never share a name, to keep finding status consistent. Keep `SEMGREP_REPO_NAME` unchanged. https://docs.semgrep.dev/kb/semgrep-ci/scan-monorepo-in-parts
- `--include` / `--exclude` splits require `SEMGREP_REPO_DISPLAY_NAME`, otherwise findings in unscanned parts may be closed. Turn off historical secrets scanning when splitting. https://docs.semgrep.dev/kb/semgrep-ci/scan-monorepo-in-parts

## Git errors in PR scans

- `Not a valid object name master` (exit 128) comes from a shallow clone, an unfetched baseline or an unsafe directory (`git config --global --add safe.directory $(pwd)`). `git diff --merge-base` needs git 2.30+. https://docs.semgrep.dev/kb/semgrep-ci/git-command-errors

## Old patterns

| Old | Current | Source |
| :-- | :-- | :-- |
| `SEMGREP_BASELINE_REF` | `SEMGREP_BASELINE_COMMIT`, itself superseded by `--baseline-commit`. The provider samples still use `SEMGREP_BASELINE_REF` | (ENV) (SAMPLES) |
| `--config` in `semgrep ci` | Rules on the Policies page | https://docs.semgrep.dev/kb/semgrep-code/reduce-false-positives |
