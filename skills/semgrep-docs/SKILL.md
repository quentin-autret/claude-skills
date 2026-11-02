---
name: semgrep-docs
description: Teaches current Semgrep practices drawn from the official documentation. Use when writing Semgrep rules, patterns and metavariables, taint mode rules (sources, sinks, sanitizers, propagators), autofix with fix or fix-regex, rule tests with ruleid and ok annotations, or when running the semgrep CLI (semgrep scan, semgrep ci, --config, output formats, exit codes), configuring .semgrepignore and nosemgrep, adding Semgrep to CI or pre-commit, tuning timeouts and memory, choosing between Semgrep Community Edition and the Pro engine, or writing Semgrep Secrets validators. Triggers on semgrep, pattern-inside, pattern-not, metavariable-regex, focus-metavariable, mode taint, interfile, semgrep --test, .semgrepignore, nosemgrep, SEMGREP_APP_TOKEN, diff-aware scan.
metadata:
  version: "latest docs, Semgrep CLI 1.181.0"
  source: https://docs.semgrep.dev/
  fetched: "2026-11-02"
---

# Semgrep

This skill covers Semgrep as documented on https://docs.semgrep.dev/. It holds surprising defaults, pitfalls, deprecated syntax and recommended patterns. Each claim ends with its source page.

## Semgrep CE versus the Pro engine

| | Semgrep Community Edition (CE) | Semgrep Code with the Pro engine |
| :-- | :-- | :-- |
| Command | `semgrep scan --config ...` | `semgrep ci` after `semgrep login`, or `semgrep scan --pro` |
| Rules | Your rules and Registry rules | Organization policies, Pro rules, private rules |
| Taint | Inside 1 function | Across functions (`--pro-intrafile`) and files (`--pro`) |
| Constant propagation | Inside 1 file | Across files |
| Pro-only syntax | | `metavariable-name`, taint propagators, `options.interfile: true` |
| Languages | Community supported, best effort | Adds Apex and Elixir, GA levels |

Sources: https://docs.semgrep.dev/semgrep-pro-vs-oss https://docs.semgrep.dev/semgrep-code/semgrep-pro-engine-intro https://docs.semgrep.dev/writing-rules/rule-syntax

- Many framework-specific Pro rules fail to return findings on CE. https://docs.semgrep.dev/languages/javascript
- Semgrep Code and Supply Chain are free for up to 10 contributors. https://docs.semgrep.dev/semgrep-pro-vs-oss

## Quick start

```yaml
# rules/no-exit.yaml
# yaml-language-server: $schema=https://json.schemastore.org/semgrep.json
rules:
  - id: use-sys-exit
    languages: [python]
    severity: MEDIUM
    message: Use sys.exit($X) instead of the shell exit built-in
    pattern: exit($X)
    fix: sys.exit($X)
```

```python
# rules/no-exit.py (test target next to the rule)
# ruleid: use-sys-exit
exit(1)
# ok: use-sys-exit
sys.exit(1)
```

```bash
pipx install semgrep                     # or: uv tool install semgrep
semgrep --validate --config rules/
semgrep --test rules/
semgrep scan --config rules/ --error .   # exit 1 on findings
```

Sources: https://docs.semgrep.dev/writing-rules/rule-defined-fix https://docs.semgrep.dev/writing-rules/testing-rules https://docs.semgrep.dev/getting-started/cli https://docs.semgrep.dev/kb/rules/using-semgrep-rule-schema-in-vscode

## Quick rules

### Rules and patterns

- `severity` is `LOW`, `MEDIUM`, `HIGH` or `CRITICAL`. https://docs.semgrep.dev/writing-rules/rule-syntax
- A pattern must be a complete expression or statement: `if $X < 5: ...`, not `if $X < 5`. https://docs.semgrep.dev/troubleshooting/rules
- Metavariables are uppercase: `$X`, `$NAME`. `$x` is not a metavariable. https://docs.semgrep.dev/writing-rules/pattern-syntax
- `pattern-not` only removes a match of the same extent. Use `pattern-not-inside` for surrounding code. https://docs.semgrep.dev/kb/rules/using-pattern-not-inside
- Conditionals (`metavariable-regex`, `metavariable-pattern`, `metavariable-comparison`) only see metavariables bound by positive patterns. https://docs.semgrep.dev/writing-rules/rule-syntax
- `metavariable-regex` is left anchored and includes the quotes of a string. https://docs.semgrep.dev/writing-rules/rule-syntax https://docs.semgrep.dev/troubleshooting/rules
- `pattern-regex: |` adds a trailing newline. Use `|-`. https://docs.semgrep.dev/troubleshooting/rules
- Imported names need the fully qualified path in patterns. https://docs.semgrep.dev/troubleshooting/rules
- In `generic` mode, `...` spans at most 10 lines and `$X` captures 1 word. https://docs.semgrep.dev/writing-rules/generic-pattern-matching

### Taint

- Sources and sanitizers default to `exact: false`, sinks to `exact: true`. https://docs.semgrep.dev/writing-rules/data-flow/taint-mode/overview
- Same-named metavariables in sources and sinks are independent. https://docs.semgrep.dev/writing-rules/data-flow/taint-mode/overview
- Target 1 argument with `focus-metavariable`, never with `pattern: $X`. https://docs.semgrep.dev/writing-rules/data-flow/taint-mode/advanced
- A cross-function, cross-file rule needs `options.interfile: true` (it is required, default `false`). https://docs.semgrep.dev/writing-rules/rule-syntax
- Semgrep reports only the shortest trace per source and sink pair. https://docs.semgrep.dev/kb/semgrep-code/finding_all_taints

### CLI and ignores

- `semgrep scan` exits 0 with findings. Add `--error`. https://docs.semgrep.dev/customize-semgrep-ce
- `semgrep ci` exits 0 on internal errors. Add `--no-suppress-errors`. https://docs.semgrep.dev/semgrep-ci/configuring-blocking-and-errors-in-ci
- `semgrep ci` rejects `--config` and ignores `--autofix`. https://docs.semgrep.dev/cli-reference
- Repeat `--config` per rule source. https://docs.semgrep.dev/cli-reference
- Local rule ids are prefixed by the relative path from the cwd to the rule's directory, with dots as separators (a rule in `configs/` becomes `configs.rule-id`). Reference rule ids with that namespace in `nosemgrep: configs.rule-id`. https://docs.semgrep.dev/running-rules https://docs.semgrep.dev/ignoring-files-folders-code
- `test/`, `tests/` and `vendor/` are ignored by default. A custom `.semgrepignore` replaces the defaults. https://docs.semgrep.dev/ignoring-files-folders-code
- Files over 1 MB are skipped. https://docs.semgrep.dev/ignoring-files-folders-code
- `--config auto` and Registry rules turn metrics on. Use `--metrics=off` when needed. https://docs.semgrep.dev/metrics

### CI

- Never set `SEMGREP_APP_TOKEN` and `SEMGREP_RULES` in the same job. https://docs.semgrep.dev/semgrep-ci/ci-environment-variables
- Diff-aware scans never run cross-file analysis. https://docs.semgrep.dev/deployment/customize-ci-jobs
- Full scans belong on the default branch only, or findings duplicate per branch. https://docs.semgrep.dev/kb/semgrep-ci/why-duplicate-findings
- Upgrade the CLI: releases before the week of 2026-10-19 trust a compromised certificate authority. https://docs.semgrep.dev/release-notes/2026-10-19

## Old patterns

| Old | Current | Source |
| :-- | :-- | :-- |
| `severity: ERROR` / `WARNING` / `INFO` | `HIGH` / `MEDIUM` / `LOW`, plus `CRITICAL` | https://docs.semgrep.dev/writing-rules/rule-syntax |
| `"=~/regex/"` in a pattern | `metavariable-regex` | https://docs.semgrep.dev/writing-rules/pattern-syntax |
| `metavariable-comparison` with `base` or `strip` | `int($X)` in `comparison` | https://docs.semgrep.dev/writing-rules/rule-syntax |
| Taint source via `pattern-inside: make_tainted($X) ...` | `by-side-effect: true` with `focus-metavariable` | https://docs.semgrep.dev/writing-rules/data-flow/taint-mode/advanced |
| `mode: extract` | Removed in 1.65.0 | https://docs.semgrep.dev/writing-rules/experiments/deprecated-experiments |
| Join mode (experimental, not actively maintained) | `options.interfile: true` | https://docs.semgrep.dev/semgrep-code/semgrep-pro-engine-intro |
| `# nosem` | `# nosemgrep` | https://docs.semgrep.dev/ignoring-files-folders-code |
| `--semgrepignore-v2` | Nothing, v2 is the only behavior since 1.117.0 | https://docs.semgrep.dev/cli-reference |
| `SEMGREP_BASELINE_REF` | `SEMGREP_BASELINE_COMMIT` or `--baseline-commit` | https://docs.semgrep.dev/semgrep-ci/ci-environment-variables |

## Reference files

| File | Read it for |
| :-- | :-- |
| [references/rule-syntax.md](references/rule-syntax.md) | Rule keys, severity, `patterns` evaluation order, negative operators, `focus-metavariable`, metavariable conditions, rule options and defaults, `paths`, version gates, experimental syntax |
| [references/pattern-syntax.md](references/pattern-syntax.md) | Valid patterns, ellipsis, metavariable forms, typed metavariables, deep expressions, equivalences, generic mode, matching absence and comments, Aliengrep |
| [references/taint-and-dataflow.md](references/taint-and-dataflow.md) | Dataflow limits, constant and symbolic propagation, taint rule shape, `exact`, side effects, propagators, false-positive options, labels, Pro taint features |
| [references/autofix-and-testing.md](references/autofix-and-testing.md) | `fix`, `fix-regex`, `--autofix`, test annotations and layout, fix tests, `--validate`, publishing private rules, deprecated experiments |
| [references/cli.md](references/cli.md) | Install and pinning, `scan` versus `ci`, `--config`, output formats, exit codes, risky flags, metrics |
| [references/ignoring.md](references/ignoring.md) | Default ignores, `.semgrepignore` v2, precedence, Git interaction, `nosemgrep`, platform path ignores |
| [references/performance-and-debugging.md](references/performance-and-debugging.md) | Timeouts, size and memory limits, jobs, killed engines, slow scans, fast rules, logs for bug reports |
| [references/ci.md](references/ci.md) | CE and platform setups, Docker image, GitHub Actions, GitLab, other providers, full and diff-aware scans, blocking, CI variables, tokens, monorepos |
| [references/pre-commit.md](references/pre-commit.md) | pre-commit hooks for CE and the platform |
| [references/pro-engine-and-languages.md](references/pro-engine-and-languages.md) | What the Pro engine adds, enabling cross-file analysis and its pitfalls, Pro rules, language support and maturity, rulesets per language, Supply Chain notes |
| [references/secrets.md](references/secrets.md) | Semgrep Secrets rules, required metadata, HTTP validators, local runs, Secrets versus Code rules, generic secrets |

