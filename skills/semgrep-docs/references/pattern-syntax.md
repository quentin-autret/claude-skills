# Pattern syntax and generic mode

## Contents

- Valid patterns
- Ellipsis
- Metavariables
- Typed metavariables
- Deep expressions and equivalences
- Generic mode
- Matching absence and comments
- Aliengrep
- Debugging a pattern that does not match

Abbreviations used below: PS = https://docs.semgrep.dev/writing-rules/pattern-syntax , GEN = https://docs.semgrep.dev/writing-rules/generic-pattern-matching

## Valid patterns

- A pattern must be a complete expression or statement. `1+` is invalid, write `1 + $X`. `if $X < 5` is invalid in Python, write `if $X < 5: ...`. (PS) https://docs.semgrep.dev/troubleshooting/rules
- Header-only patterns are allowed and useful in `pattern-inside`: `if ($E)`, `try { ... }`, `int foo(...)`, `class $A`. (PS)
- `foo` matches `x += foo()`, `return bar + foo` and `foo(1,2)`, but not `import foo`. (PS)
- A statement pattern is also searched as an expression, so `foo()` matches `a = foo();`. (PS)
- Imported names need the full path: after `import google.metrics`, write `google.metrics.send(...)`, not `metrics.send(...)`. https://docs.semgrep.dev/troubleshooting/rules
- Check that `languages` matches the target files when a pattern does not parse. https://docs.semgrep.dev/troubleshooting/rules

## Ellipsis

- `...` matches zero or more arguments, statements, parameters, fields or characters. It can also match exactly 1 item (`if (...)`, `foo = ...;`). Force exactly 1 item with a metavariable: `bar($X)`. (PS)
- Useful shapes (PS):

```text
requests.get(..., verify=False, ...)
$FUNC(..., $KEY=$VALUE, ...)
$O.foo(). ... .bar()
function ...($X) { ... }        any function, anonymous included
crypto.set_secret_key("...")     any string literal
user_dict = {..., $KEY: $VALUE, ...}
```

- The ellipsis does not leave a block. `foo(); ...; bar()` misses code where `foo()` sits inside an `if` and `bar()` after it. (PS)

## Metavariables

- Names are `$` then uppercase letters, digits or `_`, not starting with a digit: `$X`, `$USERS_2`, `$_VAR_2`. `$x`, `$1stvar` and `$VAR-WITH-DASHES` are invalid. (PS) https://docs.semgrep.dev/troubleshooting/rules
- `$_` is anonymous: it binds nothing and does not force equality. `$_1` is an ordinary metavariable. (PS)
- `$...ARGS` matches a sequence. Write it with the ellipsis in `message`: `Call to foo($...ARGS)`. (PS)
- `"$X"` binds the content of a string, `/$X/` a regex literal, `:$X` an atom. Their content may not be valid code, so a nested `metavariable-pattern` needs `language: generic`. (PS)
- Metavariables interpolate in `message`: `Setting a password on $MODEL`. (PS)
- In taint rules, same-named metavariables in sources and sinks are independent unless `options.taint_unify_mvars: true`. (PS)

## Typed metavariables

Single-file type inference only (PS):

```text
(java.util.logging.Logger $LOGGER).log(...)     Java
$X == (char *$Y)                                C
($READER : *zip.Reader).Open($INPUT)            Go
($X: DomSanitizer).sanitize(...)                TypeScript
```

- Reliable only for local variables and arguments. Class fields are not typed. Understanding is shallow (`int[] A` does not make `A[0]` an int). Go `var a, b = 1, 2` is not typed. (PS)
- The `metavariable-type` operator avoids per-language typed-metavariable syntax and lets other filters reuse the type. https://docs.semgrep.dev/writing-rules/experiments/metavariable-type
- With the Pro engine, typed metavariables also use cross-file inheritance. https://docs.semgrep.dev/semgrep-code/semgrep-pro-engine-examples

## Deep expressions and equivalences

- `<... P ...>` matches `P` nested at any depth: `if <... $USER.is_admin() ...>:`, `sql.query(<... $X ...>)`. Sub-patterns that produce many matches slow a rule. (PS) https://docs.semgrep.dev/kb/rules/rule-file-perf-principles
- Matched automatically: import aliases (`subprocess.Popen(...)` matches an aliased import), fully qualified names, constants (`set_password("password")` matches `set_password(HARDCODED_PASSWORD)`). (PS)
- Associative-commutative matching: `... && B && C` matches `(A && B) && C`, and `A | B | C` matches any permutation. Metavariables expand only while the possibilities stay few. Disable with `options.ac_matching: false`. (PS)
- Matching through intermediate assignments needs `options.symbolic_propagation: true` (experimental). https://docs.semgrep.dev/writing-rules/experiments/symbolic-propagation

## Generic mode

`languages: [generic]` handles unsupported languages, config files and mixed files. It can be evaded by unusual spellings such as XML entities. (GEN)

- The document is a sequence of ASCII words, ASCII punctuation and other bytes. Indentation drives nesting. `()`, `[]` and `{}` nest only inside 1 line. The target must be at least as indented as the pattern. (GEN)
- Pitfall: a pattern indented less than its target over-matches. `[system]` then `...` then `name = ...` also matches `name` under `[user]`. Indent the pattern like the target. (GEN)
- `$X` captures 1 word, `[A-Za-z0-9_]+`. So `p@$$w0rd` captures `p` and `ABC-DEF` is 2 tokens. Use `$...X` for a sequence, for example an HTML id with dashes. (GEN) https://docs.semgrep.dev/kb/rules/ellipsis-metavariables
- `...` reaches at most 10 lines past the last match (`generic_ellipsis_max_span`). `... ...` reaches 20 and `... ... ...` reaches 30. (GEN)
- Keep a match on 1 line with `generic_ellipsis_max_span: 0`, for example `password = $...PASSWORD`. (GEN)
- Comments are not recognized by default. Set `options.generic_comment_style` to `c`, `cpp` or `shell`. (GEN)
- `"=~/.../"` is not supported in generic mode. (GEN)
- Embedded code: capture it in generic mode, then parse it with `metavariable-pattern`. https://docs.semgrep.dev/writing-rules/rule-syntax

```yaml
languages: [generic]
patterns:
  - pattern: <script ...>$...JS</script>
  - metavariable-pattern:
      metavariable: $...JS
      language: javascript
      pattern: eval(...)
```

## Matching absence and comments

- Semgrep has no clear way to match the absence of code. Approximate it: match the whole file with `pattern-regex` and exclude files containing the content with `pattern-not-regex`. The finding then spans the whole file from line 1. https://docs.semgrep.dev/kb/rules/match-absence

```yaml
languages: [generic]
patterns:
  - pattern-regex: (?s)(.*)
  - pattern-not-regex: .*YOUR PATTERN TO BLOCK
```

- Language-specific rules generally do not match comments. Use `generic` or `pattern-regex`. https://docs.semgrep.dev/kb/rules/match-comments

```yaml
languages: [generic]
options:
  generic_ellipsis_max_span: 0
patterns:
  - pattern: TODO $...ACTION
  - pattern-not: TODO ... atlassian.net ...
```

## Aliengrep

An experimental generic engine available in Semgrep CE, enabled with `options.generic_engine: aliengrep` and `languages: [generic]`. Its feature set may change. https://docs.semgrep.dev/writing-rules/experiments/aliengrep

- Options: `generic_multiline: false` (single-line mode), `generic_caseless: true`, `generic_extra_word_characters: ["+", "/", "="]`, `generic_extra_braces: [["<", ">"]]`, `generic_braces` (replaces the whole brace set).
- `...` is lazy (shortest match). `....` spans lines in single-line mode. A leading or trailing `...` is anchored to the start or end of input.
- Back-references need identical text, whitespace included for `$...A`, and stay case-sensitive in caseless mode.
- In single-line mode newlines in the pattern match literally. Prefer the quoted form `"a\nb"` to avoid an accidental trailing newline.

## Debugging a pattern that does not match

- Run each sub-pattern alone. If all match, the Boolean logic is wrong. https://docs.semgrep.dev/troubleshooting/rules
- `semgrep scan --dump-ast` shows how Semgrep parsed a file. `--matching-explanations` adds matching details to `--json`. https://docs.semgrep.dev/cli-reference
