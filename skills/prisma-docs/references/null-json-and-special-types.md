# null, undefined, Json and special field types

## Contents
- null versus undefined
- Json fields
- Scalar lists
- Decimal, BigInt, Bytes, DateTime

## null versus undefined

- `null` is a value, `undefined` means "no filter, no change". `deleteMany({ where: { id: undefined } })` matches every row, and `findFirst({ where: { name: undefined } })` returns the first row. https://www.prisma.io/docs/orm/v7/prisma-client/special-fields-and-types/null-and-undefined
- Default: enable the `strictUndefinedChecks` preview, which makes an explicit `undefined` throw, and skip fields on purpose with `Prisma.skip`. Pair it with the TypeScript option `exactOptionalPropertyTypes`. https://www.prisma.io/docs/orm/v7/prisma-client/special-fields-and-types/null-and-undefined
  ```ts
  data: { email: optionalEmail ?? Prisma.skip }
  ```
  https://www.prisma.io/docs/orm/v7/prisma-client/special-fields-and-types/null-and-undefined
- With zero conditions, `OR: []` (or an `OR` whose filter values are all `undefined`) returns nothing, while `AND` and `NOT` return everything. Optional form filters therefore empty an `OR` and widen an `AND`. https://www.prisma.io/docs/orm/v7/prisma-client/special-fields-and-types/null-and-undefined
- API input: a `null` from a caller sets a nullable column to NULL and fails on a required one. Map "absent" to `undefined`: `args.x != null ? args.x : undefined`. https://www.prisma.io/docs/orm/v7/prisma-client/special-fields-and-types/null-and-undefined

## Json fields

- Read values are `JsonValue`: narrow with `typeof`/`Array.isArray`, then cast to `Prisma.JsonObject` or `Prisma.JsonArray`. https://www.prisma.io/docs/orm/v7/prisma-client/special-fields-and-types/working-with-json-fields
- Database NULL and JSON `null` differ. Write with `Prisma.DbNull` or `Prisma.JsonNull`, filter with `equals: Prisma.AnyNull` (no shorthand). https://www.prisma.io/docs/orm/v7/prisma-client/special-fields-and-types/working-with-json-fields
- Path filters exist on PostgreSQL and MySQL only, with different syntaxes: https://www.prisma.io/docs/orm/v7/reference/prisma-client-reference#json-filters
  ```ts
  // PostgreSQL
  path: ["a", "b"]
  // MySQL
  path: "$.a.b"
  ```
  https://www.prisma.io/docs/orm/v7/prisma-client/special-fields-and-types/working-with-json-fields
- On PostgreSQL, `array_contains` takes an array even for one value, and filtering on object key values inside arrays is not supported (MySQL only). https://www.prisma.io/docs/orm/v7/prisma-client/special-fields-and-types/working-with-json-fields
- Json string filters (`string_contains` and the like) are case sensitive by default, and `mode: "insensitive"` makes them case insensitive. The Json filters remarks, however, say filtering is case sensitive in PostgreSQL and does not support `mode` there, so test it. https://www.prisma.io/docs/orm/v7/reference/prisma-client-reference#mode-1 https://www.prisma.io/docs/orm/v7/reference/prisma-client-reference#json-filters https://www.prisma.io/docs/orm/v7/prisma-client/special-fields-and-types/working-with-json-fields
- Passing a JSON string as the `array_contains` value instead of an object gets escaped and matches nothing. You cannot select a subset of keys, filter on key presence or sort by a JSON property. https://www.prisma.io/docs/orm/v7/prisma-client/special-fields-and-types/working-with-json-fields
- Typed Json needs the third-party `prisma-json-types-generator` (second generator block, `/// [LogMetaType]` above the field). https://www.prisma.io/docs/orm/v7/prisma-client/special-fields-and-types/working-with-json-fields

## Scalar lists

- Scalar lists exist only where the database supports them. `push` and the list filters are available for PostgreSQL, CockroachDB and MongoDB. Write with `set`, append with `push`, filter with `has`, `hasEvery`, `hasSome`, `isEmpty`. https://www.prisma.io/docs/orm/v7/prisma-client/special-fields-and-types/working-with-scalar-lists-arrays
- NULL lists are matched by neither `NOT { has }` nor `isEmpty`, and `{ equals: null }` errors. Default: give list fields `@default([])`. https://www.prisma.io/docs/orm/v7/prisma-client/special-fields-and-types/working-with-scalar-lists-arrays https://www.prisma.io/docs/orm/v7/reference/prisma-client-reference#scalar-list-filters
- `equals` on a list matches the given value exactly (the reference example matches lists holding only those values). https://www.prisma.io/docs/orm/v7/reference/prisma-client-reference#equals-1

## Decimal, BigInt, Bytes, DateTime

- `Decimal` values are decimal.js: `new Prisma.Decimal(24.454545).plus(1)`. https://www.prisma.io/docs/orm/v7/prisma-client/special-fields-and-types
- `BigInt` columns return `bigint`, and `JSON.stringify` throws on them: use a replacer that converts to string. `P2033` means a number does not fit 64-bit: use `BigInt`. https://www.prisma.io/docs/orm/v7/prisma-client/special-fields-and-types https://www.prisma.io/docs/orm/v7/reference/error-reference
- `Bytes` is `Uint8Array`. https://www.prisma.io/docs/orm/v7/prisma-client/special-fields-and-types
- Pass `DateTime` values as `Date` objects. A string causes a runtime error. https://www.prisma.io/docs/orm/v7/prisma-client/special-fields-and-types
