# Intl: locales, formatting, collation, segmentation

## Contents

- Locales
- Formatters in general
- DateTimeFormat
- NumberFormat
- Collator
- Segmenter, PluralRules and other formatters
- Old patterns

## Locales

- Pick the locale from the language your page is actually shown in (language switcher, `document.documentElement.lang`), not blindly from `navigator.languages`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Internationalization#figuring_out_the_locale
- Pass an array as a fallback list. With none supported the runtime default applies. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl

```js
new Intl.DateTimeFormat(["ban", "id"]).format(date); // falls back to Indonesian
```
https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/DateTimeFormat

- Find supported locales with `Intl.X.supportedLocalesOf()`. No API lists them all. Feature-test values with `Intl.supportedValuesOf(key)` before loading a polyfill. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Internationalization#figuring_out_the_locale , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/supportedValuesOf
- Tags use hyphens: `"EN_US"` throws in `Intl.getCanonicalLocales()`. Well-formed unknown tags fall back. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/getCanonicalLocales , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl
- Options beat Unicode extension keys (`-u-ca-`, `-u-nu-`). https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/DateTimeFormat/DateTimeFormat
- `new Intl.Locale("en-US").calendar` is `undefined`. Read the effective default from `resolvedOptions()`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Internationalization#locale_information
- `Intl.Locale` takes one tag, not an array. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/Locale/Locale

## Formatters in general

- Create a formatter once and reuse `format()`. `toLocaleString()` searches locale data on every call. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Internationalization#formatting_data
- Never compare output with fixed strings: it varies by engine and may hold non-breaking spaces or bidi marks. Test with snapshots, every supported locale, several engines. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Internationalization#understanding_the_return_value
- `format` and `compare` are bound: pass them to `map` or `sort` directly. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/NumberFormat/format , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/Collator/compare
- `Collator`, `DateTimeFormat`, `NumberFormat` work without `new`. `Locale`, `Segmenter`, `PluralRules`, `ListFormat`, `RelativeTimeFormat`, `DisplayNames` and `DurationFormat` throw a `TypeError` without it. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/Collator/Collator , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/Locale/Locale

## DateTimeFormat

- Always pass the date to `format()`. Omitting it formats now. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/DateTimeFormat/format
- Default to `dateStyle`/`timeStyle`. Use component options only for fine control: mixing both is invalid. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/DateTimeFormat/DateTimeFormat
- `hour12` overrides `hourCycle`. `dayPeriod` works with a 12-hour clock only. Narrow weekdays can collide. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/DateTimeFormat/DateTimeFormat
- `timeZone` defaults to the runtime zone. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/DateTimeFormat/DateTimeFormat
- Show ranges with `formatRange()`, both ends of one type. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/DateTimeFormat/formatRange

```js
const fmt = new Intl.DateTimeFormat("en", { year: "numeric", month: "short", day: "numeric" });
fmt.formatRange(date1, date3); // 'Jan 10 – 20, 1906'
```
https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/DateTimeFormat/formatRange

- `Temporal.ZonedDateTime` throws in `format()`. Plain Temporal types need a matching field. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/DateTimeFormat/format

## NumberFormat

- `style: "currency"` needs `currency`, `style: "unit"` needs `unit`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/NumberFormat/NumberFormat
- `style: "percent"` multiplies the value by 100. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Internationalization#number_formatting
- Digit options interact. Setting `minimumFractionDigits` also applies the default `maximumFractionDigits` (3 for plain numbers, so `4.33145` with `minimumFractionDigits: 2` gives `4.331`), and the reverse. Any significant-digits option makes fraction options ignored under `roundingPriority: "auto"`. With `roundingIncrement`, both fraction options must be equal or a `RangeError` is thrown. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/NumberFormat/NumberFormat
- Rounding defaults to `halfExpand`. Choose another with `roundingMode`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/NumberFormat/NumberFormat
- Format integers above `MAX_SAFE_INTEGER` as BigInt or a decimal string. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/NumberFormat/format

```js
new Intl.NumberFormat("en-US").format(1234567891234567891n); // 1,234,567,891,234,567,891
new Intl.NumberFormat("en-US").format("1234567891234567891");
```
https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/NumberFormat/format

## Collator

- Sort with `new Intl.Collator(locale).compare`. Test only its sign. Order is locale dependent. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/Collator , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/Collator/Collator
- Sort `"2"` before `"10"` with `numeric: true`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/Collator/Collator
- Filter accent- and case-insensitively with `usage: "search"` and `sensitivity: "base"`, testing for zero. Never sort with `usage: "search"`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/Collator/Collator , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/Collator/compare

```js
const collator = new Intl.Collator("fr", { usage: "search", sensitivity: "base" });
a.filter((v) => collator.compare(v, "congres") === 0);
```
https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/Collator/compare

## Segmenter, PluralRules and other formatters

- Split graphemes, words or sentences with `Intl.Segmenter`, never by code units or spaces. `containing(i)` takes code unit indexes. `isWordLike` exists only for `granularity: "word"`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/Segmenter , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/Segmenter/segment/Segments/containing
- `Intl.PluralRules` returns a category (`one`, `few`, `other`...), not a word. `type: "ordinal"` gives English suffix categories. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/PluralRules

```js
const rules = new Intl.PluralRules("en-US", { type: "ordinal" });
const suffixes = new Map([["one","st"],["two","nd"],["few","rd"],["other","th"]]);
const ord = (n) => `${n}${suffixes.get(rules.select(n))}`;
```
https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/PluralRules

- `RelativeTimeFormat` with `numeric: "auto"` says "yesterday" rather than "1 day ago". https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/RelativeTimeFormat/format
- Join lists with `Intl.ListFormat`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/ListFormat/ListFormat
- Format durations with `Intl.DurationFormat` (`style: "digital"` gives `1:50:00`). https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/DurationFormat/DurationFormat
- `DisplayNames.of()` with `fallback: "none"` returns `undefined` for well-formed codes that have no data (`"code"` returns the code itself). Structurally invalid codes throw `RangeError`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/DisplayNames/of

## Old patterns

| Old pattern | Replacement | Source |
| :-- | :-- | :-- |
| `locale.weekInfo`, `locale.timeZones` getters | `getWeekInfo()`, `getTimeZones()` | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/Locale/getWeekInfo , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/Locale/getTimeZones |
| Calendar `islamicc` | `islamic-civil` | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/supportedValuesOf |
| `sv-u-co-reformed` | plain `sv` | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/supportedValuesOf |
| `str.split(" ")` for words | `Intl.Segmenter` | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/Segmenter |
| `toLocaleString()` in a loop | one reused formatter | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Internationalization#formatting_data |
