# Dates, times and time zones

## Contents
- Choosing a type
- Input and literals
- Time zones
- Intervals and arithmetic
- Current time
- extract, date_trunc, date_bin
- Formatting and parsing
- Old patterns

## Choosing a type

- Store instants as `timestamptz`. It stores no zone: input is converted to UTC (explicit offset, else session `TimeZone`) and output is shown in the session `TimeZone`. https://www.postgresql.org/docs/18/datatype-datetime.html
- Avoid `time with time zone`: the docs call its usefulness questionable. https://www.postgresql.org/docs/18/datatype-datetime.html
- Resolution is 1 microsecond. `timestamp(p)` and `interval(p)` take 0 to 6 fractional digits. https://www.postgresql.org/docs/18/datatype-datetime.html

## Input and literals

- The literal type is never inferred from content: `TIMESTAMP '2004-10-19 10:23:54+02'` is a timestamp without time zone and the offset is silently dropped. Write `TIMESTAMP WITH TIME ZONE '...'` or `'...'::timestamptz`. https://www.postgresql.org/docs/18/datatype-datetime.html
- Use ISO 8601 input (`1999-01-08`). `01/02/03` depends on `DateStyle`. https://www.postgresql.org/docs/18/datatype-datetime.html
- `'now'`, `'today'`, `'tomorrow'` are frozen when the literal is read: in a DEFAULT, a view or a function they go stale. Use `now()`, `CURRENT_TIMESTAMP`, `CURRENT_DATE + 1`. https://www.postgresql.org/docs/18/datatype-datetime.html https://www.postgresql.org/docs/18/functions-datetime.html
- `'infinity'` and `'-infinity'` are valid dates, timestamps and intervals. Test with `isfinite()`. https://www.postgresql.org/docs/18/datatype-datetime.html

## Time zones

- Use full IANA names (`America/New_York`), which follow DST. An abbreviation is a fixed offset: `2014-06-04 12:00 EST` is UTC-5 even in summer. https://www.postgresql.org/docs/18/datatype-datetime.html
- POSIX-style zones have the opposite sign of ISO 8601: zone `'+05'` means UTC-5. https://www.postgresql.org/docs/18/functions-datetime.html
- `timestamp AT TIME ZONE z` returns timestamptz (reads the value as local time in z). `timestamptz AT TIME ZONE z` returns timestamp (the wall-clock time in z). `AT LOCAL` uses the session zone. https://www.postgresql.org/docs/18/functions-datetime.html
- Comparing `timestamp` with `timestamptz` reads the timestamp in the session `TimeZone`. https://www.postgresql.org/docs/18/functions-datetime.html
- PostgreSQL 18 checks the session zone's own abbreviations before `timezone_abbreviations`. https://www.postgresql.org/docs/18/release-18.html

## Intervals and arithmetic

- An interval keeps months, days and microseconds apart. `+ interval '1 day'` keeps the local time across a DST change, `+ interval '24 hours'` does not. https://www.postgresql.org/docs/18/functions-datetime.html
- `date_add(ts, interval, 'Europe/Paris')` applies DST rules of a named zone instead of the session zone. https://www.postgresql.org/docs/18/functions-datetime.html
- Month arithmetic clamps: March 31 + 1 month is April 30. https://www.postgresql.org/docs/18/functions-datetime.html
- `timestamp - timestamp` gives days and time, never months. `age()` gives years, months and days. For elapsed seconds subtract `EXTRACT(EPOCH FROM ...)` values. https://www.postgresql.org/docs/18/functions-datetime.html
- `INTERVAL '1'` means 1 second. With a field qualifier, lower fields are silently dropped (`'1 day 2:03:04' HOUR TO MINUTE` loses seconds). https://www.postgresql.org/docs/18/datatype-datetime.html
- Attach an explicit sign to each field if any field of an interval is negative (`'-1 2:03:04'` makes only the days negative by default). https://www.postgresql.org/docs/18/datatype-datetime.html
- `OVERLAPS` treats periods as half-open: periods sharing only an endpoint do not overlap. https://www.postgresql.org/docs/18/functions-datetime.html

## Current time

- `now()`, `CURRENT_TIMESTAMP` and `CURRENT_DATE` return the start of the transaction and do not move within it. `statement_timestamp()` is the start of the current statement. `clock_timestamp()` is the real time and changes within a statement. https://www.postgresql.org/docs/18/functions-datetime.html
- Do not hold locks while calling `pg_sleep()`: other sessions wait on the sleeper. https://www.postgresql.org/docs/18/functions-datetime.html

## extract, date_trunc, date_bin

- Use `extract(field FROM x)`, which returns numeric, for computation and `to_char` for display. https://www.postgresql.org/docs/18/functions-datetime.html
- `dow` is Sunday 0 to Saturday 6. `isodow` is Monday 1 to Sunday 7. `to_char(x, 'D')` is Sunday 1, so it differs from `dow`. https://www.postgresql.org/docs/18/functions-datetime.html
- `week` is the ISO week: pair it with `isoyear`, not `year` (2012-12-31 is week 1 of 2013). https://www.postgresql.org/docs/18/functions-datetime.html
- `epoch` of a `timestamptz` is real seconds since 1970 UTC. For `timestamp` and `date` it ignores zones and DST. https://www.postgresql.org/docs/18/functions-datetime.html
- `EXTRACT(MINUTES FROM INTERVAL '80 minutes')` is 20 (the interval normalizes to 01:20:00). https://www.postgresql.org/docs/18/functions-datetime.html
- `date_trunc('day', tstz)` truncates to local midnight of the session zone. Pass the zone as third argument for another one: https://www.postgresql.org/docs/18/functions-datetime.html
  ```sql
  SELECT date_trunc('day', TIMESTAMP WITH TIME ZONE '2001-02-16 20:38:40+00', 'Australia/Sydney');
  ```
- `date_bin('15 minutes', ts, origin)` buckets to any stride without months: https://www.postgresql.org/docs/18/functions-datetime.html
  ```sql
  SELECT date_bin('15 minutes', TIMESTAMP '2020-02-11 15:44:17', TIMESTAMP '2001-01-01');
  ```

## Formatting and parsing

- Prefer a plain cast for standard formats. `to_timestamp` and `to_date` are for formats a cast cannot read. https://www.postgresql.org/docs/18/functions-formatting.html
- Template characters that are not patterns skip input characters without checking them. Without `FX`, separators and extra spaces are loose. https://www.postgresql.org/docs/18/functions-formatting.html
- `to_timestamp('12.3', 'SS.MS')` is 300 ms, not 3 ms. https://www.postgresql.org/docs/18/functions-formatting.html
- Day names and quarter (`DAY`, `D`, `Q`) are accepted in input but ignored. Do not mix ISO week fields (`IYYY`, `IW`, `ID`) with Gregorian ones. https://www.postgresql.org/docs/18/functions-formatting.html
- `to_char` pads `Day` and `Month` to 9 characters: write `FMDay`. `FM` applies to the next pattern only. https://www.postgresql.org/docs/18/functions-formatting.html
- ISO 8601 UTC output: https://www.postgresql.org/docs/18/functions-formatting.html
  ```sql
  to_char(current_timestamp AT TIME ZONE 'UTC', 'YYYY-MM-DD"T"HH24:MI:SS"Z"')
  ```
- Numeric templates reserve a sign column: `to_char(485, '999')` is `' 485'`. Use `FM999`. https://www.postgresql.org/docs/18/functions-formatting.html

## Old patterns

- `date_part()` (double precision) -> `extract()` (numeric). https://www.postgresql.org/docs/18/functions-datetime.html
- `'now'` or `TIMESTAMP 'now'` in DEFAULT -> `now()` or `CURRENT_TIMESTAMP`. https://www.postgresql.org/docs/18/functions-datetime.html
- `timeofday()` -> `clock_timestamp()`. https://www.postgresql.org/docs/18/functions-datetime.html
