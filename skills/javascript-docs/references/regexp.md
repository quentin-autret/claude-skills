# Regular expressions

## Contents

- Building and choosing a method
- lastIndex and stateful flags
- Unicode modes u and v
- Groups, quantifiers and assertions
- Old patterns

## Building and choosing a method

- Build from input with `new RegExp(string, flags)` and wrap user text in `RegExp.escape()`. Do not hand-roll escaping. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/RegExp/escape

```js
new RegExp(`https?://${RegExp.escape(domain)}(?=/)`, "g")
```
https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/RegExp/escape

- Pick the narrowest method: `test()` for a boolean, `match()` with `g` for all matches without groups, `matchAll()` for all matches with groups, `search()` for a position. Keep `exec()` for manual `lastIndex` control. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/RegExp/exec
- Read submatches by name with `(?<name>...)`. Add the `d` flag for start and end indices. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Regular_expressions/Named_capturing_group , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/RegExp/hasIndices

```js
const { author, timestamp } = /^(?<timestamp>\d+),(?<author>.+)$/.exec(entry).groups;
```
https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Regular_expressions/Named_capturing_group

- Do not parse URLs or HTML with regex: use the URL API or the HTML Sanitizer API. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/RegExp , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Regular_expressions/Quantifier
- Split lines with `/\r?\n/`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/RegExp
- `test()` with no argument searches for `"undefined"`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/RegExp/test
- `new RegExp(regex, flags)` with a RegExp as first argument replaces that regex's flags and resets `lastIndex` to 0. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/RegExp/RegExp

## lastIndex and stateful flags

- With `g` or `y`, `lastIndex` persists across calls and strings. `test()` and `exec()` reset it only on failure, so a reused global regex can fail on a new string. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/RegExp/test
- `exec()` loop traps: a literal inside the `while` condition restarts every time, a missing `g` never advances, and an empty match (`/^/gm`) needs a manual `lastIndex` bump. Prefer `matchAll()`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/RegExp/exec
- A regex literal or constructor call placed in a `while` condition recreates the regex on every iteration, which resets `lastIndex`. Hold the regex in a variable. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/RegExp/exec
- Use `y` (sticky) for tokenizers: it matches only at `lastIndex`. Set `lastIndex` first. `^` still means input start (or line start under `m`). https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/RegExp/lastIndex , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/RegExp/sticky

```js
const stringPattern = /"[^"]*"/y;
stringPattern.lastIndex = 16;
stringPattern.exec(input); // ['"Hello world"']
```
https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/RegExp/lastIndex

- A sticky non-global regex in `match()`/`replace()` keeps `lastIndex` after success, so repeated calls differ. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/RegExp/Symbol.replace , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/RegExp/sticky

## Unicode modes u and v

- Without `u` or `v`, patterns work on UTF-16 code units: `.` matches half an emoji, `\p{...}` and `\u{...}` are read as identity escapes instead of Unicode escapes, so `/\u{61}/` means `u` 61 times. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/RegExp/unicode , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/RegExp/dotAll
- The `v` flag is the upgrade of `u`, with `&&` intersection, `--` subtraction and `\q{...}` strings in classes. `u` and `v` together are a `SyntaxError`. `v` requires escaping `( ) [ { } / - |` in classes. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Regular_expressions/Character_class , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/RegExp/unicode

```js
str.match(/[\p{Script_Extensions=Greek}&&\p{Letter}]/gv);
str.match(/[\p{Decimal_Number}--\d]/gv);
```
https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Regular_expressions/Character_class

- Prefer `\p{...}` over hand-written ranges. `\w` and `\W` match ASCII only. Match a writing system with `\p{scx=...}` rather than `\p{sc=...}`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/RegExp , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Regular_expressions/Word_boundary_assertion , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Regular_expressions/Unicode_character_class_escape
- `v` does not treat grapheme clusters as one character: `/[🇺🇳]/v` matches `"🇺"`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/RegExp/unicodeSets
- With `u`+`i`, U+212A (Kelvin sign) and `ſ` (U+017F) match `[a-z]`. Simple case folding never maps `ß` to `ss`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/RegExp/ignoreCase
- `\b` only knows ASCII word characters (plus case-folded equivalents under `u`+`i`) and fails for languages without spaces. Use `Intl.Segmenter`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Regular_expressions/Word_boundary_assertion

## Groups, quantifiers and assertions

- Apply `i`, `m` or `s` to part of a pattern with `(?i:...)` or `(?-m:...)`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Regular_expressions/Modifier

```js
/(?:var|let|const) (?i:foo|bar)\b/
```
https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Regular_expressions/Modifier

- Under `m`, `^` and `$` match every line. Anchor to the whole input with `(?-m:^)`. `\A`, `\z`, `\Z` are experimental and need `u` or `v`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Regular_expressions/Input_boundary_assertion , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Regular_expressions/Buffer_boundary_assertion
- Greedy quantifiers maximize repetitions, not length. Alternation takes the first alternative that lets the whole match succeed. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Regular_expressions/Quantifier , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Regular_expressions/Disjunction
- `/a{1, 3}/` (with a space) matches literal text without `u`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Regular_expressions/Quantifier
- A quantified group keeps its last iteration only, and nested groups reset, possibly to `undefined`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Regular_expressions/Capturing_group
- A backreference to a group not yet matched matches empty: `/\1(a)/.test("a")` is true. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Regular_expressions/Backreference
- Lookbehind matches right to left, which changes what greedy groups capture. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Regular_expressions/Lookbehind_assertion
- Inside a non-`v` class, `\b` is backspace and `.` is literal. With `i`, `/[E-f]/i` also matches `[ \ ] ^ _` and the backtick. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Regular_expressions/Character_class
- Duplicate group names are allowed only in different alternatives. Check support. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Regular_expressions/Named_capturing_group

## Old patterns

| Old pattern | Replacement | Source |
| :-- | :-- | :-- |
| `while ((m = re.exec(s)))` | `s.matchAll(re)` | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/RegExp/exec |
| `RegExp.$1` to `RegExp.$9` (deprecated) | the indexes of the array returned by `exec()` or `match()` | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/RegExp/n |
| `re.compile()` | `new RegExp(...)` | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/RegExp/compile |
| Manual escaping with `replaceAll` | `RegExp.escape()` | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/RegExp/escape |
| Lenient non-Unicode syntax (bare `{`, `\N` octal) | the `u` or `v` flag | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/RegExp/unicode |
| `[Ѐ-ӿ]` style ranges | `\p{scx=Cyrl}` | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/RegExp |
