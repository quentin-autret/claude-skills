# Taint mode and dataflow

## Contents

- Engine limits
- Constant propagation
- Symbolic propagation
- Taint rule shape
- exact and the surprising defaults
- Sources, sanitizers and sinks on a metavariable
- Side effects
- Propagators
- Options that cut false positives
- Field and index sensitivity
- Labels
- Pro engine features
- Why a flow is not reported
- Old patterns

Abbreviations used below: TO = https://docs.semgrep.dev/writing-rules/data-flow/taint-mode/overview , TA = https://docs.semgrep.dev/writing-rules/data-flow/taint-mode/advanced , CP = https://docs.semgrep.dev/writing-rules/data-flow/constant-propagation

## Engine limits

- Dataflow is intraprocedural by default. It is not path sensitive (all paths are considered), has no pointer or shape analysis (aliasing through arrays or pointers is missed, array elements are not tracked) and ignores the effects of eval-like functions. Expect false positives and false negatives. https://docs.semgrep.dev/writing-rules/data-flow/data-flow-overview
- Dataflow runs on an intermediate language whose translation is incomplete. Unsupported constructs are silently skipped, the scan does not fail. Coverage is lower than for plain matching. https://docs.semgrep.dev/writing-rules/data-flow/status
- Field sensitivity exists for taint only, not for constant propagation. https://docs.semgrep.dev/writing-rules/data-flow/data-flow-overview

## Constant propagation

- Semgrep tracks variables that must hold a boolean, number or string, and folds them: `return 42` matches `return x` when `x` folds to 42. `metavariable-comparison` works on such variables too. (CP)
- Semgrep CE propagates within 1 file (intrafile). Semgrep AppSec Platform supports interprocedural (cross-function) and interfile (cross-file) constant propagation. (CP)
- Semgrep assumes constants are immutable and unchanged by calls, which yields false positives in languages with mutable strings (C, Ruby). Exception: a method call whose result is ignored may mutate its receiver. (CP)
- A Java `public String REGEX = "(a+)+$";` field is not propagated to other classes, since it could be mutated. `private` or `public final` makes it propagate. When propagation seems broken, check whether the constant could be mutable. (CP)
- Disable per rule with `options.constant_propagation: false`. (CP)

## Symbolic propagation

- Experimental, off by default, enabled with `options.symbolic_propagation: true`. It matches through assignments: `df = pandas.DataFrame(x); ix = df.index; ix.set_value(...)` matches `pandas.DataFrame(...).index.set_value(...)`. It stops at branches (if, loops). https://docs.semgrep.dev/writing-rules/experiments/symbolic-propagation
- `value($X)` in `message` prints the propagated value (512) rather than the code (`size`), falling back to the matched text. https://docs.semgrep.dev/writing-rules/experiments/display-propagated-metavariable

## Taint rule shape

```yaml
rules:
  - id: user-input-to-eval
    mode: taint
    languages: [python]
    severity: HIGH
    message: User input reaches $SINK
    pattern-sources:
      - pattern: get_user_input(...)
    pattern-sanitizers:
      - pattern: sanitize_input(...)
    pattern-sinks:
      - patterns:
          - pattern: eval($SINK)
          - focus-metavariable: $SINK
```

- `pattern-sources` is required. `pattern-propagators`, `pattern-sanitizers` and `pattern-sinks` complete it. Each list acts like a `pattern-either`, and each item starts with `pattern`, `patterns`, `pattern-either` or `pattern-regex`. (TO)
- Taint flows through assignments, operators and calls (arguments to result) without any propagator. (TO)
- Same-named metavariables in sources and sinks are different metavariables. `options.taint_unify_mvars: true` unifies them, which the docs advise against unless you know why. (TO) (TA)
- `message` can use sink metavariables, plus source metavariables whose names do not clash with sink ones. (TA)

## exact and the surprising defaults

| Where | `exact` default | Meaning of the default | Source |
| :-- | :-- | :-- | :-- |
| source | false | every subexpression of the match is a source | (TO) |
| sanitizer | false | every subexpression of the match is sanitized | (TO) |
| sink | true | only the whole match is a sink | (TO) |

- With non-exact sources, `source(sink(x))` reports `sink(x)`. With non-exact sanitizers, `sanitize(sink("taint"))` is not reported. (TO)
- `exact: false` on a sink makes `tainted` a sink in `sink("foo" if tainted else "bar")`. (TO)
- Write `exact: false` explicitly when a rule relies on it, so a later default change does not break the rule. (TO)

## Sources, sanitizers and sinks on a metavariable

- Function parameter as source: `pattern-inside: def foo($X, ...): ...` plus `focus-metavariable: $X`. `pattern: $X` is wrong: it matches every use of `x`, so a sanitized `x` gets tainted again. (TA)
- 1 argument as sink: `pattern: sink($SINK, ...)` plus `focus-metavariable: $SINK`. A bare `sink(...)` fires on taint in any argument. Anything matchable can be a sink, for example `pattern-inside: $ARRAY[$SINK]` with focus. (TA)

## Side effects

- `by-side-effect` defaults to false on sources and sanitizers (values `false`, `true`, `only`) and to true on propagators. (TO)
- A side-effect source or sanitizer needs `focus-metavariable` on the l-value, otherwise `by-side-effect` does nothing. `true` also taints the matched occurrence, `only` does not. (TA)

```yaml
pattern-sources:
  - patterns:
      - pattern: make_tainted($X)
      - focus-metavariable: $X
    by-side-effect: true
pattern-sanitizers:
  - patterns:
      - pattern: check_if_safe($X)
      - focus-metavariable: $X
    by-side-effect: true
```

- Without `by-side-effect`, `check_if_safe(x)` only sanitizes the `x` inside the call, not later uses. (TA)

## Propagators

```yaml
pattern-propagators:
  - pattern: $S.add($E)
    from: $E
    to: $S
```

- Custom propagators are a Pro feature and work intraprocedurally only. `from` and `to` are metavariables matching expressions. (TO) (TA)
- `by-side-effect: false` on a propagator limits it to that occurrence. (TA)
- Java lambdas: `pattern: $C.forEach(($X) -> ...)`, `from: $C`, `to: $X`. (TA)

## Options that cut false positives

All default to false (TA).

- `taint_assume_safe_booleans`, `taint_assume_safe_numbers`: values of those types are never tainted. Relies on type inference, Pro feature, third-party types ignored.
- `taint_assume_safe_indexes`: `E[I]` is tainted only when `E` is. By default `obj[tainted]` is tainted.
- `taint_assume_safe_functions`: `F(E)` is not tainted by `E`. By default an opaque call passes taint from any argument to its result. In CE every call is opaque. With the Pro engine this only covers functions without a findable definition. Re-allow chosen functions with propagators.
- `taint_only_propagate_through_assignments` (experimental): only `<l-value> = <tainted-expr>` propagates.

## Field and index sensitivity

- Basic field sensitivity: `x.a.b` can be tainted while `x` is not, and `x.a.b` can be sanitized while `x.a` stays tainted. Taint is tracked per variable, not per memory object, without aliasing. (TA)
- Index sensitivity (Pro, experimental) covers built-in `a[E]` with constant integer or string indexes only. Sanitizing an arbitrary `a[i]` cleans every index. (TA)

## Labels

Experimental. `label:` names a taint kind, `requires:` takes a Python Boolean expression (`A and not B`). (TA)

```yaml
pattern-sources:
  - pattern: user_input
    label: INPUT
  - pattern: evil(...)
    requires: INPUT
    label: EVIL
pattern-sinks:
  - pattern: sink(...)
    requires: EVIL
```

- Per-metavariable sink requirements: `requires:` as a list such as `- $SINK: BAD`. (TA)

## Pro engine features

- `--pro-intrafile` tracks taint across functions in 1 file, nearly as fast as intraprocedural. `--pro` adds cross-file analysis for a subset of languages, and each rule must set `options.interfile: true`. (TO)
- Cross-file analysis needs at least 4 GB of memory per core, 8 GB recommended. (TO)
- `options.taint_focus_on: source` reports the source location. (TA)
- `at-exit: true` on a sink (experimental) reports where control leaves the function when any tainted l-value reaches it, for leak checks such as unclosed files. (TA)
- `control: true` on a source (experimental) tracks reachability instead of data. (TA)

## Why a flow is not reported

- Semgrep reports 1 trace per source and sink pair, the shortest one. With `--pro` a shorter cross-file path hides a longer one. https://docs.semgrep.dev/kb/semgrep-code/finding_all_taints
- Inspect traces locally with `semgrep scan --dataflow-traces` or `semgrep ci --dry-run`, which sends nothing to the platform. To see the next path, make the shortest one untainted and rerun. https://docs.semgrep.dev/kb/semgrep-code/finding_all_taints

## Old patterns

| Old | Current | Source |
| :-- | :-- | :-- |
| `pattern-inside: make_tainted($X) ...` plus `pattern: $X` as a source | `by-side-effect: true` with `focus-metavariable: $X`. The old form overrides later sanitization and breaks across branches | (TA) |
| Join mode to link findings across files | `options.interfile: true` with the Pro engine. Join mode is experimental and unmaintained | https://docs.semgrep.dev/semgrep-code/semgrep-pro-engine-intro |
