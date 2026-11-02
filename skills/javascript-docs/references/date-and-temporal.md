# Date and Temporal: choosing types, plain dates and times

## Contents

- Temporal first
- Choosing a Temporal type
- Rules shared by every Temporal type
- Plain types
- Maintaining Date code
- Old patterns

## Temporal first

- MDN calls `Date` legacy: use `Temporal` in new code and migrate when possible, after checking `Temporal` support in the target runtime. Polyfills such as `@js-temporal/polyfill` are linked. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Date , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal
- Convert with `date.toTemporalInstant()` and back with `new Date(instant.epochMilliseconds)`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal/Instant

## Choosing a Temporal type

- Exact moment only: `Instant`. Exact moment plus a local time for people: `ZonedDateTime`. Wall-clock time with no zone: `PlainDateTime`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal/ZonedDateTime
- Whole-day event: `PlainDate`. Daily recurring time: `PlainTime`. Yearly recurrence: `PlainMonthDay`. Whole month: `PlainYearMonth`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal/PlainDate , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal/PlainTime , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal/PlainMonthDay , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal/PlainYearMonth
- An `Instant` has no calendar or zone: call `toZonedDateTimeISO(zone)` before reading year or hour. `epochNanoseconds` is a BigInt. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal/Instant

## Rules shared by every Temporal type

- Prefer `Temporal.X.from()` to constructors. Constructors need `new` and take raw ISO-calendar fields. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal/PlainDate/PlainDate
- Objects are immutable: `with()`, `add()`, `round()` return new objects. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal/PlainDate/with
- `valueOf()` throws, so `<`, `>` and `-` throw. Use `compare()` (a ready `sort` comparator), `equals()`, `since()`/`until()`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal/PlainDate/valueOf

```js
dates.sort(Temporal.PlainDate.compare);
Temporal.PlainDate.compare(date1, date2); // -1
date1 > date2; // TypeError
```
https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal/PlainDate/valueOf , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal/PlainDate/compare

- `"" + t` throws, `` `${t}` `` works. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Addition
- `a.since(b)` is `a - b` and returns a `Duration`. `add()`/`subtract()` take a duration and return a point. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal/PlainDate/since
- `overflow` defaults to `"constrain"`: `PlainDate.from("2021-02-29")` silently gives 2021-02-28. For object input such as `{ year: 2021, month: 13, day: 1 }`, pass `{ overflow: "reject" }` to get `RangeError`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal/PlainDate/from
- Calendars: default is ISO 8601. Never assume 12 months, 7-day weeks or 365 days (read `monthsInYear`, `daysInWeek`, `daysInYear`). Use `era`+`eraYear` as a pair, pick `month` (order in the year) or `monthCode` (name, e.g. birthdays) and never mix them. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal#calendars , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal/PlainDate
- Convert to one calendar with `withCalendar("iso8601")` before `since()`, which throws on different calendars. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal/PlainDate/since
- `month` is 1-based, unlike the `Date` month index. `dayOfWeek` 1 is usually Monday. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal/PlainDate/month , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal/PlainDate/dayOfWeek
- When formatting many times with the same arguments, create one `Intl.DateTimeFormat` instead of calling `toLocaleString()` repeatedly. Never compare output to fixed strings. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal/PlainDate/toLocaleString

## Plain types

- Month-end arithmetic clamps, and order matters: `2021-01-31` plus `{ months: 1, days: 31 }` gives 2021-03-31, while adding days then months gives 2021-04-03. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal/PlainDate/add
- `PlainDate.add({ hours: 25 })` adds 1 day. `PlainTime.add()` wraps at midnight. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal/PlainDate/add , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal/PlainTime/add
- `PlainDate.from()` ignores a time, offset or zone in a string. `PlainDateTime` ignores an offset or zone too, but the offset must not be `Z`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal/PlainDate , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal/PlainDateTime
- Build `PlainMonthDay` and `PlainYearMonth` with `from()`: the constructors' `referenceYear` and `referenceDay` make `equals()` (and `PlainYearMonth.compare()`) treat equivalent values as different. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal/PlainMonthDay/PlainMonthDay , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal/PlainYearMonth/PlainYearMonth
- `PlainMonthDay` has `monthCode` and `day`, no `month`, `compare` or `add`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal
- For days between two months, go through `toPlainDate({ day: 1 })`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal/PlainYearMonth/since
- `with()` replaces some fields, `withPlainTime()` the whole time. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal/PlainDateTime/withPlainTime
- `toZonedDateTime()` defaults to `disambiguation: "compatible"`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal/PlainDateTime/toZonedDateTime

```js
const d = Temporal.Now.plainDateISO().since(date, { largestUnit: "years", smallestUnit: "months" });
new Intl.DurationFormat("en-US", { style: "long" }).format(d);
```
https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal/PlainDate/since

## Maintaining Date code

- Parse only the date time string format `YYYY-MM-DDTHH:mm:ss.sssZ` (shorter forms allowed). Other strings are implementation-defined and may not work across browsers. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Date#date_time_string_format , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Date/parse
- `"2011-10-10"` parses as UTC but `"2011-10-10T14:48:00"` as local time. Add `Z` or an offset. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Date#date_time_string_format
- `Date()` without `new` returns a string. `new Date(undefined)` is Invalid Date, `new Date(null)` the epoch. Never pass an array. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Date/Date
- Fields overflow: month 12 is next January, day 0 is the previous month's last day. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Date/Date#individual_date_and_time_component_values
- Numeric years 0 to 99 in the constructor and `Date.UTC()` map to 1900-1999. Use `setFullYear()` for them. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Date#interpretation_of_two-digit_years
- Setters mutate. A NaN argument makes the date invalid. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Date/setDate
- On an invalid date `toISOString()` throws, `toJSON()` returns `null`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Date/toISOString , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Date/toJSON
- Plain getters use the host zone. `getTimezoneOffset()` is positive behind UTC and depends on the date. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Date/getTimezoneOffset
- `date + 1` concatenates strings. Use `getTime()`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Date/Symbol.toPrimitive
- Prefer `performance.now()` to `Date.now()` for more precise elapsed time. Epoch seconds: `Math.floor(Date.now() / 1000)`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Date#calculating_elapsed_time

## Old patterns

| Old pattern | Replacement | Source |
| :-- | :-- | :-- |
| `getYear()`, `setYear()` | `getFullYear()`, `setFullYear()` | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Date/getYear |
| `toGMTString()` | `toUTCString()` | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Deprecated_and_obsolete_features#date |
