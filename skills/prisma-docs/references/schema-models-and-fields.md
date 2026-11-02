# Schema: models, fields, defaults and indexes

## Contents
- Schema file
- Identifiers and uniqueness
- Field types and modifiers
- Defaults and id generators
- Naming and database mapping
- Indexes

## Schema file

- Canonical v7 blocks: `datasource db { provider = "postgresql" }` without `url`, and `generator client { provider = "prisma-client"  output = "./generated" }`. The datasource also accepts `relationMode`, `schemas` and `extensions`. https://www.prisma.io/docs/orm/v7/reference/prisma-schema-reference#datasource
- `//` comments are dropped. `///` and `/* */` attach to the next node and are readable by tools. https://www.prisma.io/docs/orm/v7/prisma-schema/overview
- `prisma format` has fixed rules: block attributes move to the end of the block, an empty line resets alignment. https://www.prisma.io/docs/orm/v7/prisma-schema/overview

## Identifiers and uniqueness

- Every model needs one of `@id`, `@@id`, `@unique`, `@@unique`. When a `@unique` is the only identifier, it must be required. https://www.prisma.io/docs/orm/v7/reference/prisma-schema-reference#unique
- `@id` cannot be optional or on a relation field. https://www.prisma.io/docs/orm/v7/reference/prisma-schema-reference#id
- `@@unique` fields must all be required: NULLs are distinct, so an optional field would let identical-looking rows in. `@unique` alone can be optional, and several NULL rows are allowed. https://www.prisma.io/docs/orm/v7/reference/prisma-schema-reference#unique-1 https://www.prisma.io/docs/orm/v7/reference/prisma-schema-reference#unique
- A compound key `@@id([firstName, lastName])` becomes the Client key `firstName_lastName`. Rename it with `name:`. `name` changes the Client API, `map` changes the database constraint name. https://www.prisma.io/docs/orm/v7/prisma-schema/data-model/models https://www.prisma.io/docs/orm/v7/prisma-schema/data-model/database-mapping
  ```ts
  prisma.user.findUnique({ where: { firstName_lastName: { firstName: "Alice", lastName: "Smith" } } });
  ```
  https://www.prisma.io/docs/orm/v7/reference/prisma-schema-reference#id-1
- Compound keys are accepted in the `where` of `findUnique`, `findUniqueOrThrow`, `update`, `delete`, `upsert`, and in `connect` and `connectOrCreate`. They are filter options only for unique queries. https://www.prisma.io/docs/orm/v7/prisma-client/special-fields-and-types/working-with-composite-ids-and-constraints

## Field types and modifiers

- `?` and `[]` cannot combine: optional lists do not exist. https://www.prisma.io/docs/orm/v7/reference/prisma-schema-reference#-modifier https://www.prisma.io/docs/orm/v7/reference/prisma-schema-reference#-modifier-1
- Native types (`@db.VarChar(200)`) refine the default mapping. `db pull` writes them only when they differ from the default. https://www.prisma.io/docs/orm/v7/prisma-client/type-safety/prisma-type-system
- `Decimal` is decimal.js in JS. `BigInt` is JS `bigint`. `DateTime` must be passed as a `Date`, a string gives a runtime error. https://www.prisma.io/docs/orm/v7/reference/prisma-schema-reference#datetime https://www.prisma.io/docs/orm/v7/reference/prisma-schema-reference#bigint https://www.prisma.io/docs/orm/v7/reference/prisma-schema-reference#decimal
- `Unsupported("circle")` fields exist in migrations but not in the Client. A model with a required `Unsupported` field gets no `create`, `update` or `upsert`. https://www.prisma.io/docs/orm/v7/reference/prisma-schema-reference#unsupported
- `@ignore`/`@@ignore` hide a field or model from the Client. An ignored required field without default disables `create`. https://www.prisma.io/docs/orm/v7/reference/prisma-schema-reference#ignore
- `@updatedAt` is implemented at the Prisma ORM level, not by the database, and an empty update clause (`data: {}`) leaves it unchanged. https://www.prisma.io/docs/orm/v7/reference/prisma-schema-reference#updatedat

## Defaults and id generators

- Database-level: `autoincrement()`, `now()` (`CURRENT_TIMESTAMP`), `dbgenerated("...")`, `sequence()` (CockroachDB). https://www.prisma.io/docs/orm/v7/reference/prisma-schema-reference#default
- Prisma-level (implemented by Prisma ORM, not visible in the database schema): `cuid()`, `cuid(2)`, `uuid()` (v4 by default), `uuid(7)`, `nanoid()` (21 characters, `nanoid(n)` with n from 2 to 255). https://www.prisma.io/docs/orm/v7/reference/prisma-schema-reference#default
- `ulid()` produces a 26-character string. https://www.prisma.io/docs/orm/v7/reference/prisma-schema-reference#ulid
- A `cuid()` has no fixed length: give the column at least 30 characters. https://www.prisma.io/docs/orm/v7/reference/prisma-schema-reference#cuid
- Default for ids that rows inserted outside Prisma must also get: `@default(dbgenerated("gen_random_uuid()")) @db.Uuid`. A cast wraps the whole expression: `dbgenerated("(gen_random_uuid()::TEXT)")`. https://www.prisma.io/docs/orm/v7/prisma-schema/data-model/unsupported-database-features https://www.prisma.io/docs/orm/v7/prisma-migrate/workflows/native-database-functions
- When `dbgenerated` text differs from what the database reports (`'hello'::STRING`), Migrate keeps proposing a migration: run `db pull` to copy the exact form. https://www.prisma.io/docs/orm/v7/reference/prisma-schema-reference#dbgenerated
- `Json` defaults are quoted strings: `@default("[]")`, `@default("{ \"hello\": \"world\" }")`. https://www.prisma.io/docs/orm/v7/reference/prisma-schema-reference#default
- `@default` is not allowed on a relation field, only on its scalar. https://www.prisma.io/docs/orm/v7/reference/prisma-schema-reference#default

## Naming and database mapping

- Model names are typically singular PascalCase, field names typically camelCase. Keep plural or snake_case database names with `@@map("comments")` and `@map("comment_text")`. https://www.prisma.io/docs/orm/v7/reference/prisma-schema-reference#model https://www.prisma.io/docs/orm/v7/prisma-schema/data-model/database-mapping
- Enum `@map` is database-only: `PENDING @map("pending")` generates `Status.PENDING === "PENDING"`, and queries use the schema name. https://www.prisma.io/docs/orm/v7/prisma-schema/data-model/database-mapping
- Default constraint names derive from table and column names (not model names): `{table}_pkey`, `{table}_{cols}_key`, `{table}_{cols}_idx`, `{table}_{cols}_fkey`, trimmed to the database limit. Set your own with `map:` on `@id`, `@unique`, `@@index`, `@relation`. https://www.prisma.io/docs/orm/v7/prisma-schema/data-model/database-mapping
- Use an enum for a finite, type-safe set of values, a `String` for values that change often or are user-generated (avoids schema changes). https://www.prisma.io/docs/orm/v7/more/best-practices

## Indexes

- Arguments: `length` (MySQL), `sort` (on `@@index`/`@unique` everywhere, on ids only SQL Server), `type` (PostgreSQL: `Hash`, `Gist`, `Gin`, `SpGist`, `Brin`, default BTree), `clustered` (SQL Server), `map`. `length` and `sort` go on the field: `@@index([title(sort: Asc), author(sort: Desc)])`. https://www.prisma.io/docs/orm/v7/prisma-schema/data-model/indexes
- GIN with operator class: `@@index([value(ops: JsonbPathOps)], type: Gin)`. A Hash index serves only `=` and `<>`. https://www.prisma.io/docs/orm/v7/prisma-schema/data-model/indexes
- PostgreSQL cannot sort a unique constraint: create a descending unique index in SQL. https://www.prisma.io/docs/orm/v7/prisma-schema/data-model/indexes
- Expression indexes (`lower(title)`) and `CREATE INDEX CONCURRENTLY` cannot be configured in the schema (PostgreSQL, CockroachDB), but can be configured at database level. https://www.prisma.io/docs/orm/v7/reference/prisma-schema-reference#index
- Indexes defined with a function such as `to_tsvector` are not visible with `prisma db pull`. https://www.prisma.io/docs/orm/v7/prisma-schema/data-model/indexes
- An unsupported database feature is added by customizing a migration: generate it with `--create-only`, edit the SQL, then apply. https://www.prisma.io/docs/orm/v7/prisma-migrate/workflows/unsupported-database-features
- Partial indexes need `previewFeatures = ["partialIndexes"]`. `where` works on `@unique`, `@@unique`, `@@index`, as raw SQL or a type-checked object. PostgreSQL, SQLite, SQL Server, CockroachDB, not MySQL. On CockroachDB Migrate does not detect predicate changes. https://www.prisma.io/docs/orm/v7/prisma-schema/data-model/indexes
  ```prisma
  @@unique([email], where: raw("status = 'active'"))
  @@index([title], where: { published: true })
  ```
  https://www.prisma.io/docs/orm/v7/prisma-schema/data-model/indexes
- `@@fulltext` (MySQL and MongoDB) needs the `fullTextIndex` preview. https://www.prisma.io/docs/orm/v7/prisma-schema/data-model/indexes
- Index the fields used in `where` and `orderBy`. On databases that do not enforce foreign keys (like PlanetScale), where Prisma emulates relations, add `@@index` on the relation scalar fields yourself. https://www.prisma.io/docs/orm/v7/more/best-practices
