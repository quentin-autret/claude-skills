# Temporal: ZonedDateTime, time zones, Duration and Now

## Contents

- Time zones and ZonedDateTime
- Offsets and DST ambiguity
- Arithmetic and differences
- Duration
- Temporal.Now
- Old patterns

## Time zones and ZonedDateTime

- Use a named IANA zone (`Europe/Paris`), not a fixed offset (`+05:30`): derived values across a DST change get the wrong local time. Rarely use `"UTC"` in a `ZonedDateTime`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal/ZonedDateTime#time_zones_and_offsets
- `ZonedDateTime.from()` requires a `[zone]` annotation. Parse a plain ISO string through `Instant.from()`, which reads only the offset. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal/ZonedDateTime/from#creating_a_zoneddatetime_from_an_iso_8601_rfc_3339_string

```js
const instant = Temporal.Instant.from("2021-07-01T12:34:56+02:00");
const zdt = instant.toZonedDateTimeISO("America/New_York");
```
https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal/ZonedDateTime/from

- Serialize with `toString()`/`toJSON()`, which use the RFC 9557 format and keep the `[zone]` annotation of a `ZonedDateTime`. An `Instant` is a timestamp with no time zone. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal/ZonedDateTime/toString , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal/ZonedDateTime/toJSON , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal
- Show zones to people with `toLocaleString()`, not the raw `timeZoneId`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal/ZonedDateTime/timeZoneId
- `timeZoneId` keeps the identifier you passed (never canonicalized to the primary one; only the letter case of a named zone is normalized, and an offset like `+07` becomes `+07:00`). `Temporal.Now.timeZoneId()` never returns an alias. Primary identifiers can be renamed over time. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal/ZonedDateTime/timeZoneId , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal/Now/timeZoneId , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/Locale/getTimeZones
- `Intl.DateTimeFormat` throws on a `ZonedDateTime`. Call its own `toLocaleString()` or convert to `PlainDateTime`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/DateTimeFormat/format
- A day is not always 24 hours (`hoursInDay`). Get midnight with `startOfDay()`, which returns the first existing time. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal/ZonedDateTime/hoursInDay , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal/ZonedDateTime/startOfDay
- `toPlainDateTime()` drops the zone: later arithmetic ignores DST and looks right until a transition. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal/ZonedDateTime/toPlainDateTime
- `ZonedDateTime.compare()` compares instants. Convert to `PlainDateTime` to compare wall-clock times. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal/ZonedDateTime/compare
- `getTimeZoneTransition()` may be wrong about the future. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal/ZonedDateTime/getTimeZoneTransition

## Offsets and DST ambiguity

- `disambiguation` defaults to `"compatible"`: a time in a gap moves later, a repeated time picks the earlier one. Use `"reject"` to throw. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal/ZonedDateTime#ambiguity_and_gaps_from_local_time_to_utc_time

```js
Temporal.ZonedDateTime.from("2024-03-10T02:05:00[America/New_York]", { disambiguation: "earlier" }); // 01:05-05:00
```
https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal/ZonedDateTime/from

- The `offset` option defaults to `"reject"` in `from()` (throws when stored offset and zone rules disagree) and `"prefer"` in `with()`. Pick `"ignore"` for calendar events (keep local time) and `"use"` for exact-time schedules. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal/ZonedDateTime#offset_ambiguity

```js
Temporal.ZonedDateTime.from("2019-12-23T12:00:00-02:00[America/Sao_Paulo]", { offset: "use" });    // 11:00-03:00
Temporal.ZonedDateTime.from("2019-12-23T12:00:00-02:00[America/Sao_Paulo]", { offset: "ignore" });  // 12:00-03:00
```
https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal/ZonedDateTime/from

- `Z` is not `+00:00`: `Z` states a UTC instant and ignores `offset`, `+00:00` is a local offset checked against the zone. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal/ZonedDateTime#offset_ambiguity

## Arithmetic and differences

- Do date arithmetic on `ZonedDateTime`: the date part keeps clock time, the time part uses real elapsed time. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal/ZonedDateTime/add
- `overflow` defaults to `"constrain"` (2024-08-31 plus 1 month is 2024-09-30). https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal/ZonedDateTime/add
- `ZonedDateTime.since()` defaults to hours (`PT25H` across a DST end). Day or larger units need the same zone. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal/ZonedDateTime/since

```js
end.since(start).toString();                          // PT25H
end.since(start, { largestUnit: "days" }).toString(); // P1D
```
https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal/ZonedDateTime/since

- `Instant.since()` accepts only units up to `hours` and defaults to seconds, so the result is unbalanced (hours may exceed 24). Round the result with `largestUnit` and a `relativeTo` to get days or years. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal/Instant/since

```js
const d = now.since(lastUpdated, { smallestUnit: "minutes" });
d.round({ largestUnit: "years", relativeTo: now.toZonedDateTimeISO("UTC") });
```
https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal/Instant/since

## Duration

- Compare with `Temporal.Duration.compare()` or `total(unit)`: `valueOf()` throws. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal/Duration/valueOf
- Duration arithmetic with years, months or weeks throws (`dur1.add(dur2)` is a `RangeError`). Add them to a date instead. `round()`, `total()` and `compare()` need `relativeTo` for them. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal/Duration#calendar_durations

```js
const dur1 = Temporal.Duration.from({ years: 1 });
const dur2 = Temporal.Duration.from({ months: 1 });
const start = Temporal.PlainDate.from("2021-01-01");
start.add(dur1).add(dur2).since(start); // "P396D"
```
https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal/Duration#calendar_durations

- Without a zoned `relativeTo`, a day counts as 24 hours. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal/Duration#calendar_durations
- All fields share one sign. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal/Duration#duration_sign
- Durations are not balanced: 90 minutes stays. `round()` balances. ISO strings lose unbalanced subseconds, so serialize as an object to keep them. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal/Duration#duration_balancing

```js
duration.round({ largestUnit: "hours" });                        // 3d1h41m5s -> 73:41:05
duration.round({ smallestUnit: "minutes", roundingIncrement: 15 }); // 1h17m -> 1h15m
```
https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal/Duration/round

- `round()` defaults to `halfExpand`. A `roundingIncrement` must divide the unit (hours: 1, 2, 3, 4, 6, 8, 12). https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal/Duration/round
- Format repeatedly with one `Intl.DurationFormat`, since `toLocaleString()` searches the localization data on every call. Where `Intl.DurationFormat` is unsupported, `toLocaleString()` returns the ISO string. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal/Duration/toLocaleString
- `P3W1D` and signed forms are ISO 8601-2 extensions other programs may reject. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal/Duration

## Temporal.Now

- Precision may be reduced by browsers (Chrome rounds to 0.1 ms, Firefox gives whole milliseconds). Reading `epochNanoseconds` adds no precision. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal/Now
- `Temporal.Now.plainTimeISO()` has no calendar despite its name. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal/Now/plainTimeISO

## Old patterns

| Old pattern | Replacement | Source |
| :-- | :-- | :-- |
| `Date` with UTC or host zone only | `ZonedDateTime` with an IANA zone | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal |
| Millisecond timestamps | `Instant` (nanoseconds) | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal |
| `d1 > d2` on durations | `Temporal.Duration.compare(d1, d2)` | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal/Duration/valueOf |
