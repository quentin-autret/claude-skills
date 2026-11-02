# Rule file syntax

## Contents

- Required keys and severity
- How `patterns` evaluates
- Negative operators
- focus-metavariable
- Metavariable conditions
- Rule options and their defaults
- paths, min-version, max-version
- Editor schema
- Experimental syntax
- Old patterns

Abbreviations used below: RS = https://docs.semgrep.dev/writing-rules/rule-syntax

## Required keys and severity

- Each item under `rules:` needs `id`, `message`, `severity`, `languages` and one of `pattern`, `patterns`, `pattern-either`, `pattern-regex`. Optional: `options`, `fix`, `metadata`, `min-version`, `max-version`, `paths`. (RS)
- `severity` is `LOW`, `MEDIUM`, `HIGH` or `CRITICAL`. https://docs.semgrep.dev/kb/rules/understand-severities
- Confidence (in `metadata`) describes the rule's false-positive risk, not the vulnerability. Taint rules are typically HIGH confidence. https://docs.semgrep.dev/kb/rules/understand-severities
- Supply Chain takes severity from CVE data. Authors set severity only for Code and Secrets rules. (RS)
- `metadata` never changes matching. It shows up in `--json` output. (RS)
- Apex, Elixir (`ex`/`elixir`) and Gosu exist only in the Pro engine. (RS)

## How `patterns` evaluates

`patterns` is a logical AND whose child order does not matter. Semgrep evaluates it in 4 steps (RS):

1. Positive patterns (`pattern`, `pattern-inside`, `pattern-regex`, `pattern-either`) are intersected.
2. Negative patterns (`pattern-not`, `pattern-not-inside`, `pattern-not-regex`) filter the result.
3. Conditionals (`metavariable-regex`, `metavariable-pattern`, `metavariable-comparison`) run.
4. `focus-metavariable` narrows the range.

- Conditionals only see metavariables bound by positive patterns. A metavariable bound only inside `pattern-not` is not available to them. (RS)
- Same-named metavariables must bind identical code across sub-patterns (unification). Values bound earlier in a parent AND also constrain a nested `pattern-either`. Branches of `pattern-either` need not agree with each other. (RS)
- `pattern-inside` must sit under `patterns` or `pattern-either`. The `metavariable-*` operators and the negative operators are only valid directly under `patterns`. (RS)

## Negative operators

- `pattern-not` only removes a match of the same extent. To accept code only when other code surrounds it, use `pattern-not-inside`. https://docs.semgrep.dev/kb/rules/using-pattern-not-inside

```yaml
patterns:
  - pattern: make_transaction($T)
  - pattern-not-inside: |
      verify_transaction($T)
      ...
```

- Missing cleanup (`open` without `close`) is the typical `pattern-not-inside` case, tied by a shared metavariable such as `$F`. (RS)
- `pattern-not-regex` drops any finding that overlaps the regex at all. (RS)
- `pattern-regex` is PCRE2 in multiline mode (`^` and `$` match at line breaks). Not every Perl Unicode property exists (`\p{InMusicalSymbols}` does not). Use single-quoted YAML strings for regexes with backslashes. Named groups `(?P<NAME>...)` bind metavariables (with `languages: [regex]`). (RS)
- YAML pitfall: `pattern-regex: |` appends a trailing newline. Use `|-`. https://docs.semgrep.dev/troubleshooting/rules

## focus-metavariable

- It narrows the reported range to what the metavariable matched. It matches nothing by itself. (RS)
- `pattern: $ARG` is not a substitute. It matches every expression use of the value, not the declaration. (RS)
- A narrow range keeps a triaged finding stable on the AppSec Platform when unrelated lines of a large matched block change. (RS)
- Several `focus-metavariable` entries intersect. A YAML list under one `focus-metavariable` key gives a union, which is experimental. https://docs.semgrep.dev/writing-rules/experiments/multiple-focus-metavariables

## Metavariable conditions

- `metavariable-regex` (`metavariable`, `regex`) is left anchored. Add `.*` to allow a prefix and `$` to anchor the end. It runs on the whole matched text, quotes included (`"foo"` is 5 characters). (RS) https://docs.semgrep.dev/troubleshooting/rules
- `metavariable-pattern` takes `metavariable` plus exactly one of `pattern`, `patterns`, `pattern-either`, `pattern-regex`, and nests. The metavariable must bind an expression, a statement or a statement list. Bound to an argument list, a type or a pattern, it always evaluates false. Add `language:` to parse the content of a string as code (`generic` for literal-string content). (RS)
- `metavariable-pattern` fails with "Pattern parse error in rule" on reserved words (for example `delete` in JavaScript). Use `metavariable-regex` with `regex: ^(post|put|delete|patch)$` instead. https://docs.semgrep.dev/kb/rules/pattern-parse-error
- `metavariable-comparison` (`metavariable`, `comparison`) accepts literals, `not or and`, arithmetic, comparisons, `in` / `not in`, `int()`, `str()`, `today()`, `strptime("yyyy-mm-dd")`, `re.match()` (no flags), `lower()`, `upper()`. With constant propagation on (default), a constant variable is compared by its value. `str($X)` compares the code text. (RS)
- `metavariable-name` constrains the module of an identifier (`module:` or `modules:`). It needs an account and the Pro engine, and does nothing useful with `--oss-only`. (RS)
- `metavariable-analysis` with `analyzer: redos` flags catastrophic-backtracking regexes. `analyzer: entropy` is true for high-randomness strings. https://docs.semgrep.dev/writing-rules/metavariable-analysis
- `metavariable-type` (`metavariable`, `type`, optional `language`) compares a metavariable against a type given as a string, so it provides several advantages over typed metavariables: no per-language typed syntax to memorize, and type expressions can be extracted from the pattern and reused in other metavariable filters. It is documented under experiments. https://docs.semgrep.dev/writing-rules/experiments/metavariable-type

```yaml
patterns:
  - pattern: $X == $Y
  - metavariable-type:
      metavariable: $Y
      type: String
```

## Rule options and their defaults

Set under `options:` (RS).

| Option | Default | Effect |
| :-- | :-- | :-- |
| `ac_matching` | true | Associative-commutative matching of Boolean and bitwise operators |
| `attr_expr` | true | |
| `commutative_boolop` | false | |
| `constant_propagation` | true | Literal patterns match constant variables |
| `decorators_order_matters` | false | |
| `generic_comment_style` | none | `c`, `cpp`, `shell` to skip comments in generic mode |
| `generic_ellipsis_max_span` | 10 | Lines one `...` may span in generic mode |
| `implicit_return` | true | Expression-based languages (Ruby, Julia) |
| `interfile` | false | Required for rules that use cross-function, cross-file analysis (Pro engine) |
| `symmetric_eq` | false | |
| `taint_assume_safe_functions` | false | Experimental |
| `taint_assume_safe_indexes` | false | `a[i]` is tainted when `i` is |
| `vardef_assign` | true | `$X = $E` also matches `var x = 1;` |
| `xml_attrs_implicit_ellipsis` | true | `<div />` matches `<div foo="1">` |

- `interfile` is required (set to `true`) for rules that use cross-function, cross-file analysis. (RS)
- Set `taint_assume_safe_indexes: true` for high-signal rules, leave it false for audit rules. (RS)
- Options missing from this table are experimental and may vanish. Full list: https://github.com/semgrep/semgrep/blob/develop/interfaces/Rule_options.atd (RS)

## paths, min-version, max-version

- `paths.include` / `paths.exclude` take globs relative to the project root, not the working directory. Exclude wins. A directory pattern covers its files. When you use `include`, add the test files too. (RS)
- `/src/tests` and `/*.c` are anchored to the root. `*.c` (no slash) matches at any depth. `src/*.c` (slash in the middle) is currently floating for backward compatibility and may warn: write `/src/*.c` or `**/src/*.c`. (RS)
- To exclude one file type for one Registry rule only: copy the rule, rename its `id`, add `paths.exclude`, then run `semgrep scan --config=auto --config=my_custom.yml --exclude-rule <original id>`. https://docs.semgrep.dev/kb/rules/exclude_rule_for_certain_filetypes
- `min-version` / `max-version` (Semgrep 1.38.0+) skip the rule without error on other versions. Ship 2 variants of a rule with `max-version: 1.72.999` and `min-version: 1.73.0`. (RS)

## Editor schema

- Put `# yaml-language-server: $schema=https://json.schemastore.org/semgrep.json` at the top of a rule file (Red Hat YAML extension in VS Code) for completion and validation. https://docs.semgrep.dev/kb/rules/using-semgrep-rule-schema-in-vscode
- Full schema: https://github.com/semgrep/semgrep-interfaces/blob/main/rule_schema_v1.yaml (RS)

## Experimental syntax

A newer syntax exists and is "experimental and subject to change". It cannot be mixed with the classic syntax in one rule. https://docs.semgrep.dev/writing-rules/experiments/pattern-syntax

- `pattern-either` becomes `any`, `patterns` becomes `all`, `pattern-inside` becomes `inside`, `pattern-not` becomes `not`, `pattern-regex` becomes `regex`. The `pattern:` key can be omitted.
- Search rules go under `match:`. Metavariable operators go in a `where:` list. `as: $NAME` binds any match to a metavariable.
- Taint rules go under `taint:` with `sources`, `sinks`, `propagators` (`from`/`to`), `sanitizers`, without `mode: taint`.
- Quote single-line patterns: `- def foo(): ...` unquoted breaks YAML on the `:`.

## Old patterns

| Old | Current | Source |
| :-- | :-- | :-- |
| `severity: ERROR` / `WARNING` / `INFO` | `HIGH` / `MEDIUM` / `LOW` (plus `CRITICAL`). Old values still accepted | (RS) |
| `"=~/regex/i"` inside a pattern string | `metavariable-regex` | https://docs.semgrep.dev/writing-rules/pattern-syntax |
| `metavariable-comparison` with `base: int` or `strip: true` | `int($X)` inside `comparison` | (RS) |
