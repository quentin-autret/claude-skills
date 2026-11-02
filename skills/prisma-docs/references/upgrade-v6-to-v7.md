# Upgrading from Prisma ORM 6 to 7

## Contents
- Before you start
- Upgrade checklist
- Behaviour changes that surface after the upgrade
- Old patterns

## Before you start

- Upgrade both `prisma` (dev dependency) and `@prisma/client` to 7, on the same major. https://www.prisma.io/docs/orm/v6/more/upgrades/to-v7
- Prisma 8 publishes its CLI as the `latest` tag of `prisma`. The v7 docs install the CLI with the `prev` dist-tag (`prisma@prev`) and the client as `@prisma/client@7`. https://www.prisma.io/docs/v7/getting-started https://www.prisma.io/docs/cli/v7
- Minimum Node.js 20.19.0 (22.x recommended), TypeScript 5.4.0 (5.9.x recommended). A too-old TypeScript shows type errors in generated types and circular-type errors on `groupBy`. https://www.prisma.io/docs/orm/v6/more/upgrades/to-v7 https://www.prisma.io/docs/orm/v7/reference/system-requirements
- MongoDB is not supported by Prisma ORM 7: stay on Prisma ORM 6.19 for MongoDB projects. https://www.prisma.io/docs/orm/v7/core-concepts/supported-databases/mongodb

## Upgrade checklist

1. ESM: set `"type": "module"` in `package.json`. tsconfig: `module: "ESNext"`, `moduleResolution: "bundler"`, `target: "ES2023"`, `strict: true`, `esModuleInterop: true`. A CommonJS project sets `moduleFormat = "cjs"` in the generator instead. https://www.prisma.io/docs/orm/v6/more/upgrades/to-v7 https://www.prisma.io/docs/orm/v7/prisma-schema/overview/generators
2. Generator: switch to `provider = "prisma-client"` with a mandatory `output`, drop `engineType = "binary"`, and change every import to the generated path. `binaryTargets` is documented only for `prisma-client-js`. https://www.prisma.io/docs/orm/v6/more/upgrades/to-v7 https://www.prisma.io/docs/orm/v7/prisma-schema/overview/generators
   ```prisma
   generator client {
     provider = "prisma-client"
     output   = "../src/generated/prisma"
   }
   ```
   ```ts
   import { PrismaClient, Prisma } from "./generated/prisma/client";
   ```
   https://www.prisma.io/docs/orm/v7/prisma-schema/overview/generators
3. Datasource: `url`, `directUrl` and `shadowDatabaseUrl` are deprecated in the schema `datasource` block and configured in `prisma.config.ts` instead. If you used `directUrl` for migrations, that value becomes `datasource.url` in the config, because the CLI uses the `url` connection string for migrations. https://www.prisma.io/docs/orm/v6/more/upgrades/to-v7 https://www.prisma.io/docs/orm/v7/reference/prisma-config-reference
   ```ts
   import "dotenv/config";
   import { defineConfig, env } from "prisma/config";

   export default defineConfig({
     schema: "prisma/schema.prisma",
     migrations: { path: "prisma/migrations", seed: "tsx prisma/seed.ts" },
     datasource: { url: env("DATABASE_URL") },
   });
   ```
   https://www.prisma.io/docs/orm/v7/reference/prisma-config-reference
4. Env loading: Prisma 7 does not load `.env` by default. Install `dotenv` and import `dotenv/config` at the top of `prisma.config.ts`. The `prisma-client` generator does not load `.env` at runtime either, so use `dotenv` or set the variables yourself. Bun loads `.env` itself. https://www.prisma.io/docs/orm/v6/more/upgrades/to-v7 https://www.prisma.io/docs/orm/v7/prisma-schema/overview/generators
5. Client: construct with a driver adapter. https://www.prisma.io/docs/orm/v7/prisma-client/setup-and-configuration/introduction
   ```ts
   import { PrismaPg } from "@prisma/adapter-pg";
   const adapter = new PrismaPg({ connectionString: process.env.DATABASE_URL });
   export const prisma = new PrismaClient({ adapter });
   ```
   https://www.prisma.io/docs/orm/v7/reference/prisma-client-reference#adapter
6. Restore v6 pool timeouts on the adapter if needed, and handle strict SSL (next section).
7. Replace `$use` middleware and the Metrics preview feature with Client extensions (or driver adapter metrics). https://www.prisma.io/docs/orm/v6/more/upgrades/to-v7
8. Scripts and CI: add explicit `prisma generate` (build or `postinstall`) and `prisma db seed` steps, update scripts that use `db execute --url/--schema` or `migrate diff --from-url`, remove deleted environment variables. https://www.prisma.io/docs/orm/v6/more/upgrades/to-v7 https://www.prisma.io/docs/orm/v7/prisma-client/setup-and-configuration/generating-prisma-client

## Behaviour changes that surface after the upgrade

- Pool defaults come from the Node driver. With `pg`, `connectionTimeoutMillis` defaults to 0 (no timeout, v6 used 5 s) and `idleTimeoutMillis` is 10 s (v6 used 300 s). To match v6: https://www.prisma.io/docs/orm/v7/prisma-client/setup-and-configuration/databases-connections/connection-pool
  ```ts
  new PrismaPg({ connectionString: process.env.DATABASE_URL, connectionTimeoutMillis: 5_000, idleTimeoutMillis: 300_000 });
  ```
- SSL validation is strict (node-pg instead of the Rust engine). Symptom: `P1010: User was denied access on the database`. Fix: configure the CA (`NODE_EXTRA_CA_CERTS` or `node --use-openssl-ca`). Alternative: `ssl: { rejectUnauthorized: false }` on the adapter, which skips verification. https://www.prisma.io/docs/orm/v6/more/upgrades/to-v7
- `migrate dev` and `db push` no longer run `prisma generate`, and `migrate dev`/`migrate reset` no longer seed. https://www.prisma.io/docs/orm/v6/more/upgrades/to-v7 https://www.prisma.io/docs/cli/v7/migrate/dev https://www.prisma.io/docs/orm/v7/prisma-migrate/workflows/seeding
- `prisma generate` does not need a database URL but still fails if `env()` throws while the config loads, so `env("DATABASE_URL")` makes it fail in a CI job without a database URL (`PrismaConfigEnvError`). Use `process.env.DATABASE_URL` there. https://www.prisma.io/docs/orm/v7/reference/prisma-config-reference
- Generated TS enums still use schema names, not `@map` values. With SQLite, use `@prisma/adapter-better-sqlite3` `~1.0.7` or later for correct enum behaviour. https://www.prisma.io/docs/orm/v6/more/upgrades/to-v7
- SQLite with adapters stores `DateTime` as ISO 8601. A database written by the v6 native driver needs `{ timestampFormat: "unixepoch-ms" }` on the adapter. https://www.prisma.io/docs/orm/v7/core-concepts/supported-databases/sqlite
- An Accelerate URL (`prisma://`, `prisma+postgres://`) cannot go to `PrismaPg`. Use `new PrismaClient({ accelerateUrl: process.env.DATABASE_URL }).$extends(withAccelerate())` without an adapter. https://www.prisma.io/docs/orm/v6/more/upgrades/to-v7
- Some v7 pages show `new PrismaClient()` without an adapter. Follow the upgrade guide. https://www.prisma.io/docs/orm/v7/prisma-client/setup-and-configuration/generating-prisma-client

## Old patterns

| Old | Replacement |
| --- | --- |
| `provider = "prisma-client-js"`, client in `node_modules` | `provider = "prisma-client"` with `output`, import from `<output>/client` (https://www.prisma.io/docs/orm/v7/prisma-schema/overview/generators) |
| `import { PrismaClient } from "@prisma/client"` | `import { PrismaClient } from "./generated/prisma/client"` (https://www.prisma.io/docs/orm/v6/more/upgrades/to-v7) |
| `url`, `directUrl`, `shadowDatabaseUrl` in the schema `datasource` | `datasource.url` and `datasource.shadowDatabaseUrl` in `prisma.config.ts` (https://www.prisma.io/docs/orm/v7/reference/prisma-config-reference) |
| `new PrismaClient()`, `datasourceUrl`, `datasources` | `new PrismaClient({ adapter })` (https://www.prisma.io/docs/orm/v7/reference/prisma-client-reference#adapter) |
| URL params `connection_limit`, `pool_timeout`, `connect_timeout` | adapter options such as `max`, `connectionTimeoutMillis` (https://www.prisma.io/docs/orm/v7/prisma-client/setup-and-configuration/databases-connections/connection-pool) |
| Automatic `.env` loading | `import "dotenv/config"` (https://www.prisma.io/docs/orm/v6/more/upgrades/to-v7) |
| `prisma.$use(...)` middleware | `$extends({ query: ... })` (https://www.prisma.io/docs/orm/v6/more/upgrades/to-v7) |
| `Prisma.validator<...>()(...)` | TypeScript's native `satisfies` keyword (https://www.prisma.io/docs/orm/v7/prisma-schema/overview/generators) |
| Automatic seed after `migrate dev` or `migrate reset` | `migrations.seed` in config, explicit `prisma db seed` (https://www.prisma.io/docs/orm/v7/prisma-migrate/workflows/seeding) |
| `--skip-generate`, `--skip-seed` | removed, run `prisma generate` and `prisma db seed` explicitly (https://www.prisma.io/docs/orm/v6/more/upgrades/to-v7) |
| `prisma db execute --url ... --schema ...` | `prisma db execute --file x.sql [--config other.config.ts]` (https://www.prisma.io/docs/cli/v7/db/execute) |
| `migrate diff --from-url`, `--to-schema-datasource`, `--shadow-database-url` | `--from-config-datasource`, `--to-config-datasource`, shadow URL in config (https://www.prisma.io/docs/cli/v7/migrate/diff) |
| `adapter`, `engine`, `studio`, `datasource.directUrl` keys in config | removed, Studio runs with `prisma studio --config ./prisma.config.ts` (https://www.prisma.io/docs/orm/v7/reference/prisma-config-reference) |
| `previewFeatures` `driverAdapters`, `multiSchema`, `prismaSchemaFolder`, `omitApi`, `tracing` | GA, remove the flags (https://www.prisma.io/docs/orm/v7/reference/preview-features/client-preview-features) |
| Env vars `PRISMA_CLIENT_ENGINE_TYPE`, `PRISMA_QUERY_ENGINE_*`, `PRISMA_MIGRATE_SKIP_GENERATE`, `PRISMA_MIGRATE_SKIP_SEED`, `PRISMA_GENERATE_*` | removed, delete them. `PRISMA_BINARIES_MIRROR` is deprecated, `PRISMA_ENGINES_MIRROR` is listed alongside it (https://www.prisma.io/docs/orm/v7/reference/prisma-cli-reference https://www.prisma.io/docs/orm/v6/more/upgrades/to-v7) |
| `new PrismaLibSQL(createClient(...))` | pass driver options directly: `new PrismaLibSQL({ url, authToken })` (https://www.prisma.io/docs/orm/v7/core-concepts/supported-databases/database-drivers) |
| `--data-proxy`, `--accelerate` on `prisma generate` | `--no-engine` (https://www.prisma.io/docs/orm/v7/reference/prisma-cli-reference) |
