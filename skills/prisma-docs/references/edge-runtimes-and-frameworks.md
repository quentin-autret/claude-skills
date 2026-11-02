# Edge, Bun, Deno and framework integration

## Contents
- Edge driver compatibility
- Cloudflare Workers
- Bun
- Deno
- NestJS
- Next.js

## Edge driver compatibility

- Edge runtimes have limited Node APIs and no free TCP. Support on Cloudflare and Vercel Edge is Preview and needs a compatible driver. Deno Deploy works only through the Prisma Postgres serverless driver. https://www.prisma.io/docs/orm/v7/prisma-client/deployment/edge/overview

| Driver | Cloudflare Workers | Vercel Edge |
| --- | --- | --- |
| Neon serverless (HTTP) | yes | yes |
| PlanetScale serverless (HTTP) | yes | yes |
| `pg` (TCP through `connect()`) | yes | no |
| `@libsql/client` (Turso) | yes | yes |
| D1 | yes | no |
| `@prisma/ppg` (HTTP, WebSocket) | yes | yes |

https://www.prisma.io/docs/orm/v7/prisma-client/deployment/edge/overview (the `@prisma/ppg` package name: https://www.prisma.io/docs/guides/v7/deployment/cloudflare-workers)

- Size limits: 3 MB for Cloudflare Workers on the free plan, 1 MB on Vercel free accounts. https://www.prisma.io/docs/orm/v7/prisma-client/deployment/edge/deploy-to-cloudflare https://www.prisma.io/docs/orm/v7/prisma-client/deployment/edge/deploy-to-vercel

## Cloudflare Workers

- Generator `runtime = "cloudflare"` (or `"workerd"`). With `pg`, add `compatibility_flags: ["nodejs_compat"]` and a `compatibility_date` in `wrangler.jsonc`. `@prisma/ppg` needs no flag. https://www.prisma.io/docs/guides/v7/deployment/cloudflare-workers
- Unlike a Node server, create a new adapter and client per request, then release it: https://www.prisma.io/docs/guides/v7/deployment/cloudflare-workers
  ```ts
  const prisma = new PrismaClient({ adapter: new PrismaPg({ connectionString: env.DATABASE_URL }) });
  const users = await prisma.user.findMany();
  ctx.waitUntil(prisma.$disconnect());
  ```
  https://www.prisma.io/docs/orm/v7/prisma-client/deployment/edge/deploy-to-cloudflare
- `.dev.vars` is not compatible with the `.env` files Prisma reads, so expose it to the CLI: run `dotenv -e .dev.vars -- npx prisma ...`. Production secret: `npx wrangler secret put DATABASE_URL`, https://www.prisma.io/docs/orm/v7/prisma-client/deployment/edge/deploy-to-cloudflare
- Generate the Worker env types with `wrangler types --no-strict-vars` before `wrangler dev`. https://www.prisma.io/docs/guides/v7/deployment/cloudflare-workers
- PlanetScale on Workers: `@prisma/adapter-planetscale`, `relationMode = "prisma"`, remove the `cache` field from the fetch init (`delete init["cache"]`), and the guide creates tables with `prisma db push`. https://www.prisma.io/docs/orm/v7/prisma-client/deployment/edge/deploy-to-cloudflare
- `pg` does not work with `@cloudflare/next-on-pages`: use the Neon or PlanetScale drivers. https://www.prisma.io/docs/orm/v7/prisma-client/deployment/edge/deploy-to-cloudflare
- Filter `/favicon.ico` in the handler: browsers send it automatically and it can clutter the logs. https://www.prisma.io/docs/guides/v7/deployment/cloudflare-workers

## Bun

- The guide runs `bunx --bun prisma init`: the `--bun` flag keeps the CLI on Bun instead of falling back to Node because of its `#!/usr/bin/env node` shebang. Seed: `seed: "bun run prisma/seed.ts"`. Bun loads `.env`, so drop the `dotenv/config` import. https://www.prisma.io/docs/guides/v7/runtimes/bun
- `bun build --compile index.ts` produces a single executable that needs neither Bun nor Node on the target. https://www.prisma.io/docs/guides/v7/runtimes/bun

## Deno

- Generator `runtime = "deno"` is required. `deno.json` needs `"nodeModulesDir": "auto"`, `compilerOptions.types: ["node"]`, `npm:` import mappings for `@prisma/client`, `@prisma/adapter-pg` and `prisma`, and tasks with `--env=.env -A`. https://www.prisma.io/docs/guides/v7/runtimes/deno
- Import the generated client with its `.ts` extension (`./generated/prisma/client.ts`) and read env with `Deno.env.get`. https://www.prisma.io/docs/guides/v7/runtimes/deno

## NestJS

- The guide makes the project ESM and imports with `.js` suffixes (`./generated/prisma/client.js`). `PrismaService extends PrismaClient` and builds the adapter in its constructor before `super({ adapter })`. https://www.prisma.io/docs/guides/v7/frameworks/nestjs
- Nest does not load `.env` by itself: register `ConfigModule.forRoot()` or `DATABASE_URL` is missing. https://www.prisma.io/docs/guides/v7/frameworks/nestjs
- Scaffold: `npm create prisma@stable -- --template nest`. https://www.prisma.io/docs/guides/v7/frameworks/nestjs
- SQL comment tags in Nest go through a global interceptor: `from(withQueryTags(tags, () => lastValueFrom(next.handle())))`. https://www.prisma.io/docs/orm/v7/prisma-client/observability-and-logging/sql-comments

## Next.js

- Put the client in `lib/prisma.ts` with the `globalThis` singleton (cached only outside production) to survive hot reload. https://www.prisma.io/docs/guides/v7/frameworks/nextjs
- The guide queries directly in async page functions (`const users = await prisma.user.findMany()` inside `export default async function Home()`). Add `"postinstall": "prisma generate"` for Vercel. https://www.prisma.io/docs/guides/v7/frameworks/nextjs
- The guide's sample `package.json` still shows a `"prisma": { "seed": ... }` block: the seed belongs in `migrations.seed` of the config. https://www.prisma.io/docs/guides/v7/frameworks/nextjs
- Scaffold: `npm create prisma@stable -- --template next`. https://www.prisma.io/docs/guides/v7/frameworks/nextjs
