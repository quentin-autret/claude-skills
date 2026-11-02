# Strings, Unicode and URI encoding

## Contents

- Code units, code points and graphemes
- Comparing and normalizing
- replace, match and split
- Slicing and building
- URI encoding
- Old patterns

## Code units, code points and graphemes

- Strings are UTF-16 code units. `length`, indexes, `charAt`, `charCodeAt`, `at`, `slice` and `substring` count code units, so an emoji counts 2 or more. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/length , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String#utf-16_characters_unicode_code_points_and_grapheme_clusters
- Count or iterate code points with `for...of` or `[...str]`. Count user-perceived characters with `Intl.Segmenter`: even `[...str]` splits flags, skin tones and ZWJ emoji. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String#utf-16_characters_unicode_code_points_and_grapheme_clusters , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/length

```js
const segmenter = new Intl.Segmenter("en-US", { granularity: "grapheme" });
[...segmenter.segment(str)].length;
```
https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/length

- `split("")` breaks surrogate pairs. `split(/(?:)/u)` splits by code point. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/split

```js
"😄".split(""); // ['\ud83d', '\ude04']
"😄😄".split(/(?:)/u); // [ "😄", "😄" ]
```
https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/split

- Do not loop by index with `codePointAt()`: it visits a code point twice. Use `for...of`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/codePointAt
- Build astral characters with `String.fromCodePoint()`, not `fromCharCode()`. Invalid code points throw `RangeError`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/fromCodePoint
- Detect lone surrogates with `isWellFormed()` and replace them with U+FFFD with `toWellFormed()`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/isWellFormed , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/toWellFormed
- Maximum length is engine dependent. Exceeding it throws `RangeError: Invalid string length`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/length

## Comparing and normalizing

- Compare case-insensitively with `localeCompare(b, locale, { sensitivity: "accent" })` or `Intl.Collator`, not `toUpperCase()` or `toLowerCase()` equality: the first makes `"ß"` equal `"ss"`, the second reports Turkish `ı` and `I` as unequal. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String#comparing_strings

```js
const areEqual = (str1, str2, locale = "en-US") =>
  str1.localeCompare(str2, locale, { sensitivity: "accent" }) === 0;
areEqual("ß", "ss", "de"); // false
areEqual("ı", "I", "tr"); // true
```
https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String#comparing_strings

- Test only the sign of `localeCompare()`: values other than -1 and 1 occur. Sort numerically with `{ numeric: true }`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/localeCompare
- Call `normalize()` (NFC by default) before comparing text that may mix composed and decomposed forms. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/normalize
- Use string primitives. `typeof new String("a")` is `"object"`, and `eval()` returns a String object as is instead of running it. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String#string_primitives_and_string_objects
- Coerce with `` `${x}` `` or `String(x)`. `String(x)` does not throw on Symbols. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String#string_coercion

## replace, match and split

- `replace()` with a string pattern replaces the first occurrence only. Use `replaceAll()`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/replace
- Replacement strings interpret `$&`, `$1`, `$<name>`, `$$`. User text containing `$` is misread: pass a function replacer, whose return value is used as is. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/replace

```js
"foo".replace(/(f)/, "$2"); // "$2oo"
"foo".replace("f", "$1"); // "$1oo"
```
https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/replace

- The replacer gets `groups` last only when named groups exist, so `args.at(-2)` is not always the offset. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/replace
- `replaceAll()` and `matchAll()` throw for a regex without `g`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/replaceAll
- Get every match with its groups through `matchAll()`, not an `exec()` loop. `match()` returns `null` on no match and drops groups with `g`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/matchAll , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/match
- `includes()`, `startsWith()` and `endsWith()` throw for a RegExp and return true for `""`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/includes , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/startsWith , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/endsWith
- A regex separator with capture groups splices the captures into `split()` output. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/split
- Split words with `Intl.Segmenter`, not `split(" ")`, for languages without spaces. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/Segmenter

## Slicing and building

- Use `slice()`: negative indexes count from the end. `substring()` clamps negatives to 0 and swaps reversed arguments. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/substring
- `charAt()` out of range returns `""`, brackets return `undefined`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/charAt
- Build a regex with dynamic parts from `String.raw` to avoid double escaping. Never use `String.raw` itself as an identity tag. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/raw

```js
const makeURLRegExp = (path) =>
  new RegExp(String.raw`https://developer\.mozilla\.org/${path}`);
const html = (strings, ...values) => String.raw({ raw: strings }, ...values);
```
https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/raw

- A tagged template's strings array is the same frozen object on every evaluation, so it can serve as a cache key. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Template_literals#tagged_templates

## URI encoding

- Encode each dynamic part with `encodeURIComponent()`. `encodeURI()` is for a whole, already valid URL and leaves `&` unescaped. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/encodeURI

```js
const name = "Ben & Jerry's";
const goodLink = `https://example.com/?choice=${encodeURIComponent(name)}`;
```
https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/encodeURI

- For `application/x-www-form-urlencoded`, replace `%20` with `+`. For RFC 3986, also escape `!'()*`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/encodeURIComponent
- Both functions throw `URIError` on a lone surrogate: check `isWellFormed()` first. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/encodeURIComponent

## Old patterns

| Old pattern | Replacement | Source |
| :-- | :-- | :-- |
| `substr(start, length)` | `slice()` or `substring()`, not a mechanical swap | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/substr |
| `trimLeft()`, `trimRight()` | `trimStart()`, `trimEnd()` | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/trimEnd , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/trimStart |
| `escape()`, `unescape()` | `encodeURIComponent()` or `encodeURI()`, and `decodeURIComponent()` or `decodeURI()` for `unescape()` | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/escape , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/unescape |
| `bold()`, `fontsize()`, `link()` and other HTML methods | DOM APIs such as `document.createElement()` | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String#html_wrapper_methods |
| `"" + x` | `` `${x}` `` or `String(x)` | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String#string_coercion |
| `fromCharCode` / `charCodeAt` for astral text | `fromCodePoint` / `codePointAt` | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/codePointAt |
| `str.replace(/x/g, y)` for a literal | `str.replaceAll("x", y)` | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/replace |
