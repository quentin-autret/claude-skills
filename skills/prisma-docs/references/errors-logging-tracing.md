# Errors, logging, tracing and SQL comments

## Contents
- Error classes
- Error codes worth handling
- Error formatting
- Logging and debugging
- OpenTelemetry tracing
- SQL comments

## Error classes

- Catch with `instanceof` against the `Prisma` namespace from the generated client, switch on `code`, rethrow the rest. https://www.prisma.io/docs/orm/v7/prisma-client/debugging-and-troubleshooting/handling-exceptions-and-errors
  ```ts
  import { PrismaClient, Prisma } from "../generated/prisma/client";
  try {
    await prisma.user.create({ data: { email } });
  } catch (e) {
    if (e instanceof Prisma.PrismaClientKnownRequestError && e.code === "P2002") { /* unique violation */ }
    throw e;
  }
  ```
  https://www.prisma.io/docs/orm/v7/prisma-client/debugging-and-troubleshooting/handling-exceptions-and-errors
- `PrismaClientKnownRequestError`: `code`, `meta` (for example `{ target: ["email"] }`). `PrismaClientUnknownRequestError`: no code. `PrismaClientValidationError`: validation failed (for example a missing field or a wrong field type), no `code` property. https://www.prisma.io/docs/orm/v7/reference/error-reference
- `PrismaClientInitializationError` is raised on `$connect()` or on the first query (bad credentials, unreachable host, missing env variable). Its field is `errorCode`, not `code`. https://www.prisma.io/docs/orm/v7/reference/error-reference
- `PrismaClientRustPanicError`: restart the client or the process. https://www.prisma.io/docs/orm/v7/reference/error-reference

## Error codes worth handling

| Code | Meaning |
| --- | --- |
| P2002 | unique constraint failed (`meta.target`) |
| P2003 | foreign key constraint failed |
| P2025 | operation depends on one or more required records that were not found |
| P2034 | write conflict or deadlock, retry the transaction |
| P2024 | timed out fetching a new connection from the connection pool (pool size and timeout are set per driver adapter) |
| P2037 | too many database connections |
| P2000 | value too long for the column |
| P2021 / P2022 | table / column does not exist |
| P1001 / P1010 / P1011 | cannot reach database server / user denied access on the database / error opening a TLS connection |
| P1012 | schema validation error |
| P3005 | database not empty, baseline it |
| P3009 / P3018 | a failed migration is recorded, new migrations are not applied until it is resolved |

https://www.prisma.io/docs/orm/v7/reference/error-reference

## Error formatting

- `errorFormat`: `pretty`, `colorless`, `minimal`. `NODE_ENV=production` switches to minimal automatically, `NO_COLOR` to colorless. https://www.prisma.io/docs/orm/v7/prisma-client/setup-and-configuration/error-formatting

## Logging and debugging

- Log levels `query`, `info`, `warn`, `error`, emitted to `stdout` or as events: https://www.prisma.io/docs/orm/v7/prisma-client/observability-and-logging/logging
  ```ts
  const prisma = new PrismaClient({ adapter, log: [{ emit: "event", level: "query" }, { emit: "stdout", level: "error" }] });
  prisma.$on("query", (e) => console.log(e.query, e.params, e.duration + "ms"));
  ```
  https://www.prisma.io/docs/orm/v7/prisma-client/observability-and-logging/logging
- `duration` runs from the client issuing the query to the database answering, not only execution time. `params` holds the query parameters. For MongoDB, `params` and `duration` are undefined. https://www.prisma.io/docs/orm/v7/reference/prisma-client-reference#log
- Register `$on` before `$extends`: extended clients have no `$on`. https://www.prisma.io/docs/orm/v7/reference/prisma-client-reference#on
- `DEBUG="prisma*"` traces Client and CLI (`prisma:client`, `prisma:engine`). `prisma debug` prints schema path, env and CI detection for bug reports. https://www.prisma.io/docs/orm/v7/prisma-client/debugging-and-troubleshooting/debugging https://www.prisma.io/docs/cli/v7/debug

## OpenTelemetry tracing

- The `tracing` preview feature is only needed for Prisma ORM versions between 4.2.0 and 6.1.0. Install `@prisma/instrumentation` (peer dependency `@opentelemetry/api`) and register it, most simply through `NodeSDK({ instrumentations: [new PrismaInstrumentation()] })`. https://www.prisma.io/docs/orm/v7/prisma-client/observability-and-logging/opentelemetry-tracing
- Register tracing before importing `PrismaClient` and other instrumented modules, or no span appears. https://www.prisma.io/docs/orm/v7/prisma-client/observability-and-logging/opentelemetry-tracing
- Spans include `prisma:client:operation`, `prisma:engine:db_query` (SQL in tags) and `prisma:client:transaction` for interactive transactions. https://www.prisma.io/docs/orm/v7/prisma-client/observability-and-logging/opentelemetry-tracing
- Production: `BatchSpanProcessor` and a sampler such as `TraceIdRatioBasedSampler(0.1)`, and `sdk.shutdown()` on SIGTERM for a graceful shutdown. https://www.prisma.io/docs/orm/v7/prisma-client/observability-and-logging/opentelemetry-tracing

## SQL comments

- The `comments` option appends `/*key='value'*/` to each statement: https://www.prisma.io/docs/orm/v7/prisma-client/observability-and-logging/sql-comments
  ```ts
  new PrismaClient({ adapter, comments: [queryTags(), traceContext()] });
  await withQueryTags({ route, requestId }, () => prisma.user.findMany());
  ```
  https://www.prisma.io/docs/orm/v7/prisma-client/observability-and-logging/sql-comments
- Packages: `@prisma/sqlcommenter-query-tags`, `@prisma/sqlcommenter-trace-context`, `@prisma/sqlcommenter` (types for custom plugins). https://www.prisma.io/docs/orm/v7/prisma-client/observability-and-logging/sql-comments
- A nested `withQueryTags` replaces outer tags, `withMergedQueryTags` merges them. Hono and Koa middleware await downstream handlers, so `withQueryTags(..., () => next())` works there. Express middleware uses callbacks, so wrap each route handler directly. Fastify: wrap the route handler. NestJS: use an interceptor applied with `useGlobalInterceptors`. https://www.prisma.io/docs/orm/v7/prisma-client/observability-and-logging/sql-comments
- `traceparent` is added only when `@prisma/instrumentation` is set up and the span is sampled. `context.sql` exists only with driver adapters. https://www.prisma.io/docs/orm/v7/prisma-client/observability-and-logging/sql-comments
- Plugins run synchronously in array order, later keys override earlier ones, and returning `undefined` for a key does not remove an earlier value. https://www.prisma.io/docs/orm/v7/prisma-client/observability-and-logging/sql-comments
