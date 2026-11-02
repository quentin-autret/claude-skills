# Data types (numeric, text, binary, enum, UUID, arrays, ranges)

## Contents
- Numbers and money
- Text and binary
- Enums
- UUID
- Arrays
- Ranges and multiranges

## Numbers and money

- `integer` is the common choice, with the best balance of range, storage and performance. Use `bigint` when the range of `integer` is insufficient (for example more than 2^31 identifiers over the life of a table). https://www.postgresql.org/docs/18/datatype-numeric.html
- `numeric` is exact but much slower than integer and float arithmetic. Unconstrained `numeric` keeps any scale. `numeric(p,s)` rounds to scale s and errors if the integer part overflows. Give precision and scale explicitly. https://www.postgresql.org/docs/18/datatype-numeric.html
- Rounding differs: `round(2.5::numeric)` is 3 (ties away from zero), `round(2.5::double precision)` is 2 (ties to even). https://www.postgresql.org/docs/18/datatype-numeric.html
- `numeric` accepts `'NaN'`, `'Infinity'`, `'-Infinity'`. NaN equals NaN and sorts above every number. Infinity fits only in an unconstrained numeric column. https://www.postgresql.org/docs/18/datatype-numeric.html
- `float` with no precision means `double precision`. Float equality may not behave as expected. https://www.postgresql.org/docs/18/datatype-numeric.html
- `money` depends on `lc_monetary`: a dump restored under another setting may fail. `money / integer` truncates toward zero. Cast to numeric, divide, cast back for rounding. https://www.postgresql.org/docs/18/datatype-money.html

## Text and binary

- Use `text`, or `varchar` without a length when no real limit exists. `char(n)` is usually the slowest and pads with spaces. https://www.postgresql.org/docs/18/datatype-character.html
- Inserting an over-length value into `varchar(n)` errors, but an explicit cast `'too long'::varchar(5)` truncates silently. https://www.postgresql.org/docs/18/datatype-character.html
- Trailing spaces of `char(n)` are ignored in comparisons and removed when converting to other string types, but are significant in `varchar`, `text`, LIKE and regex. https://www.postgresql.org/docs/18/datatype-character.html
- No character type can store NUL (code zero). Use `bytea` for raw bytes. https://www.postgresql.org/docs/18/datatype-character.html https://www.postgresql.org/docs/18/datatype-binary.html
- `bytea` supports a hex format `'\xDEADBEEF'` (the default `bytea_output`) and the historical escape format. Both are accepted on input. https://www.postgresql.org/docs/18/datatype-binary.html

## Enums

- `CREATE TYPE mood AS ENUM ('sad', 'ok', 'happy');` Values sort in creation order, so `current_mood > 'sad'` works. https://www.postgresql.org/docs/18/datatype-enum.html
- Values can be added or renamed, never removed or reordered without recreating the type. Use enums only for static sets. https://www.postgresql.org/docs/18/datatype-enum.html
- Labels are case sensitive. Two enum types cannot be compared without casting to text. https://www.postgresql.org/docs/18/datatype-enum.html

## UUID

- `gen_random_uuid()` and `uuidv4()` return random v4 UUIDs. PostgreSQL 18 adds `uuidv7()`: time-ordered (Unix milliseconds, then sub-millisecond time, then random bits). https://www.postgresql.org/docs/18/functions-uuid.html https://www.postgresql.org/docs/18/release-18.html
  ```sql
  id uuid PRIMARY KEY DEFAULT uuidv7()
  ```
- `uuidv7(interval)` shifts the embedded time. `uuid_extract_timestamp()` reads it back for v1 and v7 only (NULL otherwise), and it may differ from the real generation time. https://www.postgresql.org/docs/18/functions-uuid.html
- Other UUID versions come from the `uuid-ossp` module or the application. The `uuid` type stores any version. Output is always lowercase hyphenated. https://www.postgresql.org/docs/18/datatype-uuid.html

## Arrays

- Declared sizes (`integer[3]`) are ignored and enforce nothing. https://www.postgresql.org/docs/18/arrays.html
- Build arrays with `ARRAY[...]`, not the `'{...}'` string, to avoid quoting rules. https://www.postgresql.org/docs/18/arrays.html
- Subscripts start at 1 by default (other lower bounds can be declared). An out-of-range subscript returns NULL, not an error. https://www.postgresql.org/docs/18/arrays.html
- `ARRAY[1,2] || NULL` gives `{1,2}` (NULL read as an array). Use `array_append(arr, NULL)` to add a NULL element. https://www.postgresql.org/docs/18/arrays.html
- Search with `x = ANY (col)` or overlap `col && ARRAY[...]`. `@>` and `&&` can use a GIN index. https://www.postgresql.org/docs/18/arrays.html https://www.postgresql.org/docs/18/indexes-types.html
- Searching for specific elements often signals a missing child table: one row per element scales better. https://www.postgresql.org/docs/18/arrays.html
- `array_length(arr, 1)` is NULL for an empty array. `cardinality(arr)` returns 0 and counts all dimensions. https://www.postgresql.org/docs/18/functions-array.html
- `array_position`, `array_positions` and `array_remove` compare with IS NOT DISTINCT FROM, so they find and remove NULL. https://www.postgresql.org/docs/18/functions-array.html
- PostgreSQL 18 adds `array_sort(arr [, descending [, nulls_first]])` and `array_reverse()`. MIN and MAX accept arrays. https://www.postgresql.org/docs/18/functions-array.html https://www.postgresql.org/docs/18/release-18.html
- `unnest(a, b)` with several arrays works only in FROM and pads shorter arrays with NULL. https://www.postgresql.org/docs/18/functions-array.html

## Ranges and multiranges

- Built-in: `int4range`, `int8range`, `numrange`, `tsrange`, `tstzrange`, `daterange`, each with a multirange. Default bounds are `[)`. NULL as a bound means unbounded. Prefer the constructor: `tstzrange(a, b, '[)')`. https://www.postgresql.org/docs/18/rangetypes.html
- Discrete ranges (int, date) are normalized to `[)`. `[4,4)` becomes `empty`. https://www.postgresql.org/docs/18/rangetypes.html
- Index ranges with GiST (or SP-GiST) for `&&`, `@>`, `<@`. B-tree only helps equality. https://www.postgresql.org/docs/18/rangetypes.html
- Prevent overlaps with an exclusion constraint, not UNIQUE. Add scalar equality with `btree_gist`: https://www.postgresql.org/docs/18/rangetypes.html
  ```sql
  CREATE EXTENSION btree_gist;
  CREATE TABLE room_reservation (
      room text,
      during tsrange,
      EXCLUDE USING GIST (room WITH =, during WITH &&)
  );
  ```
- `range_agg` builds the union of ranges as a multirange. https://www.postgresql.org/docs/18/functions-aggregate.html
