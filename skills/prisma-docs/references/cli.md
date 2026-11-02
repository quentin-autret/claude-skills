# Prisma CLI 7

## Contents
- General
- init, generate, validate, format
- db pull, db push, db seed, db execute
- migrate subcommands
- migrate diff
- dev and studio
- Environment variables

## General

- Install the v7 CLI as a dev dependency with `prisma@prev` (the docs command). `prisma@latest` is the Prisma 8 CLI. https://www.prisma.io/docs/cli/v7
- The `db` subcommands read the datasource URL from the Prisma config file and accept `--config <path>`. Most also take `--schema <path>`. https://www.prisma.io/docs/cli/v7/db
- Removed flags: `--skip-generate`, `--skip-seed`, `db execute --url/--schema`, `migrate diff --from-url/--to-url/--from-schema-datasource/--to-schema-datasource/--shadow-database-url`. https://www.prisma.io/docs/orm/v7/reference/prisma-cli-reference
- Run the CLI with `npx prisma` or `bunx prisma` from the project root, where `package.json` declares Prisma: that is the only place the config file is detected. A global install may not find `prisma.config.ts`. https://www.prisma.io/docs/orm/v7/reference/prisma-config-reference
- With Bun, use `bunx --bun prisma init`: the `--bun` flag makes it run on Bun instead of falling back to Node.js through the CLI shebang. https://www.prisma.io/docs/guides/v7/runtimes/bun

## init, generate, validate, format

- `init` writes `prisma/schema.prisma`, the config, `.env` and a `.gitignore` (node_modules, .env, /generated/prisma). Options: `--datasource-provider`, `--output`, `--url`, `--with-model`, `--preview-feature`. `--generator-provider prisma-client-js` opts into the legacy generator. https://www.prisma.io/docs/cli/v7/init
- `generate` needs a `generator` with `provider = "prisma-client"` and `output`. Options: `--sql`, `--watch`, `--generator <name>` (repeatable), `--no-hints`. The page names `--require-models` while the reference names `--allow-no-models`: check `prisma generate --help` before scripting it. https://www.prisma.io/docs/cli/v7/generate https://www.prisma.io/docs/orm/v7/reference/prisma-cli-reference
- The sample output on the generate page shows `./node_modules/.prisma/client`, while the same page says the client goes into the `output` directory of the generator block. https://www.prisma.io/docs/cli/v7/generate
- `validate` checks the schema, and its sample error output carries code `P1012`. `format` validates, formats and persists the schema (it fails on validation errors). https://www.prisma.io/docs/cli/v7/validate https://www.prisma.io/docs/cli/v7/format
- `prisma version --json` prints versions in machine-readable form. https://www.prisma.io/docs/cli/v7/version

## db pull, db push, db seed, db execute

- `db pull` overwrites the schema file: commit first, or `--print` to stdout. `--force` drops customisations, `--schemas` picks database schemas. https://www.prisma.io/docs/cli/v7/db/pull
- `db push` creates the database if missing and syncs the schema without migrations. `--accept-data-loss`, `--force-reset`. It suits prototyping and local development, and no longer runs `prisma generate`. https://www.prisma.io/docs/cli/v7/db/push
- `db seed` accepts only `--config`, arguments for the script follow `--`. https://www.prisma.io/docs/cli/v7/db/seed
- `db execute` requires either `--file` or `--stdin`, sends the whole script as one command, reports success or failure only and never touches the migrations table. `echo 'TRUNCATE TABLE dev;' | prisma db execute --stdin`. https://www.prisma.io/docs/cli/v7/db/execute

## migrate subcommands

- Development: `migrate dev` (`--name`, `--create-only`), `migrate reset`. Staging and production: `migrate deploy`, `migrate status`, `migrate resolve`. Any stage: `migrate diff`. https://www.prisma.io/docs/cli/v7/migrate
- `migrate reset --force` skips the confirmation prompt. https://www.prisma.io/docs/cli/v7/migrate/reset
- `migrate dev` no longer runs `prisma generate` or seed scripts. https://www.prisma.io/docs/cli/v7/migrate/dev
- `migrate dev --create-only` still prompts a reset when it detects drift. https://www.prisma.io/docs/cli/v7/migrate/dev
- `migrate deploy` accepts only `--config` and `--schema`. https://www.prisma.io/docs/cli/v7/migrate/deploy
- `migrate resolve` requires `--applied <name>` or `--rolled-back <name>` and errors on a migration that succeeded. https://www.prisma.io/docs/cli/v7/migrate/resolve
- `migrate status` exit code 1: connection error, unapplied or failed migrations, diverged history, no migrations table. https://www.prisma.io/docs/cli/v7/migrate/status

## migrate diff

- Sources: `--from-empty`, `--from-schema <file>`, `--from-migrations <dir>`, `--from-config-datasource`, and the `--to-*` mirrors. Both sides share one provider. https://www.prisma.io/docs/cli/v7/migrate/diff
- `--script` prints SQL, `-o/--output <file>` writes it, `--exit-code` returns 0 empty, 1 error, 2 changes. Views and triggers are invisible to the comparison. https://www.prisma.io/docs/cli/v7/migrate/diff
- Patterns: https://www.prisma.io/docs/cli/v7/migrate/diff
  ```bash
  # roll forward after a failed migration
  prisma migrate diff --from-config-datasource --to-schema=next_datamodel.prisma --script
  # apply the diff by piping it to db execute
  prisma migrate diff --from-config-datasource --to-schema=schema.prisma --script | prisma db execute --stdin
  # capture a production hotfix
  prisma migrate diff --from-migrations ./migrations --to-config-datasource --script
  # drift gate in CI
  prisma migrate diff --exit-code --from-config-datasource --to-schema=schema.prisma
  ```
  https://www.prisma.io/docs/cli/v7/migrate/diff

## dev and studio

- `prisma dev` runs a local Prisma Postgres (port 51213, database 51214, shadow 51215). `-n/--name` isolates projects, `-d` detaches, subcommands `ls`, `rm`, `start`, `stop`. https://www.prisma.io/docs/cli/v7/dev
- `prisma studio` supports PostgreSQL, MySQL and SQLite only. `-p` (default 5555), `-b none` disables the browser, `--config`. https://www.prisma.io/docs/cli/v7/studio

## Environment variables

- Unsupported in v7: `PRISMA_CLI_QUERY_ENGINE_TYPE`, `PRISMA_CLIENT_ENGINE_TYPE`, `PRISMA_QUERY_ENGINE_BINARY`, `PRISMA_QUERY_ENGINE_LIBRARY`, `PRISMA_GENERATE_SKIP_AUTOINSTALL`, `PRISMA_SKIP_POSTINSTALL_GENERATE`, `PRISMA_GENERATE_IN_POSTINSTALL`, `PRISMA_GENERATE_DATAPROXY`, `PRISMA_GENERATE_NO_ENGINE`, `PRISMA_CLIENT_NO_RETRY`, `PRISMA_MIGRATE_SKIP_GENERATE`, `PRISMA_MIGRATE_SKIP_SEED`. https://www.prisma.io/docs/orm/v7/reference/prisma-cli-reference
- Listed without the unsupported mark: `PRISMA_SCHEMA_DISABLE_ADVISORY_LOCK`, `PRISMA_HIDE_UPDATE_MESSAGE`, `PRISMA_DISABLE_WARNINGS`, `PRISMA_ENGINES_MIRROR`, `PRISMA_SCHEMA_ENGINE_BINARY`. https://www.prisma.io/docs/orm/v7/reference/prisma-cli-reference
