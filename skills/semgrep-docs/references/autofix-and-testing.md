# Autofix, rule tests and validation

## Contents

- fix
- fix-regex
- Applying fixes
- Test annotations
- Test file layout
- Fix tests
- Validation
- Publishing private rules
- Deprecated experiments

Abbreviations used below: FIX = https://docs.semgrep.dev/writing-rules/rule-defined-fix , TEST = https://docs.semgrep.dev/writing-rules/testing-rules

## fix

- `fix:` is a replacement string with metavariable interpolation. Fixes are deterministic and written by the rule author, separate from the AI-generated fixes of the platform. (FIX)

```yaml
rules:
  - id: use-sys-exit
    languages: [python]
    message: Use `sys.exit` over the python shell `exit` built-in.
    pattern: exit($X)
    fix: sys.exit($X)
    severity: MEDIUM
```

- `fix: ""` deletes the matched code. (FIX)
- The experimental syntax adds `as: $NAME` to bind a whole match for use in `fix`. https://docs.semgrep.dev/writing-rules/experiments/pattern-syntax

## fix-regex

```yaml
fix-regex:
  regex: '(.*)\)'
  replacement: '\1, timeout=30)'
```

- `regex` and `replacement` are required. `count` is optional, and all occurrences are replaced by default (left to right, then top to bottom). It behaves like Python `re.sub`. A regex that matches nothing changes nothing. (FIX)

## Applying fixes

- `semgrep scan --autofix` writes the fixes. `--autofix --dryrun` previews them. https://docs.semgrep.dev/cli-reference
- The CLI warns that data loss can occur with `--autofix`. Run it on files under version control. https://docs.semgrep.dev/cli-reference
- `semgrep ci` ignores `--autofix` and `--dryrun`. https://docs.semgrep.dev/cli-reference
- On the AppSec Platform, fixes appear as suggestions in PR and MR comments once the rule-defined fix toggle is on (Settings > General > Code). (FIX)

## Test annotations

Put the annotation in a comment on the line right before the code it covers. (TEST)

| Annotation | Meaning |
| :-- | :-- |
| `ruleid: <id>` | The next line must match (guards against false negatives) |
| `ok: <id>` | The next line must not match (guards against false positives) |
| `todoruleid: <id>` | Known false negative, does not fail the run |
| `todook: <id>` | Known false positive, does not fail the run |

```python
# ruleid: use-sys-exit
exit(1)
# ok: use-sys-exit
sys.exit(1)
```

- `--test-ignore-todo` ignores the `todoruleid` annotations. https://docs.semgrep.dev/cli-reference
- Local rule ids get a prefix: the relative path from the cwd to the rule's directory, with dots as separators. The page also says to remove characters not allowed in a rule ID from that path. https://docs.semgrep.dev/running-rules

## Test file layout

- `semgrep --test <dir>` pairs each rule file with targets of the same base name and the rule's languages: `path/to/rule.yaml` tests `path/to/rule.py`, `path/to/rule.js` and so on. (TEST)
- A rule whose language is YAML uses a `rule.test.yaml` target. (TEST)
- Separate trees: `semgrep --test --config tests/rules/ tests/targets/`. Both trees need the same subdirectory structure. (TEST)
- Success prints "All tests passed". (TEST)
- `semgrep test` is a newer subcommand marked experimental, an improvement over `semgrep scan --test`. https://docs.semgrep.dev/cli-reference

## Fix tests

- The expected output lives in `rule.fixed.py` (`.fixed` before the extension). `--test` applies the fix to `rule.py` and diffs the result against it. Without such a file the output says "No tests for fixes found." (TEST)
- Bootstrap the expected file, then review the diff before keeping it (TEST):

```bash
cp rule.py rule.fixed.py
semgrep --config rule.yaml rule.fixed.py --autofix
```

- Success prints "All fix tests passed". (TEST)

## Validation

- `semgrep --validate --config <file>` checks the YAML and runs the `p/semgrep-rule-lints` rules (duplicate patterns, missing fields) without scanning. It is marked experimental. (TEST) https://docs.semgrep.dev/cli-reference
- `semgrep validate` is the experimental subcommand form. https://docs.semgrep.dev/cli-reference
- Exit code 4 means an invalid pattern, 5 invalid YAML, 7 at least 1 invalid rule. https://docs.semgrep.dev/cli-reference
- An older CLI can reject newer rules. Upgrade before debugging a config error. https://docs.semgrep.dev/kb/semgrep-code/semgrep-scan-troubleshooting

## Publishing private rules

- Private rules need a Team or Enterprise tier. Publish with `semgrep login` then `semgrep publish <dir>`. A directory is required. `--visibility=unlisted` or `org_private`. https://docs.semgrep.dev/writing-rules/private-rules
- Test files next to the rules are uploaded too. Publishing an existing `id` overwrites the rule. https://docs.semgrep.dev/writing-rules/private-rules
- To change the severity or metadata of a Registry rule, fork it on the platform (Code and Secrets rules only). The fork is independent: add it to a policy and disable the original. https://docs.semgrep.dev/kb/rules/changing-rule-severity-and-other-metadata

## Deprecated experiments

https://docs.semgrep.dev/writing-rules/experiments/deprecated-experiments

| Feature | Status | Replacement |
| :-- | :-- | :-- |
| Equivalences | Deprecated since 0.61.0 | None stated |
| Extract mode (`mode: extract`, `extract`, `dest-language`, `reduce`, `transform`) | Removed in 1.65.0, may return | None stated |
| Playground Turbo Mode | Removed on 2025-06-16 | None |

- Join mode (`mode: join`) is experimental, needs a login and does not run in the Playground or Editor. It only uses the first rule of a referenced file, reports only the last finding's location and is not taint aware. Prefer cross-file analysis. https://docs.semgrep.dev/writing-rules/experiments/join-mode/overview
