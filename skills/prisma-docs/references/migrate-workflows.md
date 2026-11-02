# Migrate: commands, history, drift and the shadow database

## Contents
- Which command where
- migrate dev
- migrate deploy
- db push
- migrate reset
- Migration history rules
- Drift and history conflicts
- Shadow database

## Which command where

- Development: `migrate dev` (or `db push` while prototyping). Testing, staging and production: `migrate deploy` from CI. Never `migrate dev`, `migrate reset` or `db push` against production. https://www.prisma.io/docs/orm/v7/prisma-migrate/workflows/development-and-production https://www.prisma.io/docs/orm/v7/more/best-practices
- `migrate dev` runs neither `prisma generate` nor the seed, `migrate reset` does not seed, `migrate deploy` generates nothing and `db push` triggers no generator. Run `prisma generate` and `prisma db seed` explicitly. https://www.prisma.io/docs/cli/v7/migrate/dev https://www.prisma.io/docs/cli/v7/migrate/reset https://www.prisma.io/docs/cli/v7/migrate/deploy https://www.prisma.io/docs/orm/v7/prisma-migrate/workflows/prototyping-your-schema https://www.prisma.io/docs/orm/v7/prisma-migrate/workflows/seeding
- `migrate status` exits with 1 on unapplied, failed or diverged migrations: use it as a deploy gate. https://www.prisma.io/docs/cli/v7/migrate/status
- Advisory locking stops `migrate deploy`, `migrate dev` and `migrate resolve` from running concurrently (10 s timeout, not configurable, rerun on timeout). `PRISMA_SCHEMA_DISABLE_ADVISORY_LOCK` turns it off (since 5.3.0). https://www.prisma.io/docs/orm/v7/prisma-migrate/workflows/development-and-production
- MongoDB has no Migrate: use `db push`. https://www.prisma.io/docs/cli/v7/migrate

## migrate dev

- Replays history in the shadow database to detect drift, applies pending migrations, creates a new one when the schema changed, applies it to the dev database. https://www.prisma.io/docs/orm/v7/prisma-migrate/workflows/development-and-production
- It prompts for a reset (data loss) on drift or when an applied migration was edited or removed. https://www.prisma.io/docs/orm/v7/prisma-migrate/workflows/development-and-production
- To edit the SQL before it runs: `prisma migrate dev --create-only --name x` creates the migration without applying it, edit `migration.sql`, then `prisma migrate dev` to apply. https://www.prisma.io/docs/cli/v7/migrate/dev
- Folders are `YYYYMMDDHHMMSS_<name>` and apply in lexicographic order. https://www.prisma.io/docs/orm/v7/prisma-migrate/getting-started
- Unsupported without a TTY (Docker, scripts): run the container with `-it`. https://www.prisma.io/docs/orm/v7/prisma-migrate/understanding-prisma-migrate/limitations-and-known-issues

## migrate deploy

- Applies pending migrations and warns when an applied one was modified. It does not detect drift, does not notice a missing applied migration, never resets, uses no shadow database and generates nothing. https://www.prisma.io/docs/cli/v7/migrate/deploy https://www.prisma.io/docs/orm/v7/prisma-migrate/workflows/development-and-production
- It runs only the migration files, not the Prisma schema: committing `schema.prisma` without its migration changes nothing. https://www.prisma.io/docs/orm/v7/prisma-migrate/understanding-prisma-migrate/migration-histories
- Run it in CI, not from a laptop with the production URL pasted in. https://www.prisma.io/docs/orm/v7/prisma-client/deployment/deploy-migrations-from-a-local-environment
  ```yaml
  on:
    push:
      paths: ["prisma/migrations/**"]
      branches: [main]
  # step
  - run: npx prisma migrate deploy
    env:
      DATABASE_URL: ${{ secrets.DATABASE_URL }}
  ```
  https://www.prisma.io/docs/orm/v7/prisma-client/deployment/deploy-database-changes-with-prisma-migrate
- Platforms that prune devDependencies need `prisma` in `dependencies`. https://www.prisma.io/docs/orm/v7/prisma-client/deployment/deploy-database-changes-with-prisma-migrate
- Migrations go through a direct connection: PgBouncer breaks them (`prepared statement "s0" already exists`). Point Prisma CLI at a direct URL in `prisma.config.ts` and keep the pooled URL for the client. https://www.prisma.io/docs/orm/v7/prisma-migrate/understanding-prisma-migrate/limitations-and-known-issues https://www.prisma.io/docs/orm/v7/prisma-client/setup-and-configuration/databases-connections/pgbouncer

## db push

- Syncs the schema without migration files or history. Data-loss changes need `--accept-data-loss`. It cannot be customized, so a column is dropped and recreated instead of renamed. https://www.prisma.io/docs/orm/v7/prisma-migrate/workflows/prototyping-your-schema
- Use it for local prototyping only, then create the first migration with `migrate dev --name initial-state`. On a project with history, running `migrate dev` after a `db push` experiment asks to reset the database. https://www.prisma.io/docs/orm/v7/prisma-migrate/workflows/prototyping-your-schema
- PlanetScale is the documented exception where `db push` replaces Migrate. https://www.prisma.io/docs/orm/v7/prisma-migrate/workflows/prototyping-your-schema

## migrate reset

- Drops the database (MySQL), the schema (PostgreSQL, SQL Server) or the file (SQLite) if possible, else does a soft reset, then recreates it and reapplies all migrations. It does not seed. Development only. https://www.prisma.io/docs/orm/v7/prisma-migrate/workflows/development-and-production https://www.prisma.io/docs/cli/v7/migrate/reset
- When an AI agent runs `prisma migrate reset --force`, Prisma blocks it. It proceeds only after the user explicitly consents to the data loss, recorded through `PRISMA_USER_CONSENT_FOR_DANGEROUS_AI_ACTION`. Never set that variable without that consent. https://www.prisma.io/docs/cli/v7/migrate/reset

## Migration history rules

- Commit the whole `prisma/migrations` folder, `migration_lock.toml` included (it detects provider switches). https://www.prisma.io/docs/orm/v7/prisma-migrate/understanding-prisma-migrate/migration-histories
- Never edit or delete an applied migration. Edits are caught through the `checksum` field of `_prisma_migrations`: `migrate dev` asks to reset and `migrate deploy` warns on every run. Restore the original file. https://www.prisma.io/docs/orm/v7/prisma-migrate/understanding-prisma-migrate/migration-histories https://www.prisma.io/docs/orm/v7/prisma-migrate/understanding-prisma-migrate/shadow-database
- Switching provider needs a new history: change `provider` and the URL, archive `prisma/migrations`, run `migrate dev`, re-add custom SQL by hand. Data is not moved. https://www.prisma.io/docs/orm/v7/prisma-migrate/understanding-prisma-migrate/limitations-and-known-issues
- After pulling teammates' migrations, run `migrate dev` (applies theirs, creates yours) then `prisma generate`. https://www.prisma.io/docs/guides/v7/database/schema-changes

## Drift and history conflicts

- Drift (manual change, `db push`) is detected only in dev, through the shadow database. Discard it: `migrate reset`, replicate in the schema, `migrate dev`. Keep it: `db pull`, then `migrate dev --name introspected_change`. https://www.prisma.io/docs/orm/v7/prisma-migrate/workflows/troubleshooting
- Switching git branches can remove applied migrations from the folder: resolve with a reset. https://www.prisma.io/docs/orm/v7/prisma-migrate/workflows/troubleshooting
- A failed migration stores its error in the `logs` column of `_prisma_migrations`. Common causes: a `NOT NULL` column added to a populated table (add a default), an SQL syntax error in an edited file. https://www.prisma.io/docs/orm/v7/prisma-migrate/workflows/troubleshooting
- CI drift check: `prisma migrate diff --exit-code --from-config-datasource --to-schema=schema.prisma` (exit 2 means not empty). https://www.prisma.io/docs/cli/v7/migrate/diff

## Shadow database

- Created and dropped on each `migrate dev`, also used by `migrate diff --from-migrations/--to-migrations`. Not used by `deploy` or `resolve`, so production needs none. https://www.prisma.io/docs/orm/v7/prisma-migrate/understanding-prisma-migrate/shadow-database
- The dev user must create databases: PostgreSQL superuser or `CREATEDB`. MySQL: `CREATE, ALTER, DROP, INDEX, REFERENCES` on `prisma_migrate_shadow_db%`. Otherwise: `permission denied to create database`. https://www.prisma.io/docs/orm/v7/prisma-migrate/understanding-prisma-migrate/shadow-database
- Cloud databases that cannot create databases: create one shadow database and set `datasource.shadowDatabaseUrl: env("SHADOW_DATABASE_URL")` in the config. It is reset at the start of each `migrate dev`. https://www.prisma.io/docs/orm/v7/prisma-migrate/understanding-prisma-migrate/shadow-database
- Never give `url` and `shadowDatabaseUrl` the same value: it can delete all data. https://www.prisma.io/docs/orm/v7/prisma-migrate/understanding-prisma-migrate/shadow-database
- On MySQL 8.0.35+ with `partial_revokes`, wildcard grants break: use a dedicated shadow database granted by name. https://www.prisma.io/docs/orm/v7/prisma-migrate/understanding-prisma-migrate/shadow-database
