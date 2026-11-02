# Sources

- Documentation: https://docs.semgrep.dev/
- Version covered: latest docs, Semgrep CLI 1.181.0 (release notes of the week of 2026-10-19)
- Fetched: 2026-11-02
- Raw pages: append `.md` to any page path. Index: https://docs.semgrep.dev/llms.txt
- Not covered: API reference, usage and billing, compliance, older release notes, AppSec Platform UI, Supply Chain details, Multimodal, Guardian, learning pages, cheat sheets, contributing

## Rule syntax

- https://docs.semgrep.dev/writing-rules/overview
- https://docs.semgrep.dev/writing-rules/rule-syntax
- https://docs.semgrep.dev/writing-rules/pattern-syntax
- https://docs.semgrep.dev/writing-rules/pattern-examples
- https://docs.semgrep.dev/writing-rules/generic-pattern-matching
- https://docs.semgrep.dev/writing-rules/metavariable-analysis
- https://docs.semgrep.dev/writing-rules/glossary
- https://docs.semgrep.dev/writing-rules/private-rules
- https://docs.semgrep.dev/writing-rules/experiments/pattern-syntax
- https://docs.semgrep.dev/kb/rules/changing-rule-severity-and-other-metadata
- https://docs.semgrep.dev/kb/rules/ellipsis-metavariables
- https://docs.semgrep.dev/kb/rules/exclude_rule_for_certain_filetypes
- https://docs.semgrep.dev/kb/rules/match-absence
- https://docs.semgrep.dev/kb/rules/match-comments
- https://docs.semgrep.dev/kb/rules/pattern-parse-error
- https://docs.semgrep.dev/kb/rules/rule-file-perf-principles
- https://docs.semgrep.dev/kb/rules/understand-severities
- https://docs.semgrep.dev/kb/rules/using-pattern-not-inside
- https://docs.semgrep.dev/kb/rules/using-semgrep-rule-schema-in-vscode

## Dataflow, autofix and testing

- https://docs.semgrep.dev/writing-rules/data-flow/data-flow-overview
- https://docs.semgrep.dev/writing-rules/data-flow/constant-propagation
- https://docs.semgrep.dev/writing-rules/data-flow/status
- https://docs.semgrep.dev/writing-rules/data-flow/taint-mode/overview
- https://docs.semgrep.dev/writing-rules/data-flow/taint-mode/advanced
- https://docs.semgrep.dev/writing-rules/rule-defined-fix
- https://docs.semgrep.dev/writing-rules/testing-rules
- https://docs.semgrep.dev/troubleshooting/rules
- https://docs.semgrep.dev/writing-rules/experiments/introduction
- https://docs.semgrep.dev/writing-rules/experiments/deprecated-experiments
- https://docs.semgrep.dev/writing-rules/experiments/metavariable-type
- https://docs.semgrep.dev/writing-rules/experiments/multiple-focus-metavariables
- https://docs.semgrep.dev/writing-rules/experiments/symbolic-propagation
- https://docs.semgrep.dev/writing-rules/experiments/display-propagated-metavariable
- https://docs.semgrep.dev/writing-rules/experiments/join-mode/overview
- https://docs.semgrep.dev/writing-rules/experiments/aliengrep
- https://docs.semgrep.dev/kb/semgrep-code/finding_all_taints
- https://docs.semgrep.dev/kb/semgrep-code/reduce-false-positives

## CLI and configuration

- https://docs.semgrep.dev/cli-reference
- https://docs.semgrep.dev/getting-started/cli
- https://docs.semgrep.dev/getting-started/quickstart-ce
- https://docs.semgrep.dev/running-rules
- https://docs.semgrep.dev/ignoring-files-folders-code
- https://docs.semgrep.dev/semgrepignore-v2-reference
- https://docs.semgrep.dev/customize-semgrep-ce
- https://docs.semgrep.dev/troubleshooting/semgrep
- https://docs.semgrep.dev/semgrep-pro-vs-oss
- https://docs.semgrep.dev/supported-languages
- https://docs.semgrep.dev/update
- https://docs.semgrep.dev/metrics
- https://docs.semgrep.dev/extensions/pre-commit
- https://docs.semgrep.dev/kb/integrations/customize-semgrep-precommit
- https://docs.semgrep.dev/semgrep-appsec-platform/json-and-sarif
- https://docs.semgrep.dev/kb/semgrep-code/collect-cli-logs
- https://docs.semgrep.dev/kb/semgrep-code/run-specific-version
- https://docs.semgrep.dev/kb/semgrep-code/scan-engine-kill
- https://docs.semgrep.dev/kb/semgrep-code/semgrep-scan-troubleshooting
- https://docs.semgrep.dev/kb/semgrep-code/semgrepignore-ignored
- https://docs.semgrep.dev/kb/semgrep-code/support-for-language-versions
- https://docs.semgrep.dev/kb/semgrep-code/unexpected-new-findings
- https://docs.semgrep.dev/kb/rules/run-all-available-rules
- https://docs.semgrep.dev/release-notes/2026-10-19

## CI

- https://docs.semgrep.dev/deployment/add-semgrep-to-ci
- https://docs.semgrep.dev/deployment/add-semgrep-to-other-ci-providers
- https://docs.semgrep.dev/deployment/customize-ci-jobs
- https://docs.semgrep.dev/deployment/oss-deployment
- https://docs.semgrep.dev/semgrep-ci/configuring-blocking-and-errors-in-ci
- https://docs.semgrep.dev/semgrep-ci/ci-environment-variables
- https://docs.semgrep.dev/semgrep-ci/findings-ci
- https://docs.semgrep.dev/semgrep-ci/sample-ci-configs
- https://docs.semgrep.dev/semgrep-ci/packages-in-semgrep-docker
- https://docs.semgrep.dev/kb/semgrep-ci/ci-vs-cli
- https://docs.semgrep.dev/kb/semgrep-ci/why-duplicate-findings
- https://docs.semgrep.dev/kb/semgrep-ci/scan-monorepo-in-parts
- https://docs.semgrep.dev/kb/semgrep-ci/trigger-diff-scans-env-var
- https://docs.semgrep.dev/kb/semgrep-ci/git-command-errors
- https://docs.semgrep.dev/kb/semgrep-ci/using-nonroot-docker-image-with-gha
- https://docs.semgrep.dev/kb/semgrep-ci/github-reusable-workflows-semgrep
- https://docs.semgrep.dev/kb/semgrep-ci/upload-ci-findings-to-github
- https://docs.semgrep.dev/deployment/tokens

## Languages and engines

- https://docs.semgrep.dev/languages/javascript
- https://docs.semgrep.dev/languages/python
- https://docs.semgrep.dev/languages/java
- https://docs.semgrep.dev/languages/go
- https://docs.semgrep.dev/languages/csharp
- https://docs.semgrep.dev/languages/kotlin
- https://docs.semgrep.dev/languages/ruby
- https://docs.semgrep.dev/languages/scala
- https://docs.semgrep.dev/languages/swift
- https://docs.semgrep.dev/references/language-maturity-levels
- https://docs.semgrep.dev/semgrep-ce-languages
- https://docs.semgrep.dev/semgrep-code/semgrep-pro-engine-intro
- https://docs.semgrep.dev/semgrep-code/semgrep-pro-engine-examples
- https://docs.semgrep.dev/semgrep-code/pro-rules
- https://docs.semgrep.dev/semgrep-secrets/conceptual-overview
- https://docs.semgrep.dev/semgrep-secrets/rules
- https://docs.semgrep.dev/semgrep-secrets/validators
- https://docs.semgrep.dev/semgrep-secrets/generic-secrets
- https://docs.semgrep.dev/kb/rules/pro-vs-community-secrets-vs-code-rules
