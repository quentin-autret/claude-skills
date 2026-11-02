# Observability: OpenTelemetry and NestJS Observe

## Contents
- Two options
- OpenTelemetry
- NestJS Observe SDK
- Trace propagation
- Choosing

## Two options

The docs describe OpenTelemetry, which is vendor-neutral and self-hostable, and NestJS Observe (`@nestjs/observe`), a hosted, paid service with a free tier that hooks in through the `instrument` option of `NestFactory.create()`. Both can run in the same process. https://docs.nestjs.com/observability/opentelemetry#what-nestjs-observe-adds https://docs.nestjs.com/observability/overview

## OpenTelemetry

- Instrumentation must load before Nest: put it on the `node` command line (or `NODE_OPTIONS`), not in `main.ts`. https://docs.nestjs.com/observability/opentelemetry#zero-code-setup
  ```bash
  # ES modules (default for nest new on v12)
  node --experimental-loader=@opentelemetry/instrumentation/hook.mjs \
    --import @opentelemetry/auto-instrumentations-node/register dist/main.js
  # CommonJS
  node --require @opentelemetry/auto-instrumentations-node/register dist/main.js
  ```
- On Nest 12, CommonJS builds also need the loader hook: `@nestjs/*` packages are ES modules that import Express themselves, so without it Express is not instrumented and HTTP spans lack the route. In a CommonJS build you can instead import an instrumentation file on the first line of `main.ts`, but that is not enough for ES module builds or for Express on Nest 12. https://docs.nestjs.com/observability/opentelemetry#configuring-the-sdk-in-code
- The docs warn that `@opentelemetry/instrumentation-nestjs-core` does not support `@nestjs/core` 12: its Nest-level spans are not recorded and traces hold HTTP and database spans only. Check the supported range of the release you install. https://docs.nestjs.com/observability/opentelemetry#what-gets-recorded-for-nest
- Spans stop at the controller: guards, interceptors, pipes and providers are not traced automatically. On Fastify, route spans need `@fastify/otel`. https://docs.nestjs.com/observability/opentelemetry#what-gets-recorded-for-nest
- Manual spans: end every span, including on failure. `@opentelemetry/api` is a no-op without an SDK. https://docs.nestjs.com/observability/opentelemetry#adding-your-own-spans
  ```typescript
  const tracer = trace.getTracer('orders-api');
  return tracer.startActiveSpan('PricingService.quote', async (span) => {
    try { return await this.calculate(cart); }
    catch (error) { span.recordException(error as Error); span.setStatus({ code: SpanStatusCode.ERROR }); throw error; }
    finally { span.end(); }
  });
  ```
- With a `NodeSDK` in code, call `otelSdk.shutdown()` after Nest shuts down (in `onApplicationShutdown()` with shutdown hooks) to flush the last batch. `OTLPTraceExporter` defaults to `http://localhost:4318`. https://docs.nestjs.com/observability/opentelemetry#configuring-the-sdk-in-code

## NestJS Observe SDK

- Wiring takes three edits: https://docs.nestjs.com/observability/sdk#quick-start
  ```typescript
  export const { ObserveModule, ObserveInstrument } = createObserveModule();
  // AppModule imports
  ObserveModule.forRoot({ appKey: process.env.OBSERVE_APP_KEY!, appSecret: process.env.OBSERVE_APP_SECRET!, serviceId: 'cats-app' })
  // main.ts
  const app = await NestFactory.create(AppModule, { instrument: ObserveInstrument });
  ```
- With an explicit adapter the options are the third argument. Passing them second silently drops `instrument`. https://docs.nestjs.com/observability/sdk#quick-start
- `forRoot()` reads env when `AppModule` is imported: load env first (`--env-file`) or use `forRootAsync()`. `node dist/main` does not load `.env`: set variables on the platform. https://docs.nestjs.com/observability/sdk#reading-the-key-pair-from-the-environment
- `sourceContext`, `traceIdGenerator`, `attachTraceIdToLogs` are options of `createObserveModule()`, not `forRoot()`. https://docs.nestjs.com/observability/sdk#full-configuration-reference
- `sourceContext` is on by default and ships source fragments of your code. Disable with `sourceContext: false` if that is not acceptable. https://docs.nestjs.com/observability/sdk#error-source-context
- `TracerService` methods except `currentTraceId()` and metrics throw outside a traced context. https://docs.nestjs.com/observability/manual-instrumentation
- `attachTraceIdToLogs` only augments the built-in `ConsoleLogger`. With an external logger read `TracerService.currentTraceId()`. https://docs.nestjs.com/application/logger#correlating-logs-with-requests
- `nest new` asks whether to wire Observe (default yes in an interactive terminal, skipped in CI, `--observe`/`--no-observe`). https://docs.nestjs.com/first-steps#setup

## Trace propagation

- HTTP carries `x-request-id` both ways. Outbound propagation covers `fetch`/undici everywhere and `node:http` clients (axios) only on Node 22.12 and later. https://docs.nestjs.com/observability/distributed-tracing#http-to-http
- TCP and Redis transports propagate automatically on `@nestjs/microservices` 12.0.4 or later. gRPC, NATS and others need the id forwarded in metadata or payload and read back in `traceIdGenerator`. In hybrid apps that generator receives both HTTP requests and RPC contexts. https://docs.nestjs.com/observability/distributed-tracing#tcp-redis-nats-and-other-microservice-transports
- Running beside OpenTelemetry: `ObserveModule.forRoot({ openTelemetry: true })` follows W3C `traceparent`. Observe does not export or ingest OTLP spans, only the trace id is shared. Both hook database drivers (tested with `pg`). https://docs.nestjs.com/observability/opentelemetry#running-beside-opentelemetry

## Choosing

- The docs advise OpenTelemetry when vendor neutrality, self-hosted telemetry, or one backend for all services is required. Observe is hosted, Node and NestJS only, and does not record Redis and Kafka client calls as spans. https://docs.nestjs.com/observability/opentelemetry#what-nestjs-observe-adds
- Alternatives for per-request context without Observe: an `AsyncLocalStorage` middleware or `nestjs-cls`. https://docs.nestjs.com/recipes/async-local-storage
