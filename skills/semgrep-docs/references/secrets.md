# Semgrep Secrets rules and validators

## Contents

- How Secrets works
- Rule skeleton
- Metadata
- Validator syntax
- Running local Secrets rules
- Secrets versus Code rules
- Generic secrets

Abbreviations used below: SR = https://docs.semgrep.dev/semgrep-secrets/rules , VAL = https://docs.semgrep.dev/semgrep-secrets/validators

## How Secrets works

- Secrets is an AppSec Platform product. Its pipeline is regex search, semantic analysis (dataflow and constant propagation, so renamed variables are followed), validation, then entropy analysis. https://docs.semgrep.dev/semgrep-secrets/conceptual-overview
- Validation runs in your environment. Tokens are not sent to Semgrep servers. https://docs.semgrep.dev/semgrep-secrets/conceptual-overview
- Outcomes: Confirmed valid, Confirmed invalid, Validation error (network failure, timeout or unexpected status, review by hand), No validator (review by hand). https://docs.semgrep.dev/semgrep-secrets/conceptual-overview
- Secrets scans ignore `.semgrepignore`. Use `--exclude` or the platform's Secrets path ignores. https://docs.semgrep.dev/ignoring-files-folders-code

## Rule skeleton

```yaml
rules:
  - id: github_example
    message: Validates a GitHub token against github.com
    severity: MEDIUM
    languages: [regex]
    metadata:
      secret_type: GitHub
      technology: [secrets]
    validators:
      - http:
          request:
            headers:
              Authorization: Bearer $REGEX
              Host: api.github.com
              User-Agent: Semgrep
            method: GET
            url: https://api.github.com/user
          response:
            - match:
                - status-code: 200
              result:
                validity: valid
            - match:
                - status-code: 401
              result:
                validity: invalid
    patterns:
      - patterns:
          - pattern-regex: (?<REGEX>\b((ghp|gho|ghu|ghs|ghr|github_pat)_[a-zA-Z0-9_]{36,255})\b)
          - focus-metavariable: $REGEX
          - metavariable-analysis:
              analyzer: entropy
              metavariable: $REGEX
```

Source: (SR)

- The named group `(?<REGEX>...)` becomes `$REGEX`, reused in the request. For a language other than `regex`, match a string `"$R"` and apply the same `pattern-regex` through `metavariable-pattern`. (SR)
- The prose of the docs writes `focus_metavariable` and `metavariable_analysis`. The YAML keys use hyphens. (SR)
- Write it as a YAML file or in the Editor (Rules > Editor > + > Secrets > HTTP validators). Custom validator rules stay private to the organization. (SR)

## Metadata

- `secret_type` (a descriptive service name) and `technology: [secrets]` (marks the rule as a Secrets rule). The docs do not call them mandatory. The validators samples also set `product: secrets`. (SR) (VAL)

## Validator syntax

The syntax is experimental. Only HTTP and HTTPS validation exists. (VAL)

- `request`: `method` (GET, POST, PUT, DELETE, OPTIONS, PATCH), `url` and `headers` are required. `body` is optional for POST, PUT, PATCH. `Host` overrides the host. (VAL)
- `response`: a list of `match` plus `result`. `match` holds at least 1 of `status-code`, `content`, `headers` (exact name and value). (VAL)
- `result.validity` is `valid` or `invalid`. Optional `message`, `metadata` and `severity` override the finding per state, for example a lower severity for an invalid secret. (VAL)
- When valid and invalid answers share a status code, match the body. `content` needs `language: regex` or `generic` (VAL):

```yaml
response:
  - match:
      - status-code: 200
      - content:
          language: regex
          pattern-regex: (\"ok\":true)
    result:
      validity: valid
```

- Base64 in `url`, `body` or a header: `Authorization: Basic __semgrep_internal_encode_64($REGEX:)`. Experimental. (VAL)
- The docs samples write status codes both as integers and as quoted strings. The syntax page uses integers. (VAL) (SR)

## Running local Secrets rules

- A local rule file with validators needs `semgrep ci --allow-untrusted-validators`. Otherwise publish it as a private rule. (SR)
- Only allow validators from origins you trust. `--no-secrets-validation` skips validation. `--secrets-timeout` defaults to 30 s. https://docs.semgrep.dev/cli-reference
- `--historical-secrets` also scans Git history. https://docs.semgrep.dev/cli-reference

## Secrets versus Code rules

- Secret-detecting Code rules have no validator, are less accurate and should not block a PR or MR. Pro secret rules are moving to Secrets. https://docs.semgrep.dev/kb/rules/pro-vs-community-secrets-vs-code-rules
- When both exist, the Secrets rule supersedes the Code rule, and the Code finding shows "Secrets version available". https://docs.semgrep.dev/kb/rules/pro-vs-community-secrets-vs-code-rules

## Generic secrets

https://docs.semgrep.dev/semgrep-secrets/generic-secrets

- AI-assisted detection that needs Secrets access, Semgrep Multimodal and CLI 1.86.0+ in CI.
- It uploads candidate line locations to the platform for analysis, unlike regular validators.
- It does not work in local scans started by `semgrep ci`, since Multimodal needs code access.
- Enable it at Settings > General > Secrets > Generic secrets. The CLI prints "AI augmented rules are active for secrets detection."
