# Data: Drizzle and Prisma

## Contents
- Drizzle (@nestjs/drizzle)
- Prisma 7
- Shared rules for all ORMs

## Drizzle (@nestjs/drizzle)

- The chapter uses Drizzle ORM v1 (`drizzle-orm@rc`, `drizzle-kit@rc`). `@nestjs/drizzle` also supports Drizzle 0.35 and later, where only relations and relational queries differ. https://docs.nestjs.com/data/drizzle
- No entity classes and no `forFeature()`: tables are imported objects, the database is global and injected with `@InjectDrizzle()`. Row types come from `typeof users.$inferSelect`. https://docs.nestjs.com/data/drizzle#schema https://docs.nestjs.com/data/drizzle#queries
- To read the connection string through `ConfigService`, build the options in a `forRootAsync()` factory (the factory runs once for each application): https://docs.nestjs.com/data/drizzle#async-configuration
  ```typescript
  DrizzleModule.forRootAsync({
    inject: [ConfigService],
    useFactory: (config: ConfigService) => ({
      drizzle, connection: config.getOrThrow<string>('DATABASE_URL'), relations,
    }),
  })
  ```
- `forRoot()` options are evaluated when `AppModule` is imported, so `process.env.DATABASE_URL` must already be set. https://docs.nestjs.com/data/drizzle#registering-the-database
- Passing a ready `db` instance to `forRoot()` shares it between every app built from `AppModule`, and the first app to close closes it (breaks e2e suites). Escape hatch: `autoCloseConnection: false` and close it yourself. https://docs.nestjs.com/data/drizzle#registering-the-database
- `autoCloseConnection` defaults to `true`: the client closes on `app.close()` or on signals with shutdown hooks. https://docs.nestjs.com/data/drizzle#registering-the-database
- Pass `relations` to enable `db.query`, and declare the type once: `export type Database = NodePgDatabase<typeof relations>`. Relations drive relational queries only, foreign keys come from `references()`. https://docs.nestjs.com/data/drizzle#relations
- Inside `db.transaction(async (tx) => ...)` every query must go through `tx`. Repository methods using the injected `db` run on another pool connection, outside the transaction: pass `tx` or use the `nestjs-cls` transactional plugin. https://docs.nestjs.com/data/drizzle#custom-repositories
- A query runs only when awaited. https://docs.nestjs.com/data/drizzle#queries
- Running `migrate()` in the async factory makes every instance migrate at startup. With several instances, run `drizzle-kit migrate` as a release step. Drizzle Kit runs outside Nest and reads `.env` itself. https://docs.nestjs.com/data/drizzle#migrations
- Several databases need distinct `name`s (`@InjectDrizzle('analytics')`, `getDrizzleToken('analytics')`). Unnamed ones share the `default` token. https://docs.nestjs.com/data/drizzle#multiple-databases

## Prisma 7

- There is no Nest package: wrap the generated client in a provider. The chapter targets Prisma ORM 7 on SQL databases (MongoDB stays on Prisma 6). https://docs.nestjs.com/data/prisma
- Install `prisma@7` and `@prisma/client@7` explicitly and keep them on the same major. The docs warn that the npm `latest` tag of `prisma` can point to a pre-release of another major without `migrate` and `generate`. Run the CLI with `npx`. https://docs.nestjs.com/data/prisma#installation
- Prisma 7 requires a driver adapter or a Prisma Accelerate URL (`@prisma/adapter-pg`, `@prisma/adapter-mariadb`, `@prisma/adapter-mssql`, `@prisma/adapter-better-sqlite3`). https://docs.nestjs.com/data/prisma#prismaservice
  ```typescript
  @Injectable()
  export class PrismaService extends PrismaClient implements OnModuleInit, OnModuleDestroy {
    constructor(configService: ConfigService) {
      const adapter = new PrismaPg({ connectionString: configService.getOrThrow<string>('DATABASE_URL') });
      super({ adapter });
    }
    async onModuleInit() { await this.$connect(); }
    async onModuleDestroy() { await this.$disconnect(); }
  }
  ```
- Generator `provider = "prisma-client"` with an `output` (relative to the schema), such as `../src/generated/prisma`. Import the client from `./generated/prisma/client.js`. `prisma init` without `--output` writes outside `src`. https://docs.nestjs.com/data/prisma#set-the-generator-output-path
- The URL is no longer in the schema datasource. The Prisma config file (`prisma7.config.ts` from 7.10, `prisma.config.ts` before) reads `DATABASE_URL`. Nest does not load `.env` itself: register `ConfigModule.forRoot({ isGlobal: true })`. https://docs.nestjs.com/data/prisma#installation https://docs.nestjs.com/data/prisma#set-the-database-connection
- For CommonJS projects set `moduleFormat = "cjs"` in the generator. Releases before 7.10 generate ESM whenever `module` is `nodenext`. https://docs.nestjs.com/data/prisma#configure-the-module-format
- The generated client is gitignored: run `npx prisma generate` in CI and fresh checkouts, and after every schema change (`migrate dev` does not generate). https://docs.nestjs.com/data/prisma#generate-prisma-client
- `$connect()` in `onModuleInit()` fails misconfiguration at startup (otherwise the client connects on the first query). `$disconnect()` also ends the adapter pool. Enable shutdown hooks so it runs on `SIGTERM`. https://docs.nestjs.com/data/prisma#connection-lifecycle
- Generated types are compile-time only: validate bodies with DTO classes or schemas, not Prisma types. https://docs.nestjs.com/data/prisma#querying-from-services
- In `$transaction(async (tx) => ...)`, queries through `this.prisma` run outside the transaction: use `tx`. https://docs.nestjs.com/data/prisma#transactions The array form `$transaction([q1, q2])` cannot include an `outbox.add()`. https://docs.nestjs.com/reliability/outbox#keep-messages-in-your-database
- Conditional writes for stores (locks, idempotency): `updateMany()`/`deleteMany()` with the owner and expiry in `where`, and check `count === 1`. `upsert()` has no update condition, so use `$queryRaw` there. https://docs.nestjs.com/reliability/locks#the-store-contract https://docs.nestjs.com/reliability/idempotency#the-store-contract
- Tests: `{ provide: PrismaService, useValue: prismaMock }` (the class is the token), or `overrideProvider(PrismaService)`. https://docs.nestjs.com/data/prisma#testing

## Old patterns

| Old | Current |
| --- | --- |
| `this.$on('beforeExit', ...)` | Nest lifecycle hooks plus `enableShutdownHooks()` (not supported in Prisma 7) |
| `url = env("DATABASE_URL")` in the schema | URL in the Prisma config file, adapter in the constructor |

https://docs.nestjs.com/data/prisma#connection-lifecycle https://docs.nestjs.com/data/prisma#set-the-generator-output-path

## Shared rules for all ORMs

- TypeORM and Drizzle migrations run through their own CLI outside Nest: no DI or other Nest features in migrations. https://docs.nestjs.com/data/typeorm#migrations https://docs.nestjs.com/data/drizzle#migrations
- With TypeORM, several data sources need unique names, also placed next to `useFactory` in `forRootAsync()`. Unnamed connections all become `default` and override each other. https://docs.nestjs.com/data/typeorm#multiple-databases
- With MikroORM, the connection closes on signals only with `app.enableShutdownHooks()`. https://docs.nestjs.com/data/mikroorm#application-shutdown
