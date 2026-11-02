---
name: nestjs-docs
description: Pitfalls, surprising defaults, version 12 changes and recommended patterns from the official NestJS documentation, each backed by its docs.nestjs.com URL. Use when writing, reviewing or upgrading NestJS 12 server code, including migration from NestJS 11. Also use for any task touching NestJS or @nestjs/* packages, such as controllers, providers, modules, dependency injection, guards, pipes, interceptors, exception filters, validation, serialization, authentication, authorization, security headers, CSRF, cookies, file uploads, caching, configuration, logging, scheduling, queues, events, HTTP client, health checks, testing, microservices, WebSockets, GraphQL, OpenAPI, ORMs (Prisma, Drizzle, TypeORM, Sequelize, Mongoose, MikroORM), outbox, webhooks, idempotency, resilience, durable workflows, observability and deployment.
metadata:
  version: "NestJS 12 (latest docs)"
  source: "https://docs.nestjs.com"
  read-date: "2026-11-02"
---

# NestJS documentation

Rules distilled from https://docs.nestjs.com that a model is likely to get wrong: version 12 changes, surprising defaults and documented gotchas. Generic NestJS knowledge is left out. Every rule ends with the docs URL that backs it.

## Version covered

The references follow the latest docs, which describe NestJS 12. The migration reference covers the move from 11 to 12. Check the project's `@nestjs/core` version in `package.json` first: on 11, read the migration reference before applying a rule marked "from v12.x".

## How to navigate

Read only the reference the task needs, from the table at the end. Each reference opens with a contents list, so jump to the section. Follow the docs URL when the full page is needed. Where the docs come from is listed in [SOURCES.md](SOURCES.md).

## Quick start: the pitfalls that bite most

1. From v12.3 the docs recommend `NodeAdapter` from `@nestjs/platform-node` for new apps. Express still runs when no adapter is passed. https://docs.nestjs.com/http/performance#choosing-an-adapter
2. With an explicit adapter, application options are the third argument of `NestFactory.create()`. Options passed second are silently dropped. https://docs.nestjs.com/observability/sdk#quick-start
3. Injecting `@Res()` or `@Next()` turns off interceptors, `@HttpCode()` and `@Header()` for that route. Use `@Res({ passthrough: true })` to set headers or cookies and still return a value. https://docs.nestjs.com/controllers#library-specific-approach
4. The `schema` option of `@Body()`, `@Query()`, `@Param()` only attaches metadata. Bind `StandardSchemaValidationPipe` (globally, for example) or nothing is validated. https://docs.nestjs.com/migration-guide#route-decorator-schemas
5. From v12.1, call `app.useSecurityHeaders()` right after `NestFactory.create()` instead of Helmet. Middleware registered before it answers without headers, and calling it after `listen()` or twice throws. https://docs.nestjs.com/security/helmet#getting-started
6. From v12.1, `app.enableCsrfProtection()` rejects cross-origin writes from `Sec-Fetch-Site` and `Origin`, with no token or cookie. It does not check WebSocket upgrades. https://docs.nestjs.com/security/csrf https://docs.nestjs.com/security/csrf#limitations
7. From v12.1 cookies are built in (`httpAdapter.setCookie()`, `@Cookies()`), and their `maxAge` is in seconds. A millisecond value carried over from Express `res.cookie()` lives 1000 times longer. `httpOnly` and `secure` default to `false`. https://docs.nestjs.com/http/cookies#setting-and-clearing-cookies
8. Call `ScheduleModule.forRoot()` in one module only. Each extra import registers every `@Cron()`, `@Interval()` and `@Timeout()` again. https://docs.nestjs.com/application/task-scheduling#installation
9. Every instance runs every scheduled job. Add `@OnOneInstance()` from `@nestjs/locks` with a shared lock store. https://docs.nestjs.com/application/task-scheduling#running-on-several-instances
10. `@nestjs/http-client` retries by default: 3 attempts for GET, HEAD, OPTIONS, PUT and DELETE. POST and PATCH retry only with `retry: { methods: ['POST'] }`, paired with an idempotency key. https://docs.nestjs.com/application/http-client#retries
11. Terminus 12 removes `HealthIndicator` and `HealthCheckError`. Inject `HealthIndicatorService` and return `up()` or `down()`. A throw reports unhealthy only inside `attempt()`. https://docs.nestjs.com/migration-guide#terminus-module
12. `@Optional()` is not inherited by a redeclared constructor: a subclass with its own constructor throws `UnknownDependenciesException` until it repeats the marker. https://docs.nestjs.com/migration-guide#optional-is-no-longer-inherited-by-a-redeclared-constructor
13. All v12 core packages are ESM-only. CommonJS apps keep working through `require(esm)`, but Jest loads them only on Node.js v24.9 or later (`ERR_REQUIRE_ASYNC_MODULE` before). https://docs.nestjs.com/migration-guide#esm-packages https://docs.nestjs.com/migration-guide#testing-stack
14. Multer's default memory storage has no size limit. Always set `limits.fileSize`, which stops the upload with 413 as it happens, unlike `MaxFileSizeValidator`. https://docs.nestjs.com/http/file-upload#memory-usage-and-storage https://docs.nestjs.com/http/file-upload#file-size-limit

Example for rules 4 and 5, from the docs:

```typescript
const app = await NestFactory.create(AppModule);
app.useSecurityHeaders();
app.enableCsrfProtection();
app.useGlobalPipes(new StandardSchemaValidationPipe());
// controller
create(@Body({ schema: createUserSchema }) body: CreateUserDto)
```

https://docs.nestjs.com/security/helmet#getting-started https://docs.nestjs.com/migration-guide#route-decorator-schemas

## Old patterns

| Old | Current |
| --- | --- |
| Manual `npm-check-updates` bump | `nest upgrade --dry-run`, then `nest upgrade` (https://docs.nestjs.com/migration-guide#upgrading-packages) |
| `app.use(helmet())` | `app.useSecurityHeaders()` (https://docs.nestjs.com/security/helmet#using-helmet-directly) |
| `csrf-csrf` or `@fastify/csrf-protection` tokens | `app.enableCsrfProtection()` (https://docs.nestjs.com/security/csrf#when-you-need-a-token-based-approach) |
| `@nestjs/axios` with `firstValueFrom()` | `@nestjs/http-client`, Promise-based (https://docs.nestjs.com/application/http-client#migrating-from-nestjsaxios) |
| `extends HealthIndicator`, `throw new HealthCheckError()` | `HealthIndicatorService.check(key)` (https://docs.nestjs.com/migration-guide#terminus-module) |
| GraphQL `playground` | a `graphiql` options object (https://docs.nestjs.com/migration-guide#graphql-ide-configuration) |
| `subscriptions-transport-ws` | `subscriptions: { 'graphql-ws': true }` (https://docs.nestjs.com/migration-guide#graphql-subscriptions-transport) |
| `nats` package with `StringCodec`/`JSONCodec` | `@nats-io/transport-node` and `msg.json()` (https://docs.nestjs.com/migration-guide#nats-v3) |
| Joi options directly in `validationOptions` | `validationOptions.libraryOptions` (https://docs.nestjs.com/migration-guide#config-module) |
| CLI webpack builder | `--builder rspack` (https://docs.nestjs.com/migration-guide#webpack-deprecation-in-cli-workflows) |
| Bull (`@nestjs/bull`) | BullMQ (`@nestjs/bullmq`) (https://docs.nestjs.com/application/queues#bull-installation) |

## References

| Reference | Read it when |
| --- | --- |
| [references/migration-11-to-12.md](references/migration-11-to-12.md) | Upgrading from NestJS 11, or code behaves differently after the upgrade |
| [references/http-platform.md](references/http-platform.md) | Choosing an adapter, versioning, cookies, sessions, raw body, SSE, shutdown |
| [references/request-pipeline.md](references/request-pipeline.md) | Writing controllers, middleware, pipes, guards, interceptors, filters, custom decorators |
| [references/dependency-injection.md](references/dependency-injection.md) | Providers, tokens, dynamic modules, scopes, circular dependencies, lifecycle hooks |
| [references/validation-and-serialization.md](references/validation-and-serialization.md) | Choosing a validation pipe, Standard Schema, class-validator, mapped types, serialization |
| [references/configuration-and-logging.md](references/configuration-and-logging.md) | `ConfigModule`, env file precedence, env validation, the built-in logger |
| [references/security.md](references/security.md) | Security headers, CSRF, CORS, rate limiting, hashing |
| [references/authentication.md](references/authentication.md) | Sessions, tokens and API keys with `@nestjs/authentication` |
| [references/authorization.md](references/authorization.md) | Policies and permission checks with `@nestjs/authorization` |
| [references/caching.md](references/caching.md) | `@nestjs/cache-manager`, TTLs, `CacheInterceptor` |
| [references/file-upload-and-storage.md](references/file-upload-and-storage.md) | Uploads, size limits, streaming responses, `@nestjs/storage` disks |
| [references/http-client.md](references/http-client.md) | Calling other services with `@nestjs/http-client`, retries, timeouts |
| [references/events-scheduling-queues.md](references/events-scheduling-queues.md) | Event emitter, cron jobs, BullMQ queues |
| [references/locks-and-health-checks.md](references/locks-and-health-checks.md) | Distributed locks with `@nestjs/locks`, Terminus health checks |
| [references/resilience.md](references/resilience.md) | Timeouts, retries, circuit breakers, bulkheads with `@nestjs/resilience` |
| [references/idempotency.md](references/idempotency.md) | `Idempotency-Key` handling with `@nestjs/idempotency` |
| [references/outbox.md](references/outbox.md) | Publishing events reliably after a commit with `@nestjs/outbox` |
| [references/webhooks.md](references/webhooks.md) | Sending and receiving webhooks with `@nestjs/webhooks` |
| [references/workflows.md](references/workflows.md) | Long-running durable processes with `@nestjs/workflows` |
| [references/cqrs.md](references/cqrs.md) | Commands, queries, events, aggregates and sagas with `@nestjs/cqrs` |
| [references/mail.md](references/mail.md) | Sending mail with `@nestjs/mail` |
| [references/i18n.md](references/i18n.md) | Translations and locale resolution with `@nestjs/i18n` |
| [references/data-drizzle-prisma.md](references/data-drizzle-prisma.md) | Drizzle or Prisma integration, transactions |
| [references/data-typeorm-sequelize-mongoose-mikroorm.md](references/data-typeorm-sequelize-mongoose-mikroorm.md) | TypeORM, Sequelize, Mongoose or MikroORM integration |
| [references/microservices.md](references/microservices.md) | Transports, message patterns, Kafka, gRPC, hybrid apps |
| [references/websockets.md](references/websockets.md) | Gateways, WebSocket authentication, adapters and scaling |
| [references/graphql.md](references/graphql.md) | GraphQL setup, resolvers, subscriptions and types |
| [references/graphql-federation-and-advanced.md](references/graphql-federation-and-advanced.md) | GraphQL mapped types, CLI plugin, directives, plugins, federation |
| [references/openapi.md](references/openapi.md) | Swagger setup, operations, security schemes, the OpenAPI CLI plugin |
| [references/testing.md](references/testing.md) | Unit and e2e tests, overriding providers, Vitest and Jest pitfalls |
| [references/cli-and-build.md](references/cli-and-build.md) | Nest CLI, workspaces, libraries, SWC, hot reload, REPL |
| [references/observability.md](references/observability.md) | OpenTelemetry or NestJS Observe tracing |
| [references/deployment.md](references/deployment.md) | Production builds, standalone apps, serverless, Devtools |
