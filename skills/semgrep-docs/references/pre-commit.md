# Semgrep as a pre-commit hook

## Semgrep CE hook

```yaml
repos:
  - repo: https://github.com/semgrep/pre-commit
    rev: 'vX.Y.Z'           # a current release tag
    hooks:
      - id: semgrep
        args: ['--config', '<RULESET_URL>', '--error', '--skip-unknown-extensions']
```

Source: https://docs.semgrep.dev/extensions/pre-commit

- `--error` is what makes the hook fail on findings, since `semgrep scan` exits 0 otherwise. https://docs.semgrep.dev/customize-semgrep-ce
- The docs pages show different `rev` tags (v1.172.0, v1.126.0, v1.101.0) that may be outdated. Check the repository for a current release instead of copying one. https://docs.semgrep.dev/extensions/pre-commit https://docs.semgrep.dev/kb/integrations/customize-semgrep-precommit
- Useful extra args: `--disable-version-check`, `--quiet`, `--skip-unknown-extensions`. https://docs.semgrep.dev/kb/integrations/customize-semgrep-precommit
- Environment form instead of `--config`: `entry: sh -c 'env SEMGREP_RULES=<RULESET_URL> semgrep scan --code --no-suppress-errors --baseline-commit HEAD'`. https://docs.semgrep.dev/kb/integrations/customize-semgrep-precommit
- Pro rules in the plain hook: `args: ["--pro", "--disable-version-check", "--quiet", "--skip-unknown-extensions"]`, with a local login. https://docs.semgrep.dev/kb/integrations/customize-semgrep-precommit

## AppSec Platform hook

- The `semgrep-ci` hook needs `semgrep login`, uses the organization rules and engine, and does not send results to the platform. https://docs.semgrep.dev/extensions/pre-commit
- Keep `pass_filenames: false`. `semgrep ci` scans the folder and does not expect file names. https://docs.semgrep.dev/kb/integrations/customize-semgrep-precommit
- Scan only the current commit: `args: ["ci", "--dry-run", "--baseline-commit", "HEAD", "--secrets"]` (the docs example limits it to Secrets). `--dry-run` keeps local findings off the platform. https://docs.semgrep.dev/kb/integrations/customize-semgrep-precommit

## Output

- pre-commit hides output when a hook passes. Add `verbose: true` to see non-blocking findings. https://docs.semgrep.dev/kb/integrations/customize-semgrep-precommit
