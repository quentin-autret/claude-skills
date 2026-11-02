# Migrate recipes: custom SQL, zero downtime, baselining, hotfixes, squashing, seeding

## Contents
- Customizing a migration
- Expand and contract
- Baselining an existing database
- Hotfixes and failed migrations
- Down migrations
- Squashing
- Seeding

## Customizing a migration

- A rename is generated as DROP plus ADD (data loss). Create with `--create-only` and replace it: `ALTER TABLE "Profile" RENAME COLUMN "biograpy" TO "biography";`. https://www.prisma.io/docs/orm/v7/prisma-migrate/workflows/customizing-migrations
- When a change needs custom SQL to avoid data loss, draft it with `migrate dev --create-only`, edit the generated SQL, then apply with `migrate dev`. https://www.prisma.io/docs/orm/v7/prisma-migrate/workflows/customizing-migrations
- Stored procedures, triggers, views and other features the Prisma schema cannot represent go into a committed `--create-only` migration. https://www.prisma.io/docs/orm/v7/prisma-migrate/workflows/unsupported-database-features
- Changing which side of a one-to-one holds the key: add a nullable column, copy with `UPDATE ... FROM`, `SET NOT NULL`, drop the old column, then add the unique index and foreign key. https://www.prisma.io/docs/orm/v7/prisma-migrate/workflows/customizing-migrations
- Partial indexes need no custom SQL: use the `where` argument on `@@index`, `@@unique` and `@unique`. https://www.prisma.io/docs/orm/v7/prisma-migrate/workflows/unsupported-database-features

## Expand and contract

- Changing an existing field the app uses, such as renaming it, can cause downtime between the migration and the app deploy. Steps for `bio` to `biography`: add the new field and migrate, deploy an app that writes both and reads the old one, copy data in an empty `--create-only` migration (`UPDATE "Profile" SET biography = bio;`), verify, read the new field, stop writing the old one, then remove it and migrate. https://www.prisma.io/docs/orm/v7/prisma-migrate/workflows/customizing-migrations
- A TypeScript data migration runs between the expand migration and the contract migration (the old column is dropped later, on a separate branch), inside one `$transaction`. https://www.prisma.io/docs/guides/v7/database/data-migration
- Back up and test on a copy of production data first. https://www.prisma.io/docs/guides/v7/database/data-migration

## Baselining an existing database

Needed when adding Migrate to a database that already holds data and cannot be reset. Baselining is performed once and can be done from a local instance, whereas deploying other migrations from a local machine is discouraged. https://www.prisma.io/docs/orm/v7/prisma-migrate/workflows/baselining https://www.prisma.io/docs/orm/v7/prisma-client/deployment/deploy-migrations-from-a-local-environment

```bash
npx prisma db pull
mkdir -p prisma/migrations/0_init
npx prisma migrate diff --from-empty --to-schema prisma/schema.prisma --script > prisma/migrations/0_init/migration.sql
# review the SQL
npx prisma migrate resolve --applied 0_init
```

https://www.prisma.io/docs/v7/prisma-orm/add-to-existing-project/postgresql

- The `0_` prefix makes it sort first. Add triggers and other unsupported features to the baseline SQL, or replace it with database dump output such as `pg_dump`. https://www.prisma.io/docs/orm/v7/prisma-migrate/getting-started
- With `pg_dump`, run `SELECT pg_catalog.set_config('search_path', '', false);` or you get "The underlying table for model '_prisma_migrations' does not exist". https://www.prisma.io/docs/orm/v7/prisma-migrate/getting-started

## Hotfixes and failed migrations

- `migrate resolve` fixes history on deployment databases: recovering a failed migration, baselining, reconciling a hotfix. It errors on a migration that succeeded. Run `migrate status` first. https://www.prisma.io/docs/cli/v7/migrate/resolve
- Hotfix applied directly in production: replicate it in the schema, `migrate dev --name retroactively-add-index` locally, then `migrate resolve --applied <full name>` on each patched database. CI applies it to the others. https://www.prisma.io/docs/orm/v7/prisma-migrate/workflows/patching-and-hotfixing
- A failed migration blocks further deploys. Either `migrate resolve --rolled-back <name>`, undo partial work, fix and redeploy, or finish the steps by hand and `migrate resolve --applied <name>`. https://www.prisma.io/docs/orm/v7/prisma-migrate/workflows/patching-and-hotfixing
- `db execute` has no `--url`. Point it at another database with a separate config file: https://www.prisma.io/docs/orm/v7/prisma-migrate/workflows/patching-and-hotfixing
  ```bash
  prisma migrate diff --from-config-datasource --to-migrations ./prisma/migrations --config prisma.config.prod.ts --script > backward.sql
  prisma db execute --config prisma.config.prod.ts --file backward.sql
  prisma migrate resolve --rolled-back Unique
  ```
  https://www.prisma.io/docs/orm/v7/prisma-migrate/workflows/patching-and-hotfixing

## Down migrations

- A down migration is a SQL file you generate with `migrate diff` before creating the up migration, here `down.sql`: https://www.prisma.io/docs/orm/v7/prisma-migrate/workflows/generating-down-migrations
  ```bash
  prisma migrate diff --from-schema prisma/schema.prisma --to-migrations prisma/migrations --script > down.sql
  prisma migrate dev --name add_profile
  ```
  https://www.prisma.io/docs/orm/v7/prisma-migrate/workflows/generating-down-migrations
- Apply with `db execute --file ./down.sql` then `migrate resolve --rolled-back add_profile`, which works only on a failed migration. To revert a successful one, restore the old schema and create a new forward migration. https://www.prisma.io/docs/orm/v7/prisma-migrate/workflows/generating-down-migrations
- `migrate diff` ignores hand-written SQL (views, triggers): write their inverse yourself. https://www.prisma.io/docs/orm/v7/prisma-migrate/workflows/generating-down-migrations

## Squashing

- Feature branch, before merge: only if none of its migrations reached production and none holds custom SQL. Reset local `prisma/migrations` to the main branch, then `migrate dev --name squashed_migrations`. https://www.prisma.io/docs/orm/v7/prisma-migrate/workflows/squashing-migrations
- Production history: empty `prisma/migrations`, create `000000000000_squashed_migrations/migration.sql` with `migrate diff --from-empty --to-schema ./prisma/schema.prisma --script`, then `migrate resolve --applied 000000000000_squashed_migrations`. https://www.prisma.io/docs/orm/v7/prisma-migrate/workflows/squashing-migrations
- Squashing drops custom SQL from old migrations: re-add it. https://www.prisma.io/docs/orm/v7/prisma-migrate/workflows/squashing-migrations

## Seeding

- The command sits in `migrations.seed` of the config and runs only on `prisma db seed`. Pass arguments after `--`: `prisma db seed -- --environment development`. https://www.prisma.io/docs/orm/v7/prisma-migrate/workflows/seeding
- The seed builds its own client with a driver adapter, and closes both client and pool at the end and on error: https://www.prisma.io/docs/orm/v7/prisma-migrate/workflows/seeding
  ```ts
  import "dotenv/config";
  import { Pool } from "pg";
  import { PrismaPg } from "@prisma/adapter-pg";
  import { PrismaClient } from "../prisma/generated/client";
  const pool = new Pool({ connectionString: `${process.env.DATABASE_URL}` });
  const prisma = new PrismaClient({ adapter: new PrismaPg(pool) });
  // ... await prisma.$disconnect(); await pool.end();
  ```
  https://www.prisma.io/docs/orm/v7/prisma-migrate/workflows/seeding
- The example seeds use `upsert` with `update: {}` on a unique field (`email`), and a raw insert through `$executeRaw` that ends in `ON CONFLICT DO NOTHING`. https://www.prisma.io/docs/orm/v7/prisma-migrate/workflows/seeding
