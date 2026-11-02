# Semgrep CLI

## Contents

- Install, pin and upgrade
- scan versus ci
- Choosing rules with --config
- Output formats
- Exit codes and errors
- Risky and experimental flags
- Metrics
- Old patterns

Abbreviations used below: CLI = https://docs.semgrep.dev/cli-reference , GS = https://docs.semgrep.dev/getting-started/cli

## Install, pin and upgrade

- Install with `pipx install semgrep` or `uv tool install semgrep`. Homebrew is best-effort and often lags behind. (GS) https://docs.semgrep.dev/update
- Pin: `pipx install semgrep==x.y.z` (`--force` replaces an install), `uv tool install semgrep==x.y.z`, one-off `uvx semgrep@x.y.z --version`, Docker `semgrep/semgrep:x.y.z`. Homebrew cannot hold several versions. https://docs.semgrep.dev/kb/semgrep-code/run-specific-version
- Upgrade: `pipx upgrade semgrep`, `uv tool upgrade semgrep`, `docker pull semgrep/semgrep:latest`. Then run `semgrep install-semgrep-pro` again to update the Pro engine binary. https://docs.semgrep.dev/update https://docs.semgrep.dev/semgrep-code/semgrep-pro-engine-intro
- With the AppSec Platform, stay on a current release. Old releases can fail to install or to talk to the platform. https://docs.semgrep.dev/kb/semgrep-code/run-specific-version
- The 2026-10-19 release rejects certificates from the compromised eMudhra authority. Older versions let a network attacker impersonate Semgrep servers and steal the token. Homebrew users also run `brew update && brew upgrade certifi ca-certificates`. https://docs.semgrep.dev/release-notes/2026-10-19
- Windows is beta. Set `PYTHONUTF8=1` for the user before `pipx install semgrep`, and use Python 3.9 or later. Semgrep does not run in parallel on Windows. https://docs.semgrep.dev/getting-started/quickstart-ce https://docs.semgrep.dev/customize-semgrep-ce (the Python 3.9 floor is stated on https://docs.semgrep.dev/semgrep-code/semgrep-pro-engine-intro)
- Proxies: `HTTP_PROXY`, `HTTPS_PROXY`, `ALL_PROXY`. An empty value warns and continues. An invalid value stops the scan. (CLI)

## scan versus ci

| | `semgrep scan` | `semgrep ci` |
| :-- | :-- | :-- |
| Account | None | Login or `SEMGREP_APP_TOKEN` required |
| Rules | `--config` | Organization policies. `--config` is not supported |
| Findings on exit | Exit 0 unless `--error` | Fails on blocking findings |
| PR or MR context | Full scan | Diff-aware, only new findings |
| `--autofix` | Applied | Ignored |

Sources: (CLI) (GS) https://docs.semgrep.dev/customize-semgrep-ce

- Product selection in `ci`: `--code`, `--supply-chain`, `--secrets`. (GS)
- `semgrep ci --dry-run` (with a dash) sends nothing and prints the JSON it would send. `semgrep scan --dryrun` (no dash) only previews `--autofix`. (CLI)
- `semgrep ci --oss-only` forces the open source engine even when the platform enables the Pro engine. (GS)
- Cross-file analysis for `ci` is switched on in the platform (Settings > General > Code), not by a flag. (GS)
- A 401 with Registry rules: run `semgrep logout` and retry. If that works the stored token was invalid, so run `semgrep login` again. https://docs.semgrep.dev/kb/semgrep-code/semgrep-scan-troubleshooting
- Subcommands besides `scan` and `ci`: `login`, `logout`, `publish`, `show`, `test`, `validate`, `lsp`, `mcp`, `install-semgrep-pro`. (CLI)

## Choosing rules with --config

```bash
semgrep scan --config=auto                          # Registry rules picked for the project
semgrep scan --config p/python --config rules/x.yaml   # repeat the flag per source
semgrep scan --config .                             # every YAML rule under the cwd
semgrep scan -e '$X == $X' --lang=py src/           # ephemeral rule
```

Sources: https://docs.semgrep.dev/running-rules https://docs.semgrep.dev/customize-semgrep-ce

- `--config` takes a file, a directory, a URL or a Registry name. Several sources need the `--config` flag repeated before each one. `SEMGREP_RULES` is the environment form. (CLI)
- `--config auto` sends the project URL to the Registry and turns metrics on. (CLI) https://docs.semgrep.dev/metrics
- `-e` requires `-l`/`--lang`. `--replacement` only works with `-e`. (CLI)
- Local rule ids get a prefix: the relative path from the cwd to the rule's directory, with separators turned into dots. A rule `rule-id-3` in `configs/` is reported as `configs.rule-id-3`. `--no-rewrite-rule-ids` turns this off. (CLI) https://docs.semgrep.dev/running-rules
- Rules in hidden directories are loaded when passed through `--config`. https://docs.semgrep.dev/running-rules
- `--exclude-rule=<id>` (repeatable) skips a rule. `--severity` filters by severity. (CLI)
- `--config=r/all` runs every public Registry rule and produces a lot of noise. If an invalid rule breaks it, run `semgrep --config r/all . -d` to cache the rules in `~/.semgrep/semgrep_rules.json`, then scan with `--config ~/.semgrep/semgrep_rules.json`. https://docs.semgrep.dev/kb/rules/run-all-available-rules
- Registry rulesets change over time, so findings can grow with unchanged code. https://docs.semgrep.dev/kb/semgrep-code/unexpected-new-findings
- `semgrep scan --show-supported-languages` prints the live language list. (CLI)

## Output formats

- Formats: `--text`, `--json`, `--sarif`, `--junit-xml`, `--gitlab-sast`, `--gitlab-secrets`, `--emacs`, `--vim`. Each has `--<format>-output=FILE` for an extra copy. `-o` writes the primary output to a file or URL. https://docs.semgrep.dev/customize-semgrep-ce

```bash
semgrep scan --text --output=semgrep.txt --json-output=semgrep.json --sarif-output=semgrep.sarif
```

- Findings go to stdout, progress and errors to stderr. `semgrep ci -o findings.txt 2> semgrep.log` separates them. https://docs.semgrep.dev/kb/semgrep-code/collect-cli-logs
- `--dataflow-traces` shows how values reach a finding (text and SARIF). (CLI)
- CE JSON lacks `fingerprint`, `is_ignored`, `lines`, `metavars` and the `semgrep.dev` metadata, and SARIF lacks `fingerprints`. In platform output `lines` is the matched text, not line numbers. https://docs.semgrep.dev/semgrep-appsec-platform/json-and-sarif
- JSON schema: `semgrep_output_v1.jsonschema` in semgrep/semgrep-interfaces. https://docs.semgrep.dev/customize-semgrep-ce
- Size limits: `--max-chars-per-line` 160, `--max-lines-per-finding` 10 (0 unlimited), `--max-match-context-size` (guards against minified files). (CLI)

## Exit codes and errors

| Code | Meaning |
| :-- | :-- |
| 0 | Success, or findings without `--error` |
| 1 | Findings with `--error` (or blocking findings in `ci`) |
| 2 | Semgrep failed |
| 3 | Invalid syntax in a scanned file (only with `--strict`) |
| 4 | Invalid pattern in a rule |
| 5 | Config is not valid YAML |
| 7 | At least 1 invalid rule |
| 8 | Language not understood |
| 13 | Invalid API key |

Source: (CLI). Wording differs between help blocks, so treat 0, 1 and 2 as the stable codes.

- `semgrep ci` suppresses internal errors and exits 0 by default. `--no-suppress-errors` makes them fail. https://docs.semgrep.dev/semgrep-ci/configuring-blocking-and-errors-in-ci
- `--strict` fails on warning-level errors and on invalid config. (CLI)
- Exit -9 or -11 is a crash, usually out of memory. https://docs.semgrep.dev/troubleshooting/semgrep
- `--baseline-commit=<sha>` reports only findings absent from that commit. It aborts outside Git, with unstaged changes or with an unknown hash. (CLI)

## Risky and experimental flags

- `--allow-local-builds` (experimental) builds the project to resolve dependencies, which can execute project code. Avoid it on untrusted code. (CLI)
- `--allow-untrusted-validators` runs Secrets validators from origins other than semgrep.dev. (CLI)
- `--remote`, `--files-with-matches` and `--incremental-output` require `--experimental`. (CLI)
- `--scan-unknown-extensions` only applies to files passed directly on the command line. (CLI)

## Metrics

- `--metrics=auto` (default) sends metrics when rules come from the Registry or when logged in, and not with only local rules or `-e`. Use `--metrics=off` or `SEMGREP_SEND_METRICS=off`. https://docs.semgrep.dev/metrics
- Never sent: source code, private rules. Sent: version, timings, anonymous id, IP, hashed project and config, finding counts, Registry ruleset names in plain text. https://docs.semgrep.dev/metrics
- GitLab's Semgrep SAST analyzer runs with `--metrics on`. https://docs.semgrep.dev/metrics

## Old patterns

| Old | Current | Source |
| :-- | :-- | :-- |
| `--semgrepignore-v2`, `--no-semgrepignore-v2` | Nothing: v2 is the only behavior | (CLI) |
| `--x-no-python-schema-validation` | `--x-rule-validation=core-only` | (CLI) |
| `--x-parmap` process parallelism | Default thread parallelism | (CLI) |
| Exit code 14 | Deprecated | (CLI) |
| Copying an old pinned version for platform use | A current release | https://docs.semgrep.dev/kb/semgrep-code/run-specific-version |
