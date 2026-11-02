# Performance, memory and debugging

## Contents

- Scan limits and defaults
- Parallelism
- Out of memory and killed engines
- Slow scans
- Writing fast rules
- Isolating a problem
- Logs for a bug report

Abbreviations used below: CLI = https://docs.semgrep.dev/cli-reference , TS = https://docs.semgrep.dev/kb/semgrep-code/semgrep-scan-troubleshooting , KILL = https://docs.semgrep.dev/kb/semgrep-code/scan-engine-kill

## Scan limits and defaults

| Flag | Default | Note |
| :-- | :-- | :-- |
| `--timeout` | 5 s per rule and file | 0 means no limit. Usual range 3 s to 30 s |
| `--timeout-threshold` | 3 | Timeouts on 1 file before it is skipped. 0 means no limit |
| `--max-target-bytes` | 1000000 | Larger files are skipped |
| `--max-memory` | 0 (unlimited, MiB) | Exceeding it skips that rule and file |
| `--interfile-timeout` | 0 locally, 3 hours in CI | On expiry Semgrep falls back to the open source engine |

Sources: (CLI) (TS) https://docs.semgrep.dev/getting-started/cli

- In CI, `SEMGREP_TIMEOUT` sets `--timeout`. https://docs.semgrep.dev/deployment/customize-ci-jobs
- `--verbose` or `--debug` lists skipped files. https://docs.semgrep.dev/kb/rules/rule-file-perf-principles
- Semgrep caps file size and memory, not the number of files. Check the per-language file counts in the scan status. (KILL)

## Parallelism

- `-j/--jobs` defaults to about 85% of logical cores. Stay 10 to 15% under the core count (10 or 11 jobs on 12 cores) and do not oversubscribe. (CLI)
- With `--pro`, the default is `-j 1`. https://docs.semgrep.dev/getting-started/cli
- Semgrep does not run in parallel on Windows. https://docs.semgrep.dev/customize-semgrep-ce

## Out of memory and killed engines

- Exit -9 or -11 is a crash, usually memory exhaustion. https://docs.semgrep.dev/troubleshooting/semgrep
- Remedies: more memory, `--max-memory` near 70% of what is available, `-j 1`, a larger stack with `ulimit -s`, or scanning in parts. (KILL) https://docs.semgrep.dev/troubleshooting/semgrep
- The docs quote different `--max-memory` defaults for cross-file scans in CI (5000 MB in an error message; in the CLI reference, for CI scans that use the Pro Engine, 90% of container memory or 8 GiB). Set it explicitly. (KILL) (CLI)
- Run the products one after the other: `semgrep ci --code`, then `--supply-chain`, then `--secrets`. (KILL)
- Size RAM by scanning with `-j 1` while watching swap, then add about 10% headroom. `-j 2` needs a bit under twice as much. Avoid swap. (KILL)
- Interfile (cross-file) analysis needs at least 4 GB per core. 8 GB per core or more is recommended. https://docs.semgrep.dev/writing-rules/data-flow/taint-mode/overview

## Slow scans

- First cut the file set with `.semgrepignore`, then lower `--max-target-bytes`. (TS)
- `--exclude-minified-files` is off by default. Minified JavaScript is a typical cause of slowness. (CLI) https://docs.semgrep.dev/kb/rules/rule-file-perf-principles
- `semgrep scan --time` reports the slowest rules and files (per rule and target in JSON). It is meant for internal use and may change. (TS) (CLI)
- `--disable-version-check` may reduce exit time after results are returned. (CLI)

## Writing fast rules

https://docs.semgrep.dev/kb/rules/rule-file-perf-principles

- Sub-patterns with many matches, such as `<... $X ...>`, slow the whole rule.
- One slow rule among fast ones can cause a significant slowdown. Interfile rules (`interfile: true`) scale better than linearly, so doubling them raises runtime by less than double.
- Cross-file scan time grows with groups of mutually dependent files, exponentially in the worst case.

## Isolating a problem

- Upgrade first. An older Semgrep often fails on newer rules. (TS)
- Drop the `--pro*` flags or add `--oss-only` to rule out cross-file analysis. (TS)
- Narrow with `--exclude-rule`, `--exclude` and `--include`. (TS)
- Debug a rule by running each sub-pattern alone. https://docs.semgrep.dev/troubleshooting/rules

## Logs for a bug report

- `-v` adds the rule list and skipped files. `--debug` is very large. Pair them with `--max-log-list-entries=0`, otherwise lists over 100 entries are cut. (TS)
- `semgrep ci &> semgrep.log` captures stdout and stderr. Avoid `-q` while troubleshooting. https://docs.semgrep.dev/kb/semgrep-code/collect-cli-logs
- `--dump-ast` prints the parse tree. `--matching-explanations` adds matching details to JSON. (CLI)
- `--time` and `--trace` are meant for internal use and may change or be removed. Any `--x-*` flag is experimental and may be removed. `--profile` records Pyro Caml profiles (default target localhost:4040). (CLI)
- For memory issues report total size, file count, peak memory and machine specs. Open public issues at https://github.com/semgrep/semgrep/issues, and use Support for cross-file or sensitive cases. (TS)
