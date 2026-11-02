# Connections, driver adapters and pooling

## Contents
- Driver adapters
- One client per process
- Pool settings per adapter
- Serverless connection limits
- PgBouncer and other poolers
- Read replicas

## Driver adapters

- Relational datasources use a driver adapter by default in Prisma ORM 7. https://www.prisma.io/docs/orm/v7/prisma-client/setup-and-configuration/databases-connections/connection-pool
- The connection URL lives in `prisma.config.ts` for the CLI, but the app passes its own connection string to the adapter in code. https://www.prisma.io/docs/orm/v7/core-concepts/supported-databases/database-drivers
  ```ts
  import "dotenv/config";
  import { PrismaPg } from "@prisma/adapter-pg";
  import { PrismaClient } from "../generated/prisma/client";

  const adapter = new PrismaPg({ connectionString: process.env.DATABASE_URL });
  export const prisma = new PrismaClient({ adapter });
  ```
  https://www.prisma.io/docs/v7/prisma-orm/quickstart/postgresql
- Instantiate the driver adapter directly with the driver's options. Creating a driver client first and passing it in is not needed. https://www.prisma.io/docs/orm/v7/core-concepts/supported-databases/database-drivers
- Prisma-maintained adapters: `pg`, `@prisma/adapter-ppg` (Prisma Postgres), `mariadb`, `better-sqlite3`, libSQL, `mssql`. Serverless: Prisma Postgres, Neon, PlanetScale, Cloudflare D1. https://www.prisma.io/docs/orm/v7/core-concepts/supported-databases/database-drivers
- The PostgreSQL schema goes to the adapter, not the URL: `new PrismaPg({ connectionString }, { schema: "mySchema" })`. https://www.prisma.io/docs/orm/v7/core-concepts/supported-databases/postgresql
- Prisma timeouts (transactions) and driver timeouts are separate settings: check which one fired. https://www.prisma.io/docs/orm/v7/prisma-client/setup-and-configuration/databases-connections/connection-pool
- Adapter-managed connections are not exposed individually. https://www.prisma.io/docs/orm/v7/prisma-client/setup-and-configuration/databases-connections/connection-pool

## One client per process

- Each `PrismaClient` owns a pool. Several instances create several pools and can hit the database connection limit (`FATAL: sorry, too many clients already`). Export one instance from a module. https://www.prisma.io/docs/orm/v7/prisma-client/setup-and-configuration/introduction
- With hot reload (Next.js dev and similar), cache it on `globalThis` outside production: https://www.prisma.io/docs/orm/v7/prisma-client/setup-and-configuration/databases-connections
  ```ts
  const globalForPrisma = globalThis as unknown as { prisma: PrismaClient };
  export const prisma = globalForPrisma.prisma ?? new PrismaClient({ adapter });
  if (process.env.NODE_ENV !== "production") globalForPrisma.prisma = prisma;
  ```
  https://www.prisma.io/docs/orm/v7/prisma-client/setup-and-configuration/databases-connections
- `$connect()` is lazy: the first query opens the pool, process exit closes it. Call `$connect()` only when the first request must not pay the connection cost. https://www.prisma.io/docs/orm/v7/prisma-client/setup-and-configuration/databases-connections/connection-management
- Do not call `$disconnect()` after each request or at the end of a serverless handler. Call it in an infrequent script that creates its own client inside a long-running app (in `.then` and `.catch`), and when releasing a temporary client (Cloudflare Workers: `ctx.waitUntil(prisma.$disconnect())`). https://www.prisma.io/docs/orm/v7/prisma-client/setup-and-configuration/databases-connections/connection-management
- `$disconnect()` runs the `beforeExit` hook, then closes the pool. `prisma.$on("beforeExit", ...)` runs on external shutdown (SIGINT) with the client still usable. https://www.prisma.io/docs/orm/v7/prisma-client/setup-and-configuration/databases-connections/connection-management
- Each CLI command opens about 1 connection (`migrate dev` up to 4). https://www.prisma.io/docs/orm/v7/prisma-client/setup-and-configuration/databases-connections

## Pool settings per adapter

Pool size and timeouts are adapter options. There are no connection-URL parameters for them in Prisma ORM 7. https://www.prisma.io/docs/orm/v7/prisma-client/setup-and-configuration/databases-connections

| Adapter | Pool size | Connect/acquire timeout | Idle timeout |
| --- | --- | --- | --- |
| `pg` | `max` 10 | `connectionTimeoutMillis` 0 (none) | `idleTimeoutMillis` 10 s |
| `mariadb` | `connectionLimit` 10 | `acquireTimeout` 10 s, `connectTimeout` 1 s | `idleTimeout` 1800 (seconds) |
| `mssql` | `pool.max` 10 | `connectionTimeout` 15 s | `pool.idleTimeoutMillis` 30 s |

https://www.prisma.io/docs/orm/v7/prisma-client/setup-and-configuration/databases-connections/connection-pool

- `pg` has no connect timeout by default (`0`). Setting `connectionTimeoutMillis` adds one (the docs example uses `5_000` to match the former 5 s connect timeout). https://www.prisma.io/docs/orm/v7/prisma-client/setup-and-configuration/databases-connections/connection-pool
- `mariadb` `idleTimeout` is in seconds, unlike the other millisecond options. https://www.prisma.io/docs/orm/v7/prisma-client/setup-and-configuration/databases-connections/connection-pool
- `pg` also has `maxLifetimeSeconds` (default 0). https://www.prisma.io/docs/orm/v7/prisma-client/setup-and-configuration/databases-connections/connection-pool
- `P2024` is "Timed out fetching a new connection from the connection pool". In v7, pool size and timeout are set per driver adapter. https://www.prisma.io/docs/orm/v7/reference/error-reference
- A pool timeout can occur when many users hit the app at once or when many queries run in parallel (for example `await Promise.all()`). https://www.prisma.io/docs/orm/v7/prisma-client/setup-and-configuration/databases-connections

## Serverless connection limits

- Each function instance has its own client and pool: total connections = pool size x concurrent instances. Paused functions keep their connections open by default, and containers marked for removal can keep one open for an unknown time. https://www.prisma.io/docs/orm/v7/prisma-client/setup-and-configuration/databases-connections
- Instantiate the client outside the handler so warm invocations reuse it. Keep the adapter pool small without an external pooler, and cap function concurrency below (database max connections / connections per instance). https://www.prisma.io/docs/orm/v7/prisma-client/setup-and-configuration/databases-connections
- AWS RDS Proxy gives no pooling benefit: Prisma uses prepared statements, which pin the session. https://www.prisma.io/docs/orm/v7/prisma-client/deployment/caveats-when-deploying-to-aws-platforms

## PgBouncer and other poolers

- Run PgBouncer in transaction mode and set `max_prepared_statements` above 0. Add `?pgbouncer=true` only for PgBouncer older than 1.21.0. For Supabase Supavisor, add `?pgbouncer=true`. Other poolers with similar limits can usually use it too. https://www.prisma.io/docs/orm/v7/prisma-client/setup-and-configuration/databases-connections/pgbouncer
- Migrate breaks behind PgBouncer (`prepared statement "s0" already exists`). Default pattern: pooled `DATABASE_URL` for the adapter, direct `DIRECT_URL` for the CLI. https://www.prisma.io/docs/orm/v7/prisma-client/setup-and-configuration/databases-connections/pgbouncer
  ```ts
  // prisma.config.ts
  export default defineConfig({ schema: "prisma/schema.prisma", datasource: { url: env("DIRECT_URL") } });
  // app
  const adapter = new PrismaPg({ connectionString: process.env.DATABASE_URL });
  ```
  https://www.prisma.io/docs/orm/v7/prisma-client/setup-and-configuration/databases-connections/pgbouncer
- The pooler port often differs from 5432 (Supabase transaction pooler: 6543). Neon pools through the `-pooler` host. https://www.prisma.io/docs/orm/v7/core-concepts/supported-databases/postgresql
- Prisma Postgres: app traffic on `pooled.db.prisma.io`, CLI on the direct `db.prisma.io`. https://www.prisma.io/docs/orm/v7/core-concepts/supported-databases/postgresql

## Read replicas

- `@prisma/extension-read-replicas`: build one full `PrismaClient` with its own adapter per replica, then extend the primary. https://www.prisma.io/docs/orm/v7/prisma-client/setup-and-configuration/read-replicas
  ```ts
  const prisma = mainClient.$extends(readReplicas({ replicas: [replicaClient] }));
  ```
  https://www.prisma.io/docs/orm/v7/prisma-client/setup-and-configuration/read-replicas
- Reads go to a random replica. Writes and `$transaction` go to the primary. Use `prisma.$primary()` to run a read against the primary, `$replica()` to run a query against a replica. https://www.prisma.io/docs/orm/v7/prisma-client/setup-and-configuration/read-replicas
