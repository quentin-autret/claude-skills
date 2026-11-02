# Raw SQL: TypedSQL, $queryRaw and SafeQL

## Contents
- Which tool
- TypedSQL
- $queryRaw and $executeRaw
- SQL injection rules
- Unsupported fields and SafeQL

## Which tool

- Order of preference: the Client API, then TypedSQL (typed `.sql` files), then `$queryRaw`/`$executeRaw` tagged templates for dynamic queries, then the `Unsafe` variants, which carry a significant SQL injection risk. https://www.prisma.io/docs/orm/v7/prisma-client/using-raw-sql https://www.prisma.io/docs/orm/v7/prisma-client/using-raw-sql/raw-queries

## TypedSQL

- Enable `previewFeatures = ["typedSql"]`, put `.sql` files in `prisma/sql/` (file name must be a valid JS identifier, not starting with `$`), apply pending migrations, then run `prisma generate --sql`. https://www.prisma.io/docs/orm/v7/prisma-client/using-raw-sql/typedsql
  ```ts
  import { getUsersWithPosts } from "./generated/prisma/sql";
  const users = await prisma.$queryRawTyped(getUsersWithPosts());
  ```
  https://www.prisma.io/docs/orm/v7/prisma-client/using-raw-sql/typedsql
- Generation needs a live database, reached through `datasource.url` in the config. A custom folder goes in `typedSql: { path: "./prisma/sql" }`. https://www.prisma.io/docs/orm/v7/prisma-client/using-raw-sql/typedsql
- Placeholders: PostgreSQL `$1`, MySQL `?`, SQLite `$1`, `?` or `:name`. PostgreSQL arrays use `= ANY($1)`. https://www.prisma.io/docs/orm/v7/prisma-client/using-raw-sql/typedsql
- Inference works on PostgreSQL and MySQL 8+. SQLite and older MySQL need comments such as `-- @param {Int} $1:minAge`, with `?` for nullable. Array arguments cannot be typed manually. https://www.prisma.io/docs/orm/v7/prisma-client/using-raw-sql/typedsql
- Queries whose columns are chosen at runtime are not natively supported by TypedSQL: use `$queryRawUnsafe` or `$executeRawUnsafe`, which need care against SQL injection and give up TypedSQL type safety. https://www.prisma.io/docs/orm/v7/prisma-client/using-raw-sql/typedsql
- Every driver adapter is compatible except `@prisma/adapter-better-sqlite3`; for SQLite use `@prisma/adapter-libsql`. https://www.prisma.io/docs/orm/v7/prisma-client/using-raw-sql/typedsql

## $queryRaw and $executeRaw

- The tagged template becomes a prepared statement: `` prisma.$queryRaw`SELECT * FROM "User" WHERE email = ${email}` ``. `$queryRaw` accepts only a template or `Prisma.sql`. `$executeRaw` returns the affected row count. https://www.prisma.io/docs/orm/v7/prisma-client/using-raw-sql/raw-queries
- One statement per call. Prepared statements allow only a subset of SQL (`START TRANSACTION` is not permitted). PostgreSQL does not support `ALTER` in a prepared statement: the documented workaround is `$executeRawUnsafe`, which does not escape the value. https://www.prisma.io/docs/orm/v7/prisma-client/using-raw-sql/raw-queries
- Variables are values only: not inside string literals (pass the whole string, or `'..' || ${name}`), not identifiers, not keywords (`ORDER BY ${dir}` fails). https://www.prisma.io/docs/orm/v7/prisma-client/using-raw-sql/raw-queries
- Helpers: `Prisma.join(ids)` for `IN (...)`, `Prisma.empty` for an absent fragment (not `""`), `Prisma.sql` to compose. https://www.prisma.io/docs/orm/v7/prisma-client/using-raw-sql/raw-queries
- No implicit casts: `` SELECT LENGTH(${42}) `` fails on PostgreSQL, write `${42}::text`. JS integers reach PostgreSQL as INT8. https://www.prisma.io/docs/orm/v7/prisma-client/using-raw-sql/raw-queries
- `$queryRaw<User[]>` types are not checked against the database. Results are not normalised: Boolean is `1/0` on MySQL and `true/false` on PostgreSQL. 64-bit integers come back as `BigInt`, decimals as `Decimal`. https://www.prisma.io/docs/orm/v7/prisma-client/using-raw-sql/raw-queries
- `Unsupported` columns must be cast to a supported type in the query (`location::text`), if the column supports the cast. https://www.prisma.io/docs/orm/v7/prisma-client/using-raw-sql/raw-queries
- `prisma db execute --file x.sql` runs a script from the CLI without touching migration history and returns no data. https://www.prisma.io/docs/cli/v7/db/execute

## SQL injection rules

- Safe: a literal tagged template with `${}`, or `Prisma.sql`. Safe `Unsafe` use is parameterised: https://www.prisma.io/docs/orm/v7/prisma-client/using-raw-sql/raw-queries
  ```ts
  prisma.$queryRawUnsafe("SELECT * FROM User WHERE (name = $1 OR email = $2)", name, email);
  ```
  https://www.prisma.io/docs/orm/v7/prisma-client/using-raw-sql/raw-queries
- Vulnerable: building the string first (including faking a template from a strings array), any `Prisma.raw(userInput)`, concatenation into `$queryRawUnsafe`. https://www.prisma.io/docs/orm/v7/prisma-client/using-raw-sql/raw-queries
- With PostgreSQL `ILIKE`, put `%` wildcards into the bound value (`` `%${fragment}` ``), never in the SQL next to the placeholder (`%$2` does not work). https://www.prisma.io/docs/orm/v7/prisma-client/using-raw-sql/raw-queries

## Unsupported fields and SafeQL

- Raw queries remain the documented route for `Unsupported` fields such as PostGIS `Unsupported("geography(Point, 4326)")`. Wrap them in `model` extension methods so callers get typed functions. https://www.prisma.io/docs/orm/v7/prisma-client/using-raw-sql/safeql
- SafeQL is an ESLint plugin (`@ts-safeql/eslint-plugin` with `libpg-query`) that checks `$queryRaw`/`$executeRaw` against the real schema. Rule `@ts-safeql/check-sql` with `migrationsDir: "./prisma/migrations"` and `tag: "prisma.+($queryRaw|$executeRaw)"`. It needs a database URL at lint time. https://www.prisma.io/docs/orm/v7/prisma-client/using-raw-sql/safeql
- `Unsupported` fields are typed `any`. https://www.prisma.io/docs/orm/v7/prisma-client/using-raw-sql/safeql
