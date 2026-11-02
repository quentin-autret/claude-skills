# Testing

## Contents
- Runner and setup
- Testing module
- Overriding providers and enhancers
- E2E tests
- Request-scoped providers
- Vitest and Jest pitfalls with ESM, SWC and CLI plugins

## Runner and setup

- `nest new` ESM projects use Vitest, CommonJS projects use Jest. `@nestjs/testing` is runner-agnostic. https://docs.nestjs.com/migration-guide#testing-stack https://docs.nestjs.com/fundamentals/testing
- Jest can load the ESM-only v12 packages only on Node.js v24.9 or later (older versions fail with `ERR_REQUIRE_ASYNC_MODULE`). `nest upgrade` bumps Jest to v30. https://docs.nestjs.com/migration-guide#testing-stack
- Unit tests sit next to the class (`.spec` or `.test` suffix), e2e tests in `test/` with an `.e2e-spec.ts` suffix. https://docs.nestjs.com/fundamentals/testing#unit-testing https://docs.nestjs.com/fundamentals/testing#end-to-end-testing

## Testing module

```typescript
const moduleRef = await Test.createTestingModule({
  controllers: [CatsController],
  providers: [CatsService],
}).compile();
catsService = moduleRef.get(CatsService);
```
https://docs.nestjs.com/fundamentals/testing#testing-utilities

- `get()` returns static instances only. Use `resolve()` for transient or request-scoped providers (each call gives a new sub-tree). https://docs.nestjs.com/fundamentals/testing#testing-utilities
- `.useMocker((token) => ...)` mocks every missing dependency. `REQUEST` and `INQUIRER` cannot be auto-mocked: override them. https://docs.nestjs.com/fundamentals/testing#auto-mocking
- After `compile()` alone, `HttpAdapterHost#httpAdapter` is `undefined`. Call `createNestApplication()` if code needs it during init. https://docs.nestjs.com/fundamentals/testing#end-to-end-testing
- Only error logs reach the console by default. Pass a logger with `setLogger()`. https://docs.nestjs.com/fundamentals/testing#end-to-end-testing

## Overriding providers and enhancers

- `overrideProvider()`, `overrideGuard()`, `overrideInterceptor()`, `overrideFilter()`, `overridePipe()` take `useClass`, `useValue` or `useFactory`. `overrideModule(M).useModule(Alt)` swaps a module. https://docs.nestjs.com/fundamentals/testing#end-to-end-testing
- A global enhancer registered with `APP_GUARD` (and other `APP_*`) needs extra steps to be overridden. Register it with `useExisting` plus the class as a provider: https://docs.nestjs.com/fundamentals/testing#overriding-globally-registered-enhancers
  ```typescript
  providers: [{ provide: APP_GUARD, useExisting: JwtAuthGuard }, JwtAuthGuard]
  // test
  .overrideProvider(JwtAuthGuard).useClass(MockAuthGuard)
  ```
- ORM tokens for mocks: `getRepositoryToken(User)` (TypeORM, MikroORM), `getModelToken(Cat.name)` (Mongoose), `getModelToken(User)` (Sequelize), `getDrizzleToken()` (Drizzle), the `PrismaService` class itself (Prisma). https://docs.nestjs.com/data/typeorm#testing https://docs.nestjs.com/data/mikroorm#testing https://docs.nestjs.com/data/sequelize#testing https://docs.nestjs.com/data/mongodb#testing https://docs.nestjs.com/data/drizzle#testing https://docs.nestjs.com/data/prisma#testing
- A Drizzle query builder mock must implement every method in the chain your code calls. Beyond simple queries mock your own repositories or use a real database. https://docs.nestjs.com/data/drizzle#testing

## E2E tests

- `moduleRef.createNestApplication()` needs an explicit `await app.init()`. Close with `await app.close()` in `afterAll`. https://docs.nestjs.com/fundamentals/testing#end-to-end-testing
- Fastify: `createNestApplication<NestFastifyApplication>(new FastifyAdapter())`, `await app.init()`, `await app.getHttpAdapter().getInstance().ready()`, then `app.inject({ method, url })`. https://docs.nestjs.com/fundamentals/testing#end-to-end-testing
- Webhook receivers need the raw body in tests too: `moduleRef.createNestApplication({ rawBody: true })`. https://docs.nestjs.com/http/webhooks#testing
- Passing a Drizzle `db` instance to `forRoot()` shares one instance across every test app, and the first app to close closes it. Create the db in `forRootAsync()` or set `autoCloseConnection: false`. https://docs.nestjs.com/data/drizzle#registering-the-database
- The first-party reliability packages (outbox, workflows, locks) override the Drizzle database with an in-process PGlite (`@electric-sql/pglite`) one: `overrideProvider(getDrizzleToken()).useValue(db)`. PGlite has a single connection, so test races between instances against a PostgreSQL server. https://docs.nestjs.com/reliability/outbox#testing https://docs.nestjs.com/reliability/workflows https://docs.nestjs.com/reliability/locks

## Request-scoped providers

```typescript
const contextId = ContextIdFactory.create();
vi.spyOn(ContextIdFactory, 'getByRequest').mockImplementation(() => contextId);
catsService = await moduleRef.resolve(CatsService, contextId);
```
https://docs.nestjs.com/fundamentals/testing#testing-request-scoped-instances

## Vitest and Jest pitfalls with ESM, SWC and CLI plugins

- In ESM e2e tests write `import request from 'supertest'`. `import * as request from 'supertest'` is not callable. https://docs.nestjs.com/recipes/swc#update-imports-in-e2e-tests
- Vitest with SWC: `unplugin-swc`, `plugins: [swc.vite({ module: { type: 'es6' } })]` (set `module` explicitly so `.swcrc` does not override it). A separate `vitest.config.e2e.ts` includes `**/*.e2e-spec.ts`. https://docs.nestjs.com/recipes/swc#configuration
- Vitest does not resolve TypeScript path aliases: add `resolve.alias: { src: resolve(import.meta.dirname, './src') }`. https://docs.nestjs.com/recipes/swc#path-aliases
- Jest with SWC: `@swc/jest` transform plus `jsc.transform.legacyDecorator` and `decoratorMetadata` in `.swcrc`. https://docs.nestjs.com/recipes/swc#jest--swc
- Monorepo libraries: `tsconfig` `paths` are compiler-only. Mirror each alias in Jest `moduleNameMapper` or imports fail at test time. https://docs.nestjs.com/cli/libraries#using-libraries
- `ts-jest` compiles without the Nest CLI, so the `@nestjs/swagger` and `@nestjs/graphql` CLI plugins are not applied. The GraphQL symptom is `Object type <name> must define one or more fields.` Register an AST transformer (`name`, `version`, `factory`) under `astTransformers.before` in the ts-jest `transform` entry, and bump `version` whenever its config changes. https://docs.nestjs.com/graphql/cli-plugin#integration-with-ts-jest-e2e-tests https://docs.nestjs.com/openapi/cli-plugin#integration-with-ts-jest-e2e-tests
- If Jest keeps stale plugin output: `npx jest --clearCache`. https://docs.nestjs.com/openapi/cli-plugin#troubleshooting-jest-e2e-tests
- Fake only what you need: `vi.useFakeTimers({ toFake: ['setTimeout', 'clearTimeout', 'Date'] })` keeps real I/O for supertest. https://docs.nestjs.com/reliability/resilience#testing
- `enableShutdownHooks()` starts listeners, so running multiple Nest apps in one Node.js process (for example parallel test runs) can trigger Node's too-many-listeners warning. https://docs.nestjs.com/fundamentals/lifecycle-events#application-shutdown
