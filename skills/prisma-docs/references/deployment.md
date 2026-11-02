# Deployment: builds, CI, Docker, serverless

## Contents
- Builds and the generated client
- Production migrations
- Docker
- Vercel
- AWS
- Monorepos and multiple databases

## Builds and the generated client

- Run `prisma generate` on every deployment, preferably through `"postinstall": "prisma generate"` (the recommended option; prepending it to the build command also works). Platforms that cache dependencies (Vercel, Netlify) otherwise skip the `postinstall` run and ship a stale client ("Prisma has detected that this project was built on Vercel"). https://www.prisma.io/docs/orm/v7/more/troubleshooting/nextjs https://www.prisma.io/docs/orm/v7/prisma-client/deployment/serverless/deploy-to-vercel
- Builds that skip devDependencies fail (`prisma: command not found` on Vercel, "@prisma/client did not initialize yet" on Elastic Beanstalk): move `prisma` to `dependencies`. https://www.prisma.io/docs/orm/v7/prisma-client/deployment/serverless/deploy-to-vercel https://www.prisma.io/docs/orm/v7/prisma-client/deployment/caveats-when-deploying-to-aws-platforms
- Paradigm choice: a continuously running server suits consistent traffic, serverless suits sudden spikes. https://www.prisma.io/docs/orm/v7/prisma-client/deployment/deploy-prisma
- Edge runtimes (V8 isolates) lack many Node.js APIs and cannot freely open TCP connections, so only edge-compatible drivers work there. https://www.prisma.io/docs/orm/v7/prisma-client/deployment/edge/overview

## Production migrations

- Run `prisma migrate deploy` as part of the CI/CD pipeline. Vercel example, in the order generate, migrate deploy, build: `"vercel-build": "prisma generate && prisma migrate deploy && next build"`. https://www.prisma.io/docs/orm/v7/prisma-client/deployment/serverless/deploy-to-vercel
- Optional lock-risk check on PostgreSQL before deploying: `npx @flvmnt/pgfence analyze --ci --max-risk medium prisma/migrations/**/migration.sql`. https://www.prisma.io/docs/orm/v7/prisma-client/deployment/deploy-database-changes-with-prisma-migrate
- Vercel Preview deployments share the production `DATABASE_URL` by default, so a pull request carrying a migration alters production. Set a Preview-only `DATABASE_URL` pointing to another database. https://www.prisma.io/docs/orm/v7/prisma-client/deployment/serverless/deploy-to-vercel

## Docker

- `node:alpine` (musl) or `node:slim` (glibc) both work. On Alpine never install glibc (`libc6-compat`): it stops Prisma from running. Some older slim images come without `libssl`, so the guide installs it with `apt-get install -y openssl`. https://www.prisma.io/docs/guides/v7/deployment/docker
- Copy `package.json` and the lockfile, `npm ci`, then copy the sources. The guide's container command runs `npm run db:deploy` (`npx prisma migrate deploy && npx prisma generate`) before `npm run dev`. The guide installs `prisma` as a dev dependency. https://www.prisma.io/docs/guides/v7/deployment/docker
- In compose, gate the app on a Postgres `healthcheck` (`pg_isready`) with `depends_on: condition: service_healthy`, and use the service hostname in the URL (`@postgres_db:5432`), not `localhost`. https://www.prisma.io/docs/guides/v7/deployment/docker
- Studio as a service: `npx prisma studio --port 5555 --browser none`. https://www.prisma.io/docs/guides/v7/deployment/docker

## Vercel

- Node.js is the default and recommended function runtime. Edge is opt-in (`export const runtime = "edge"`). https://www.prisma.io/docs/orm/v7/prisma-client/deployment/edge/deploy-to-vercel
- Fluid compute: attach the `pg` pool so idle connections close before the function suspends: https://www.prisma.io/docs/orm/v7/prisma-client/deployment/serverless/deploy-to-vercel
  ```ts
  const pool = new Pool({ connectionString: process.env.POSTGRES_URL });
  attachDatabasePool(pool);
  export const prisma = new PrismaClient({ adapter: new PrismaPg(pool) });
  ```
  https://www.prisma.io/docs/orm/v7/prisma-client/deployment/serverless/deploy-to-vercel
- For a database created with Vercel Postgres (a Neon database), use `@prisma/adapter-neon`, point `prisma.config.ts` at the direct URL (`POSTGRES_URL_NON_POOLING`) and the adapter at the pooled one (`POSTGRES_PRISMA_URL`). https://www.prisma.io/docs/orm/v7/prisma-client/deployment/edge/deploy-to-vercel
- `pg` does not run on Vercel Edge: use the Neon, PlanetScale, libSQL or Prisma Postgres serverless drivers there. https://www.prisma.io/docs/orm/v7/prisma-client/deployment/edge/overview

## AWS

- RDS Proxy brings no pooling benefit (prepared statements pin sessions). https://www.prisma.io/docs/orm/v7/prisma-client/deployment/caveats-when-deploying-to-aws-platforms
- RDS `P1010` comes from strict SSL. `?sslmode=no-verify` skips certificate checks and the docs warn it may not meet production needs: configure a valid certificate instead. https://www.prisma.io/docs/orm/v7/prisma-client/deployment/caveats-when-deploying-to-aws-platforms
- Elastic Beanstalk also needs `.npmrc` with `unsafe-perm=true` so `postinstall` can write the client. https://www.prisma.io/docs/orm/v7/prisma-client/deployment/caveats-when-deploying-to-aws-platforms
- Lambda: AWS SAM has no `.env` support (use Parameters, SSM or Secrets Manager). Serverless Framework uses `serverless-dotenv-plugin`. SST: pass `Config` secrets to the adapter. https://www.prisma.io/docs/orm/v7/prisma-client/deployment/serverless/deploy-to-aws-lambda
- Instantiate the client outside the handler and never `$disconnect()` per invocation. https://www.prisma.io/docs/orm/v7/more/best-practices

## Monorepos and multiple databases

- Put the schema in a shared package with a custom `output`, install Prisma at the root to avoid version conflicts, gitignore the generated folder. https://www.prisma.io/docs/orm/v7/more/troubleshooting/nextjs https://www.prisma.io/docs/guides/v7/deployment/turborepo
- pnpm: a `packages/database` package that re-exports the client, consumed as `"database": "workspace:*"`, Prisma pinned with a pnpm `catalog`. Order: `db:deploy`, then `db:generate`, then the app build. https://www.prisma.io/docs/guides/v7/deployment/pnpm-workspaces
- Turborepo: `globalEnv: ["DATABASE_URL"]`, `db:generate`/`db:migrate`/`db:deploy` with `cache: false`, and `build`/`dev` with `dependsOn: ["^db:generate"]` so a fresh checkout builds. https://www.prisma.io/docs/guides/v7/deployment/turborepo
- Several databases: one schema folder, one config, one `output` and one env variable per database. Every command passes `--config ./<dir>/prisma.config.ts`, `postinstall` generates every client, each client is memoized under its own key on the global object. https://www.prisma.io/docs/guides/v7/database/multiple-databases
- Multi-tenant client factories must manage the lifecycle of the clients they create, to avoid resource exhaustion. https://www.prisma.io/docs/orm/v7/more/troubleshooting/nextjs
