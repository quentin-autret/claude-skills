# Database-specific notes

## Contents
- Connection URLs and support matrix
- PostgreSQL and hosted Postgres
- CockroachDB
- MySQL, MariaDB and PlanetScale
- SQLite, Turso and Cloudflare D1
- SQL Server
- MongoDB

## Connection URLs and support matrix

- A schema can have only one `datasource` block. Its connection URL is set in `prisma.config.ts`, not in the schema. https://www.prisma.io/docs/orm/v7/prisma-schema/overview/data-sources
- Percent-encode special characters in MySQL, PostgreSQL and CockroachDB URLs (`p@$$w0rd` becomes `p%40%24%24w0rd`). SQL Server wraps such values in braces. https://www.prisma.io/docs/orm/v7/reference/connection-urls https://www.prisma.io/docs/orm/v7/core-concepts/supported-databases/sql-server
- Scalar lists: PostgreSQL, CockroachDB, MongoDB only. Enums: not on SQL Server (SQLite and MongoDB enforce them in Prisma only). Json: not on SQL Server. https://www.prisma.io/docs/orm/v7/core-concepts/supported-databases https://www.prisma.io/docs/orm/v7/reference/prisma-schema-reference#enum https://www.prisma.io/docs/orm/v7/core-concepts/supported-databases/sql-server
- Versions: PostgreSQL 9.6 to 18, MySQL 5.6+, MariaDB 10.0+, SQL Server 2017+, CockroachDB 21.2.4+. AWS Aurora works, not the Aurora Serverless Data API. https://www.prisma.io/docs/orm/v7/reference/supported-databases

## PostgreSQL and hosted Postgres

- Default type mappings to keep in mind: `DateTime` is `timestamp(3)` (`@db.Timestamptz(x)` selects `timestamptz`), `Decimal` is `decimal(65,30)`, `Json` is `jsonb`. https://www.prisma.io/docs/orm/v7/reference/prisma-schema-reference#datetime
- Unix socket: `?host=/var/run/postgresql/`. https://www.prisma.io/docs/orm/v7/core-concepts/supported-databases/postgresql
- Neon: `-pooler` in the host for pooling, scales to zero after 5 minutes idle (cold start of a few hundred ms). Use the serverless driver adapter `@prisma/adapter-neon` (`PrismaNeon`) and a direct URL for the CLI. https://www.prisma.io/docs/orm/v7/core-concepts/supported-databases/postgresql https://www.prisma.io/docs/orm/v7/core-concepts/supported-databases/database-drivers
- Supabase: direct `db.<ref>.supabase.co:5432`, transaction pooler on port 6543 with `?pgbouncer=true`, session pooler on 5432 of the pooler host. https://www.prisma.io/docs/orm/v7/core-concepts/supported-databases/postgresql
- AWS RDS forces SSL: `P1010` comes from certificate validation. `?sslmode=no-verify` skips verification and the docs warn it may not meet production needs: configure a valid certificate. https://www.prisma.io/docs/orm/v7/prisma-client/deployment/caveats-when-deploying-to-aws-platforms

## CockroachDB

- `provider = "cockroachdb"`, PostgreSQL URL format. Use `autoincrement()` on `BigInt` (backed by `unique_rowid()`) and avoid sequential ids. `sequence()` exists for existing databases. Native types differ (`STRING(n)` instead of `VARCHAR(n)`). https://www.prisma.io/docs/orm/v7/core-concepts/supported-databases/postgresql
- Indexes: only GIN and BTree, no `ops` argument. https://www.prisma.io/docs/orm/v7/prisma-schema/data-model/indexes

## MySQL, MariaDB and PlanetScale

- Both MySQL and MariaDB use `provider = "mysql"` and `@prisma/adapter-mariadb`. https://www.prisma.io/docs/orm/v7/core-concepts/supported-databases/mysql
  ```ts
  new PrismaMariaDb({ host: "localhost", port: 3306, connectionLimit: 5 });
  ```
  https://www.prisma.io/docs/orm/v7/core-concepts/supported-databases/mysql
- `String` defaults to `VARCHAR(191)`. A `TINYINT(1)` maps to `Boolean`, but `TINYINT(1) UNSIGNED` or a length above 1 maps to `Int`. Zero dates (`0000-00-00`) are unsupported. https://www.prisma.io/docs/orm/v7/reference/prisma-schema-reference#boolean https://www.prisma.io/docs/orm/v7/core-concepts/supported-databases/mysql
- SSL: `sslaccept` defaults to `accept_invalid_certs`. For strict validation set `?sslcert=./cert.pem&sslaccept=strict`. https://www.prisma.io/docs/orm/v7/core-concepts/supported-databases/mysql
- An `@id` on `@db.VarChar(3000)` exceeds the index size limit: use `@id(length: 100)`. https://www.prisma.io/docs/orm/v7/prisma-schema/data-model/indexes
- PlanetScale: `@prisma/adapter-planetscale`. Use `prisma db push`, not Migrate (P3022 on production branches). Without foreign keys enabled, set `relationMode = "prisma"` and add `@@index` on every relation scalar. `@shardKey` needs the `shardKeys` preview. https://www.prisma.io/docs/orm/v7/core-concepts/supported-databases/mysql

## SQLite, Turso and Cloudflare D1

- Adapters: `new PrismaBetterSqlite3({ url: "file:./prisma/dev.db" })`, `new PrismaLibSQL({ url, authToken })`, `new PrismaD1(env.DB)`. https://www.prisma.io/docs/orm/v7/core-concepts/supported-databases/sqlite
- `DateTime` is stored as ISO 8601. A database written by Prisma's native driver needs `{ timestampFormat: "unixepoch-ms" }`. https://www.prisma.io/docs/orm/v7/core-concepts/supported-databases/sqlite
- Enum values are not enforced by the database. Integer overflow gives `P2023`. https://www.prisma.io/docs/orm/v7/core-concepts/supported-databases/sqlite
- TypedSQL does not work with `@prisma/adapter-better-sqlite3`: use `@prisma/adapter-libsql`. https://www.prisma.io/docs/orm/v7/core-concepts/supported-databases/database-drivers
- D1 has no transaction support. Apply schema changes with `prisma migrate diff` plus `wrangler d1 migrations apply`. Turso uses `prisma migrate diff` plus the Turso CLI. https://www.prisma.io/docs/orm/v7/core-concepts/supported-databases/sqlite
- Each Prisma release ships a fixed SQLite version. https://www.prisma.io/docs/orm/v7/reference/supported-databases

## SQL Server

- Adapter `@prisma/adapter-mssql`: `new PrismaMssql({ server, port, database, user, password, options: { encrypt, trustServerCertificate } })`. https://www.prisma.io/docs/orm/v7/core-concepts/supported-databases/sql-server
- URL format `sqlserver://HOST:PORT;database=...;user=...;password=...;encrypt=true`. Wrap values containing `: \ = ; / [ ] { }` in `{}`. `schema` defaults to `dbo`. https://www.prisma.io/docs/orm/v7/core-concepts/supported-databases/sql-server
- Only one NULL per UNIQUE constraint. No `Json`, no enums, no `Restrict` (use `NoAction`). Cyclic references and multiple cascade paths need `NoAction` instead of cascading actions. https://www.prisma.io/docs/orm/v7/core-concepts/supported-databases/sql-server https://www.prisma.io/docs/orm/v7/prisma-schema/data-model/relations/referential-actions
- Raw-query string parameters are `NVARCHAR`: cast them for indexed `VARCHAR` columns, `CAST(${x} AS VARCHAR(40))`. https://www.prisma.io/docs/orm/v7/core-concepts/supported-databases/sql-server
- Migrate recreates the table when adding or removing `autoincrement()` or dropping all columns. Keep `schema=dbo` consistent with production. https://www.prisma.io/docs/orm/v7/core-concepts/supported-databases/sql-server

## MongoDB

- Prisma ORM 7 does not support MongoDB. Use Prisma ORM 6.19. MongoDB examples on v7 pages (with `url` in the `datasource` block) are v6 syntax: never copy them into a SQL project. In Prisma ORM 7 the connection URL belongs in `prisma.config.ts`; a `url` in the schema `datasource` block is for version 6. https://www.prisma.io/docs/orm/v7/core-concepts/supported-databases/mongodb https://www.prisma.io/docs/orm/v7/reference/connection-urls
