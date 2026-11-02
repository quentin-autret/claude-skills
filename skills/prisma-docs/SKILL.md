---
name: prisma-docs
description: Pitfalls, Prisma ORM 7 changes and recommended patterns from the official Prisma documentation, each backed by its prisma.io/docs URL. Use when writing, reviewing or upgrading code that uses Prisma ORM 7, including migration from Prisma ORM 6. Also use for any task touching schema.prisma, prisma.config.ts, the prisma-client generator, Prisma Client, driver adapters (@prisma/adapter-pg and others), connection pools, PgBouncer, models, relations, indexes, db pull, queries (findMany, findUnique, upsert, createMany, select, include, omit, nested writes, groupBy, distinct, pagination), $transaction, null and undefined, Json fields, client extensions ($extends), TypedSQL, $queryRaw, error codes (P2002, P2025), logging, tracing, testing, Prisma Migrate (migrate dev, migrate deploy, db push, migrate reset, migrate resolve, migrate diff, shadow database, baselining, seeding), the Prisma CLI, and deployment to Docker, serverless, edge, Bun, Deno, NestJS or Next.js.
metadata:
  version: "Prisma ORM 7 (docs /orm/v7)"
  source: "https://www.prisma.io/docs"
  read-date: "2026-11-02"
---

# Prisma ORM 7 documentation

Rules distilled from https://www.prisma.io/docs that a model is likely to get wrong: Prisma ORM 7 changes, surprising defaults and documented gotchas. Generic ORM and TypeScript knowledge is left out. Every rule ends with the docs URL that backs it.

## Version covered

The references follow the Prisma ORM 7 docs tree (`/docs/orm/v7`, `/docs/cli/v7`, `/docs/guides/v7`). The site default is Prisma ORM 8, a separate rewrite that this skill does not cover. Check the `prisma` and `@prisma/client` versions in the project's `package.json` first. On 6.x, read [references/upgrade-v6-to-v7.md](references/upgrade-v6-to-v7.md) before applying any rule. On 8.x, use the Prisma 8 docs instead.

Some v7 pages still carry v6 code (`new PrismaClient()` without adapter, imports from `@prisma/client`, `url` in the schema datasource). The references flag these and give the v7 form.

## How to navigate

Read only the reference the task needs, from the table at the end. References over 100 lines open with a contents list. Follow the docs URL when the full page is needed. The pages read are listed in [SOURCES.md](SOURCES.md).

## Quick start: the pitfalls that bite most

1. `prisma@latest` installs the Prisma 8 CLI. Pin `prisma` and `@prisma/client` to the same 7.x version (the docs use `prisma@prev` and `@prisma/client@7`). https://www.prisma.io/docs/v7/getting-started
2. The generator is `provider = "prisma-client"` with a required `output`. Import `PrismaClient` and `Prisma` from `<output>/client`, never from `@prisma/client`, and never from `internal/*`. https://www.prisma.io/docs/orm/v7/prisma-schema/overview/generators
3. `new PrismaClient()` needs a driver adapter (or `accelerateUrl`). The connection URL lives in `prisma.config.ts`, yet the app passes its own connection string to the adapter in application code. https://www.prisma.io/docs/orm/v7/core-concepts/supported-databases/database-drivers
4. Prisma ORM 7 no longer loads `.env` by default, neither for the CLI nor for the client, so a missing variable is the usual cause of "nothing connects" after the upgrade. Load it explicitly: `import "dotenv/config"` in `prisma.config.ts` and in the app entry point (Bun excepted). https://www.prisma.io/docs/orm/v6/more/upgrades/to-v7
5. `migrate dev` and `db push` do not run `prisma generate`, and `migrate dev` and `migrate reset` do not seed. Run `prisma generate` and `prisma db seed` explicitly, with `"postinstall": "prisma generate"` for builds. https://www.prisma.io/docs/cli/v7/migrate/dev https://www.prisma.io/docs/cli/v7/migrate/reset https://www.prisma.io/docs/cli/v7/db/push
6. `env("DATABASE_URL")` in the config throws when the variable is missing, and every CLI command loads the config, so `prisma generate` fails in CI without a database URL. Use `process.env.DATABASE_URL` there. https://www.prisma.io/docs/orm/v7/reference/prisma-config-reference
7. Pool settings belong to the adapter, not the URL. The `pg` adapter has no connect timeout by default (`connectionTimeoutMillis` is 0). Setting it adds one, the docs example uses `5_000` to match the former 5 s. https://www.prisma.io/docs/orm/v7/prisma-client/setup-and-configuration/databases-connections/connection-pool
8. SSL validation is strict with `pg`, and `P1010: User was denied access` usually means an unverified certificate. Configure the CA rather than turning verification off. https://www.prisma.io/docs/orm/v6/more/upgrades/to-v7
9. `undefined` in `where` means "no filter": `deleteMany({ where: { id: undefined } })` deletes every row. Validate the input, then enable the `strictUndefinedChecks` preview (`previewFeatures = ["strictUndefinedChecks"]` in the generator, then `prisma generate`), which makes an explicit `undefined` throw, and skip a field on purpose with `Prisma.skip` (`email: optionalEmail ?? Prisma.skip`). https://www.prisma.io/docs/orm/v7/prisma-client/special-fields-and-types/null-and-undefined
10. `not`, `in` and `notIn` never match NULL rows. Add `{ field: null }` in an `OR` to include them. https://www.prisma.io/docs/orm/v7/reference/prisma-client-reference#not
11. Inside `$transaction(async (tx) => ...)` every query goes through `tx`, the transaction stays short, and `Promise.all` does not parallelise. https://www.prisma.io/docs/orm/v7/prisma-client/queries/transactions
12. Concurrent `upsert` or `connectOrCreate` can throw `P2002`: catch it and retry. https://www.prisma.io/docs/orm/v7/reference/prisma-client-reference#unique-key-constraint-errors-on-upserts
13. `migrate deploy` is the only production command. It detects no drift and generates nothing. Migrations need a direct connection, since PgBouncer breaks them. https://www.prisma.io/docs/cli/v7/migrate/deploy https://www.prisma.io/docs/orm/v7/prisma-client/setup-and-configuration/databases-connections/pgbouncer
14. A default rename in a migration is DROP plus ADD. Create with `migrate dev --create-only` and edit the SQL to `RENAME COLUMN`. https://www.prisma.io/docs/orm/v7/prisma-migrate/workflows/customizing-migrations
15. One `PrismaClient` per process: each instance owns a pool. In serverless, create it outside the handler and never `$disconnect()` per request. Cloudflare Workers are the exception (a client per request). https://www.prisma.io/docs/orm/v7/prisma-client/setup-and-configuration/databases-connections https://www.prisma.io/docs/guides/v7/deployment/cloudflare-workers
16. Prisma ORM 7 does not support MongoDB: stay on 6.19. https://www.prisma.io/docs/orm/v7/core-concepts/supported-databases/mongodb

Example for rules 2 to 4, from the docs:

```ts
// prisma.config.ts
import "dotenv/config";
import { defineConfig, env } from "prisma/config";
export default defineConfig({
  schema: "prisma/schema.prisma",
  migrations: { path: "prisma/migrations", seed: "tsx prisma/seed.ts" },
  datasource: { url: env("DATABASE_URL") },
});

// src/db.ts
import "dotenv/config";
import { PrismaPg } from "@prisma/adapter-pg";
import { PrismaClient } from "./generated/prisma/client";
const adapter = new PrismaPg({ connectionString: process.env.DATABASE_URL });
export const prisma = new PrismaClient({ adapter });
```

https://www.prisma.io/docs/orm/v7/reference/prisma-config-reference https://www.prisma.io/docs/v7/prisma-orm/quickstart/postgresql

## References

| Reference | Read it when |
| --- | --- |
| [references/upgrade-v6-to-v7.md](references/upgrade-v6-to-v7.md) | Upgrading from Prisma ORM 6, or code behaves differently after the upgrade, with the old-to-new mapping |
| [references/setup-and-config.md](references/setup-and-config.md) | Installing, `prisma.config.ts`, env loading, generator options, generated files, preview features |
| [references/connections-and-pooling.md](references/connections-and-pooling.md) | Driver adapters, client lifecycle, pool settings, serverless limits, PgBouncer, read replicas |
| [references/databases.md](references/databases.md) | PostgreSQL, Neon, Supabase, CockroachDB, MySQL, PlanetScale, SQLite, D1, SQL Server specifics |
| [references/schema-models-and-fields.md](references/schema-models-and-fields.md) | Ids, unique constraints, field types, defaults, `@map`, constraint names, indexes |
| [references/schema-relations.md](references/schema-relations.md) | Relations, many-to-many, self-relations, referential actions, `relationMode` |
| [references/schema-advanced-and-introspection.md](references/schema-advanced-and-introspection.md) | `db pull`, multi-schema, views, external tables, unsupported features, inheritance |
| [references/client-queries.md](references/client-queries.md) | CRUD, bulk operations, `select`/`include`/`omit`, nested writes, filters, pagination, `groupBy`, performance |
| [references/transactions-and-concurrency.md](references/transactions-and-concurrency.md) | `$transaction`, isolation levels, retries, upsert races, optimistic concurrency |
| [references/null-json-and-special-types.md](references/null-json-and-special-types.md) | `null` versus `undefined`, Json fields, scalar lists, Decimal, BigInt, DateTime |
| [references/extensions-and-types.md](references/extensions-and-types.md) | `$extends` components, shared extensions, generated types, TypeScript performance |
| [references/raw-sql.md](references/raw-sql.md) | TypedSQL, `$queryRaw`, `$executeRaw`, injection rules, `Unsupported` fields, SafeQL |
| [references/errors-logging-tracing.md](references/errors-logging-tracing.md) | Error classes and codes, error format, logging, OpenTelemetry, SQL comments |
| [references/testing.md](references/testing.md) | Mocking the client, integration tests on a real database, per-PR databases |
| [references/migrate-workflows.md](references/migrate-workflows.md) | Choosing `migrate dev`, `deploy`, `db push`, `reset`, history rules, drift, shadow database |
| [references/migrate-recipes.md](references/migrate-recipes.md) | Custom SQL, expand and contract, baselining, hotfixes, down migrations, squashing, seeding |
| [references/cli.md](references/cli.md) | Prisma CLI commands, flags, `migrate diff` patterns, `prisma dev`, Studio, CLI env variables |
| [references/deployment.md](references/deployment.md) | Build and `postinstall`, production migrations, Docker, Vercel, AWS, monorepos, several databases |
| [references/edge-runtimes-and-frameworks.md](references/edge-runtimes-and-frameworks.md) | Edge drivers, Cloudflare Workers, Bun, Deno, NestJS, Next.js |
