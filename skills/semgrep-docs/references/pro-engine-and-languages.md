# Pro engine, Pro rules and languages

## Contents

- What Semgrep CE analyzes
- What the Pro engine adds
- Turning on cross-file analysis
- Cross-file pitfalls
- Pro rules and rule overlap
- Language support
- Maturity levels
- Registry rulesets per language
- Supply Chain notes

Abbreviations used below: PRO = https://docs.semgrep.dev/semgrep-code/semgrep-pro-engine-intro , EX = https://docs.semgrep.dev/semgrep-code/semgrep-pro-engine-examples , LANG = https://docs.semgrep.dev/semgrep-ce-languages

## What Semgrep CE analyzes

- Single-file analysis: cross-function constant propagation, single-function taint, semantic matching. Nothing crosses a function boundary for taint or a file boundary for anything. https://docs.semgrep.dev/semgrep-pro-vs-oss https://docs.semgrep.dev/languages/javascript
- Many framework-specific Pro rules fail to return findings under CE. Full coverage needs `semgrep login && semgrep ci`. https://docs.semgrep.dev/languages/javascript
- CE has no triage, tuning or reporting. Export JSON or SARIF to another tool. https://docs.semgrep.dev/semgrep-pro-vs-oss

## What the Pro engine adds

- Cross-function analysis, on by default in Semgrep Code, and optional cross-file analysis. Cross-file includes cross-function. (PRO)
- Framework-specific control flow for Java, JavaScript, Python and TypeScript. (LANG)
- Beyond CE (EX):
  - Taint through wrapper functions over several hops and files.
  - Class inheritance: a rule on `ExampleException` also finds a subclass `BadRequest` declared in another file.
  - Typed metavariables that use cross-file inheritance.
  - Constants imported from another file are resolved, so `dangerous("Employees")` matches through an imported constant.
- `metavariable-name` and custom taint propagators need the Pro engine. https://docs.semgrep.dev/writing-rules/rule-syntax https://docs.semgrep.dev/writing-rules/data-flow/taint-mode/overview
- Semgrep Code and Supply Chain are free for up to 10 contributors. https://docs.semgrep.dev/semgrep-pro-vs-oss

## Turning on cross-file analysis

- Platform: Settings > General > Code > Cross-file analysis. Source code is not uploaded. (PRO)
- Locally: `semgrep install-semgrep-pro` (needs login), then `semgrep scan --pro --config rules.yaml .`. Reinstall the Pro binary after each CLI upgrade. (PRO) (EX) https://docs.semgrep.dev/kb/semgrep-ci/ci-vs-cli
- Flags: `--pro` (cross-file plus Pro languages), `--pro-intrafile` (cross-function in 1 file), `--pro-languages`, `--pro-path-sensitive` (implies `--pro-intrafile`). https://docs.semgrep.dev/cli-reference
- A custom rule only uses cross-file and cross-function analysis with this option. Without it the docs example misses the true positive (PRO):

```yaml
options:
  interfile: true
```

- In the Playground, switch the Pro toggle on. Cross-file true positives get a purple star. (EX)

## Cross-file pitfalls

- Diff-aware (PR and MR) scans never run cross-file analysis. Only full scans do. (PRO)
- Semgrep falls back to single-file analysis when cross-file preprocessing exceeds 5 GB of memory or the scan exceeds 3 hours. (PRO)
- For 1 or 2 problem repositories, `semgrep ci --pro-intrafile` overrides the platform setting. `semgrep ci --oss-only` reverts to the open source engine. (PRO) https://docs.semgrep.dev/kb/semgrep-ci/ci-vs-cli
- Name resolution differs from CE, so an `interfile: true` rule can give results that differ from CE. (PRO)
- CommonJS: a function defined first and assigned to `module.exports` later is not tracked. An inline `module.exports = function get_user() {...}` or an ES `export function` works. (EX)

```js
function get_user() { return get_user_input("example") }
module.exports = get_user   // not tracked across files
```

- Scan time grows with groups of mutually dependent files, in the worst case exponentially up to the limits. https://docs.semgrep.dev/kb/rules/rule-file-perf-principles

## Pro rules and rule overlap

- Pro rules are written by Semgrep, run in the Default ruleset of a platform account and skip languages absent from the repository at no cost. https://docs.semgrep.dev/semgrep-code/pro-rules
- Running a chosen Pro rule from the CLI requires a login. Copy the command from the rule card (Registry > Visibility > Pro rules > Run locally). https://docs.semgrep.dev/semgrep-code/pro-rules
- A CE rule alone, where an overlapping Pro rule exists, gives more false positives. Running both gives duplicates. Badges "Pro rule available" and "Upgrade available" point to the better rule. https://docs.semgrep.dev/kb/rules/pro-vs-community-secrets-vs-code-rules
- Check a rule's `license` metadata before relying on it. https://docs.semgrep.dev/languages/javascript

## Language support

From (LANG) and https://docs.semgrep.dev/supported-languages:

| Level in Semgrep Code | Languages |
| :-- | :-- |
| GA, cross-file dataflow | C/C++, C#, Go, Java, JavaScript, Kotlin, Python, TypeScript |
| GA, cross-function dataflow only | JSX, PHP, Ruby, Rust, Scala, Swift, Terraform |
| GA, no dataflow | Generic, JSON, Elixir |
| Beta | Apex |
| Experimental | Dart, Bash, Cairo, Circom, Clojure, Dockerfile, Hack, HTML, Jsonnet, Julia, Lisp, Lua, Move on Aptos, Move on Sui, OCaml, R, Scheme, Solidity, YAML, XML |

- In CE every language is community supported, at best effort, with single-function analysis and community rules. Elixir and Apex are not available in CE. (LANG)
- C#: CE parses up to C# 7.0, Semgrep Code up to C# 13. (LANG)
- Scala and Terraform have community rules only. (LANG)
- Pro rules are updated weekly for language syntax changes. Community rules vary. https://docs.semgrep.dev/kb/semgrep-code/support-for-language-versions

## Maturity levels

https://docs.semgrep.dev/references/language-maturity-levels

| Level | Parse rate | Pro rules | Syntax |
| :-- | :-- | :-- | :-- |
| GA | 99%+ | 10+ | Regex, equivalences, deep expressions, types |
| Beta | 95%+ | 5+ | Full metavariable support and equality |
| Experimental | 90%+ | 0+ | Ellipsis and basic metavariables |
| Community supported | 90%+ | 0+ | Same as Experimental |

## Registry rulesets per language

Semgrep Code defaults are `p/default` and `p/comment`. https://docs.semgrep.dev/languages/javascript

| Language | CE rulesets | Source |
| :-- | :-- | :-- |
| JavaScript, TypeScript | `p/javascript`, `p/typescript`, `p/nodejs`, `p/expressjs`, `p/nextjs`, `p/nestjs`, `p/koa`, `p/hapi`, `p/eslint`, `p/headless-browser` | https://docs.semgrep.dev/languages/javascript |
| Python | `p/default`, `p/python`, `p/trailofbits`, `p/xss` | https://docs.semgrep.dev/languages/python |
| Java | `p/default`, `p/java`, `p/findsecbugs` | https://docs.semgrep.dev/languages/java |
| Go | `p/default`, `p/golang`, `p/gosec` | https://docs.semgrep.dev/languages/go |
| C# | `p/default`, `p/csharp`, `p/gitlab` | https://docs.semgrep.dev/languages/csharp |
| Kotlin | `p/default`, `p/kotlin` | https://docs.semgrep.dev/languages/kotlin |
| Ruby | `p/default`, `p/ruby`, `p/brakeman` | https://docs.semgrep.dev/languages/ruby |
| Scala | `p/default`, `p/scala` | https://docs.semgrep.dev/languages/scala |

- The Python page links `p/xss` to the `p/trailofbits` URL. Check the ruleset before relying on it. https://docs.semgrep.dev/languages/python
- The Swift page lists no CE ruleset. https://docs.semgrep.dev/languages/swift

## Supply Chain notes

- Beta means CVE coverage without reachability. GA means reachability for every critical and high vulnerability. https://docs.semgrep.dev/references/language-maturity-levels
- C/C++, Terraform, Generic and JSON have no Supply Chain support. https://docs.semgrep.dev/supported-languages
- `bun.lock` is read, binary `bun.lockb` is not. Convert with `bun install --save-text-lockfile --frozen-lockfile --lockfile-only`, then delete `bun.lockb`. https://docs.semgrep.dev/release-notes/2026-10-19
- Dynamic dependency resolution (beta) scans without lockfiles using `--allow-local-builds`. It covers Maven and Gradle (Java, Kotlin), pip `setup.py` projects only (Python, CI and CLI only) and NuGet (C#, see below). It does not work on Windows for any package manager. https://docs.semgrep.dev/semgrep-supply-chain/set-up-and-configure https://docs.semgrep.dev/languages/python https://docs.semgrep.dev/languages/java https://docs.semgrep.dev/languages/kotlin
- C# without `packages.lock.json` needs `semgrep ci --allow-local-builds` and a `.csproj`, only in your own CI or locally on Linux or macOS. On Windows commit a lockfile (`RestorePackagesWithLockFile` true, then `dotnet restore`). https://docs.semgrep.dev/languages/csharp
