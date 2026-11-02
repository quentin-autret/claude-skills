# Schema: multi-schema, views, external tables, introspection

## Contents
- Choosing the source of truth
- Introspection with db pull
- Multiple database schemas
- Views
- Externally managed tables
- Features without schema syntax
- Table inheritance

## Choosing the source of truth

- Default: the Prisma schema is the truth, edit models then run `prisma migrate dev`. Escape hatch for SQL-first teams: change the database in SQL, run `prisma db pull`, then `prisma generate`. https://www.prisma.io/docs/orm/v7/core-concepts/data-modeling https://www.prisma.io/docs/orm/v7/prisma-schema/introspection

## Introspection with db pull

- `db pull` reads the URL from the config and overwrites `schema.prisma`: commit first, or use `--print`. `--force` ignores existing customisations. https://www.prisma.io/docs/cli/v7/db/pull
- On a relational database it merges: model and enum order, comments, `@map`/`@@map`, `@updatedAt`, `cuid()`/`uuid()` defaults and custom relation names survive. Field order comes from the database. https://www.prisma.io/docs/orm/v7/prisma-schema/introspection
- Relations are detected only from real foreign keys. A key with UNIQUE becomes one-to-one, a plain key one-to-many. Two keys between the same tables produce relation names like `Post_authorToUser` and verbose field names like `User_Post_authorToUser`: rename the relation fields. https://www.prisma.io/docs/orm/v7/prisma-schema/introspection
- Identifiers must match `[A-Za-z][A-Za-z0-9_]*`. Invalid characters are dropped or replaced by `_`, with `@map` keeping the original. Two tables that sanitise to the same name break `prisma generate` until you rename one. https://www.prisma.io/docs/orm/v7/prisma-schema/introspection
- Tables without a unique identifier get `@@ignore`. There is no partial introspection: use a database user that sees only the wanted tables, or `@@ignore` models. https://www.prisma.io/docs/orm/v7/prisma-schema/introspection https://www.prisma.io/docs/orm/v7/reference/prisma-schema-reference#ignore
- After introspection, rename plural snake_case models and relation fields by hand (`users` to `User` with `@@map("users")`). https://www.prisma.io/docs/orm/v7/prisma-client/setup-and-configuration/custom-model-and-field-names

## Multiple database schemas

- PostgreSQL, CockroachDB and SQL Server: `schemas = ["base", "shop"]` in the datasource, then `@@schema("base")` on each model and enum to place it in a database schema. https://www.prisma.io/docs/orm/v7/prisma-schema/data-model/multi-schema
- The `schema` value of the connection string is the default database schema for raw queries; other queries use the schema of the model or enum queried. The same table name in 2 schemas needs distinct model names with `@@map`. Moving a model to another schema drops and recreates the table. https://www.prisma.io/docs/orm/v7/prisma-schema/data-model/multi-schema

## Views

- Preview `views`: `view UserInfo { id Int  email String }`. Migrate and `db push` never create views: write `CREATE VIEW` in a `--create-only` migration. https://www.prisma.io/docs/orm/v7/prisma-schema/data-model/views
- Views are read-only in the Client (no create, update, delete, upsert). No `@id` or indexes. `@unique` is not enforced and only enables `findUnique`, relations and cursors. Materialized views are not supported: create one in the database with SQL and refresh it manually, for example with TypedSQL. https://www.prisma.io/docs/orm/v7/prisma-schema/data-model/views
- `db pull` writes view SQL under `prisma/views/<schema>/<View>.sql` (PostgreSQL, MySQL, SQL Server, CockroachDB). https://www.prisma.io/docs/orm/v7/prisma-schema/data-model/views

## Externally managed tables

- Tables owned by another system stay queryable while Migrate ignores them: https://www.prisma.io/docs/orm/v7/prisma-schema/data-model/externally-managed-tables
  ```ts
  export default defineConfig({
    schema: "prisma/schema.prisma",
    datasource: { url: env("DATABASE_URL") },
    migrations: { path: "prisma/migrations", initShadowDb: `CREATE TABLE public.users (id SERIAL PRIMARY KEY);` },
    experimental: { externalTables: true },
    tables: { external: ["public.users"] },
    enums: { external: ["public.role"] },
  });
  ```
  https://www.prisma.io/docs/orm/v7/prisma-schema/data-model/externally-managed-tables
- PostgreSQL and SQL Server need `schema.table` names, MySQL and SQLite plain names (errors `P3023` and `P3024`). Without `experimental.externalTables: true` the config fails to load. https://www.prisma.io/docs/orm/v7/reference/error-reference https://www.prisma.io/docs/orm/v7/reference/prisma-config-reference
- `initShadowDb` gives the shadow database a stub of the external table (primary keys required) when a managed table references it. https://www.prisma.io/docs/orm/v7/prisma-schema/data-model/externally-managed-tables
- Prisma does not check that the model matches the table. When the owner changes it, run `db pull` then `generate`. https://www.prisma.io/docs/orm/v7/prisma-schema/data-model/externally-managed-tables

## Features without schema syntax

- CHECK and EXCLUDE constraints, expression indexes, INCLUDE indexes, row-level security, partitioned tables, deferred constraints, comments, NULLS FIRST/LAST have no schema syntax: write them in a `--create-only` migration. `db pull` warns about most of them (all except INCLUDE indexes) and comments the affected models and fields. https://www.prisma.io/docs/orm/v7/prisma-schema/data-model/unsupported-database-features https://www.prisma.io/docs/orm/v7/reference/database-features https://www.prisma.io/docs/orm/v7/prisma-schema/introspection
- CHECK constraints are enforced by the database only. When several fail, only the first in alphabetical order is reported. https://www.prisma.io/docs/orm/v7/more/troubleshooting/check-constraints
- PostgreSQL extensions are installed by a migration, never by hand, because the shadow database also needs them: `migrate dev --create-only`, add `CREATE EXTENSION IF NOT EXISTS citext;`, then apply. Extension types use `Unsupported(...)`. https://www.prisma.io/docs/orm/v7/prisma-schema/postgresql-extensions https://www.prisma.io/docs/orm/v7/prisma-migrate/workflows/native-database-functions
- Alternative: the `postgresqlExtensions` preview with `extensions = [pgcrypto]` in the datasource lets Migrate apply them. https://www.prisma.io/docs/orm/v7/prisma-migrate/workflows/native-database-functions

## Table inheritance

- No union types in the schema. Single-table: one model with a `type` enum and optional subtype fields. Multi-table: a parent model with optional one-to-one relations to child models holding `activityId Int @unique`. https://www.prisma.io/docs/orm/v7/prisma-schema/data-model/table-inheritance
- Single-table costs wide rows and many NULLs. Multi-table costs joins on read and gives two ids (parent and child), so business code must pick one. Client extensions can wrap either for a cleaner API. https://www.prisma.io/docs/orm/v7/prisma-schema/data-model/table-inheritance
