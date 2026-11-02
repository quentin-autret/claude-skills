# Setup, prisma.config.ts and the generator

## Contents
- Installing Prisma ORM 7
- prisma.config.ts
- Environment variables
- The prisma-client generator
- Generated files and imports
- Generating the client
- Preview features

## Installing Prisma ORM 7

- New project: `npx create-prisma@stable`. Existing project: `npx prisma@prev init`. Install v7 with the `prev` tag for the CLI and the `@7` tag for the client. https://www.prisma.io/docs/v7/getting-started
- PostgreSQL packages: `prisma@prev` and `@types/pg` (dev), `@prisma/client@7 @prisma/adapter-pg pg dotenv`. https://www.prisma.io/docs/v7/prisma-orm/quickstart/postgresql
- MySQL/MariaDB adapter: `@prisma/adapter-mariadb`. https://www.prisma.io/docs/orm/v7/core-concepts/supported-databases/mysql
- SQLite adapter: `@prisma/adapter-better-sqlite3`. https://www.prisma.io/docs/orm/v7/core-concepts/supported-databases/sqlite
- `prisma init --datasource-provider postgresql --output ../generated/prisma` creates `prisma/schema.prisma`, `.env`, the config file and a `.gitignore` with `/generated/prisma`. Without `--datasource-provider`, init targets local Prisma Postgres. With Bun use `bunx --bun prisma init`. https://www.prisma.io/docs/cli/v7/init
- From 7.10, `prisma init` writes `prisma7.config.ts` (Prisma 8 only accepts `prisma.config.ts`). The v7 CLI discovers either name. https://www.prisma.io/docs/v7/prisma-orm/quickstart/postgresql
- Linux needs OpenSSL (1.0.x, 1.1.x or 3.x), zlib, libgcc and glibc or musl. Distroless images lack `libz.so.1`: copy it in from a Debian image. https://www.prisma.io/docs/orm/v7/reference/system-requirements

## prisma.config.ts

The config file is the CLI configuration: schema location, migrations path, seed command, datasource URL. Put it at the project root next to `package.json`. https://www.prisma.io/docs/orm/v6/more/upgrades/to-v7

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

- Keys: `schema` (file or folder), `migrations.path`, `migrations.seed`, `migrations.initShadowDb` (SQL run on the shadow database first), `views.path`, `typedSql.path`, `datasource.url` (required), `datasource.shadowDatabaseUrl`, `tables.external`, `enums.external`, `experimental.externalTables`. `satisfies PrismaConfig` is the alternative to `defineConfig`. https://www.prisma.io/docs/orm/v7/reference/prisma-config-reference
- `env("X")` throws when `X` is missing, and every CLI command loads the config. A CI job that only runs `prisma generate` then fails with `PrismaConfigEnvError`. Fix: read `process.env.DATABASE_URL` directly (`!` or `?? ''`) when the variable may be absent, and use `env()` only where it must be present. https://www.prisma.io/docs/orm/v7/reference/prisma-config-reference
- Discovery: `npx prisma` and `bunx prisma` find the config only from the project root (from a subfolder: `Could not find Prisma Schema`, fix with `--config ../prisma.config.ts`). `pnpm prisma` finds it from subfolders. Paths inside the config resolve relative to the config file. Avoid a global `prisma` install. https://www.prisma.io/docs/orm/v7/reference/prisma-config-reference
- File names: `prisma.config.ts` for small projects, `.config/prisma.ts` for larger ones (extensions ts, js, mjs, cjs, mts, cts). https://www.prisma.io/docs/orm/v7/reference/prisma-config-reference
- `migrations.seed` runs only on `prisma db seed`. https://www.prisma.io/docs/orm/v7/reference/prisma-config-reference
- Multi-file schema: `schema` points at the folder, the file holding the `generator` block sits directly in it (`prisma/schema.prisma`, not `prisma/models/schema.prisma`) and `migrations/` sits at the same level. Name files by domain (`user.prisma`). https://www.prisma.io/docs/orm/v7/prisma-schema/overview/location
- Schema lookup order: `--schema`, then `schema` in `prisma.config.ts`, then `./prisma/schema.prisma`, then `./schema.prisma`. https://www.prisma.io/docs/orm/v7/prisma-schema/overview/location

## Environment variables

- Nothing loads `.env` for you: import `dotenv/config` in the config and in the app entry point, or use `node --env-file`. Bun loads `.env` itself, Deno uses `--env=.env`. https://www.prisma.io/docs/orm/v6/more/upgrades/to-v7 https://www.prisma.io/docs/guides/v7/runtimes/deno https://www.prisma.io/docs/orm/v7/core-concepts/supported-databases/postgresql
- A missing `DATABASE_URL` with `pg` shows as `SASL: SCRAM-SERVER-FIRST-MESSAGE: client password must be a string`. https://www.prisma.io/docs/orm/v7/core-concepts/supported-databases/postgresql
- Conflicting variables between `./.env` and another `.env` (for example `prisma/.env`) make the CLI throw. Do not keep several environments' URLs in one `.env`: use one file per environment with `dotenv-cli` (`dotenv -e .env.sample -- npx prisma migrate deploy`). https://www.prisma.io/docs/orm/v7/more/dev-environment/environment-variables
- The environment-variables page describes automatic loading of `.env` by the CLI, which the upgrade guide contradicts. Trust the upgrade guide: load explicitly. https://www.prisma.io/docs/orm/v6/more/upgrades/to-v7
- Useful variables (`DEBUG` and `NO_COLOR` are Prisma Client ones, the rest are CLI ones): `DEBUG` (for example `"prisma:client"`), `PRISMA_HIDE_UPDATE_MESSAGE`, `PRISMA_DISABLE_WARNINGS`, `PRISMA_SCHEMA_DISABLE_ADVISORY_LOCK`, `NO_COLOR`, `HTTPS_PROXY`. https://www.prisma.io/docs/orm/v7/reference/environment-variables-reference

## The prisma-client generator

```prisma
generator client {
  provider = "prisma-client"
  output   = "../src/generated/prisma"
}
```

https://www.prisma.io/docs/orm/v7/prisma-schema/overview/generators

- `output` is required and relative to the schema file. The output is plain TypeScript compiled with your code and does not load `.env`. The docs recommend adding the path to `.gitignore` and running `prisma generate` from a build script or postinstall hook. https://www.prisma.io/docs/orm/v7/reference/prisma-schema-reference#generator https://www.prisma.io/docs/orm/v7/prisma-schema/overview/generators
- `runtime`: `nodejs` (default), `deno`, `bun`, `workerd`/`cloudflare`, `vercel-edge`/`edge-light`, `react-native`. https://www.prisma.io/docs/orm/v7/prisma-schema/overview/generators
- `moduleFormat`: `esm` or `cjs`, inferred. The docs recommend `esm` unless there is a reason. https://www.prisma.io/docs/orm/v7/reference/prisma-schema-reference#generator
- `importFileExtension`: inferred (`ts`, `mts`, `cts`, `js`, `mjs`, `cjs` or empty). Running through `tsx` fails with `Cannot find module './internal/class.js'` until you set `importFileExtension = "ts"`. https://www.prisma.io/docs/orm/v7/prisma-schema/overview/generators
- `generatedFileExtension`: `ts`, `mts`, `cts`. `compilerBuild`: `fast` (default, larger) or `small` (smallest bundle, slightly slower). https://www.prisma.io/docs/orm/v7/reference/prisma-schema-reference#generator

## Generated files and imports

- `client.ts`: full `PrismaClient` and `Prisma` namespace, server only. `browser.ts`: types, enums, `Prisma.JsonNull`, `Prisma.Decimal`, no Node dependency, no real client. `enums.ts`: slim, best for tree shaking. `models/<Name>.ts`: model types and inputs. https://www.prisma.io/docs/orm/v7/prisma-schema/overview/generators
- Never import from `internal/*`: unstable. https://www.prisma.io/docs/orm/v7/prisma-schema/overview/generators
- The model type is `User` from `client`/`browser` but `UserModel` from `models`. Prefer granular imports (`enums`, `models/User`) for typecheck speed. https://www.prisma.io/docs/orm/v7/prisma-schema/overview/generators
- `Prisma.validator` does not exist with this generator: use `satisfies`. https://www.prisma.io/docs/orm/v7/prisma-schema/overview/generators
- Optional linked package for a clean import path: `pnpm add db@link:./generated/prisma` (npm: `npm add db@./generated/prisma`). https://www.prisma.io/docs/orm/v7/core-concepts/supported-databases/database-drivers

## Generating the client

- Run `prisma generate` after every schema or generator change and after pulling teammates' schema changes. `prisma migrate dev` and `prisma db push` do not run `prisma generate` for you. https://www.prisma.io/docs/orm/v7/prisma-client/setup-and-configuration/generating-prisma-client https://www.prisma.io/docs/cli/v7/migrate/dev https://www.prisma.io/docs/orm/v6/more/upgrades/to-v7
- Default: `"postinstall": "prisma generate"`, or `prisma generate` before the production build. https://www.prisma.io/docs/orm/v7/prisma-client/setup-and-configuration/generating-prisma-client
- `prisma generate --generator client --generator zod_schemas` runs a subset of generators. `--sql` builds TypedSQL, `--watch` regenerates. https://www.prisma.io/docs/cli/v7/generate
- Client polyfills (not in the database): `cuid()`/`uuid()` ids, `@updatedAt`, implicit many-to-many tables. Prisma Client provides them, the database does not. https://www.prisma.io/docs/orm/v7/prisma-client/setup-and-configuration/database-polyfills

## Preview features

- Enable in the generator, then regenerate (in Visual Studio Code, run "TypeScript: Restart TS server" if types do not appear): `previewFeatures = ["relationJoins"]`. https://www.prisma.io/docs/orm/v7/reference/preview-features/client-preview-features
- Active client previews: `views`, `relationJoins`, `nativeDistinct`, `typedSql`, `strictUndefinedChecks`, `fullTextSearchPostgres`, `shardKeys`, `partialIndexes`. No CLI preview feature is active. https://www.prisma.io/docs/orm/v7/reference/preview-features/client-preview-features https://www.prisma.io/docs/orm/v7/reference/preview-features/cli-preview-features
- Minor releases may break Preview features only. Validate them in staging first. https://www.prisma.io/docs/orm/v7/more/releases
