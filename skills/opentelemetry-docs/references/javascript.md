# JavaScript and Node.js

## Contents

- Status and packages
- Node.js SDK setup and load order
- Zero-code for Node.js
- Traces
- Metrics and views
- Exporters
- Context, propagation, resources
- Browser and serverless

## Status and packages

- Traces and metrics stable, logs in development. Browser instrumentation is experimental and mostly unspecified. https://opentelemetry.io/docs/languages/js/
- Supported runtimes: active and maintenance LTS Node.js. TypeScript follows the DefinitelyTyped 2-year window, so a minor release can drop an old TypeScript. https://opentelemetry.io/docs/languages/js/#version-support
- Current trace SDK package is `@opentelemetry/sdk-trace` (TracerProvider, BatchSpanProcessor, ConsoleSpanExporter, TraceIdRatioBasedSampler); metrics in `@opentelemetry/sdk-metrics`. Sampler docs: https://opentelemetry.io/docs/languages/js/sampling/ . Doc package.json versions: api ^1.9.0, sdk-trace ^2.11.0, resources ^2.11.0, semantic-conventions ^1.43.0, exporter-trace-otlp-http ^0.222.0, auto-instrumentations-node ^0.80.0 (from the serverless page). https://opentelemetry.io/docs/languages/js/getting-started/nodejs/ , https://opentelemetry.io/docs/languages/js/serverless/
- Library authors skip SDK initialization. Without an initialized `TracerProvider` (or meter provider) the tracing and metrics APIs use a no-op implementation and generate no data, so initialize the SDK before any other module loads. https://opentelemetry.io/docs/languages/js/instrumentation/#initialize-tracing

## Node.js SDK setup and load order

- The SDK must load before any other module: `node --import ./instrumentation.mjs app.js`. TypeScript: `npx tsx --import ./instrumentation.ts app.ts` (Node.js 20+). https://opentelemetry.io/docs/languages/js/getting-started/nodejs/

```js
import { NodeSDK } from '@opentelemetry/sdk-node';
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-proto';

const sdk = new NodeSDK({
  traceExporter: new OTLPTraceExporter({ url: 'http://localhost:4318/v1/traces' }),
  instrumentations: [getNodeAutoInstrumentations()],
});
sdk.start();
```

https://opentelemetry.io/docs/languages/js/getting-started/nodejs/ , https://opentelemetry.io/docs/languages/js/exporters/#usage-with-nodejs

- Do not combine `--import ./instrumentation.mjs` with `NODE_OPTIONS="--require @opentelemetry/auto-instrumentations-node/register"`: the docs warn that conflicting `--import` or `--require` flags in `NODE_OPTIONS` must be avoided. https://opentelemetry.io/docs/languages/js/getting-started/nodejs/#run-the-instrumented-app
- The docs assume CommonJS output. ESM apps need the loader hook: `node --experimental-loader=@opentelemetry/instrumentation/hook.mjs --require ./instrumentation.js app.js`. https://opentelemetry.io/docs/languages/js/getting-started/nodejs/#run-the-instrumented-app
- Using `@opentelemetry/sdk-trace` without NodeSDK: set spans active in context yourself; `startActiveSpan` will not do it for you there. https://opentelemetry.io/docs/languages/js/instrumentation/#using-sdk-trace-and-manually-propagating-span-context
- Diagnostics in code: `diag.setLogger(new DiagConsoleLogger(), DiagLogLevel.DEBUG)`. https://opentelemetry.io/docs/languages/js/resources/#process--environment-resource-detection

## Zero-code for Node.js

```shell
export OTEL_SERVICE_NAME="your-service-name"
export OTEL_EXPORTER_OTLP_ENDPOINT="your-endpoint"
export OTEL_NODE_RESOURCE_DETECTORS="env,host,os"
export NODE_OPTIONS="--require @opentelemetry/auto-instrumentations-node/register"
node app.js
```

https://opentelemetry.io/docs/zero-code/js/#configuring-the-module

- All resource detectors and all supported instrumentations are on by default. Limit detectors with `OTEL_NODE_RESOURCE_DETECTORS`. https://opentelemetry.io/docs/zero-code/js/configuration/#sdk-resource-detector-configuration
- `OTEL_NODE_ENABLED_INSTRUMENTATIONS="http,express"` and `OTEL_NODE_DISABLED_INSTRUMENTATIONS="fs,grpc"` take names without `@opentelemetry/instrumentation-`. Enabled applies first, then disabled, so a name in both is disabled. https://opentelemetry.io/docs/zero-code/js/configuration/#excluding-instrumentation-libraries
- `OTEL_LOG_LEVEL` default `info` (keep it in production); `debug` is extremely verbose. https://opentelemetry.io/docs/zero-code/js/#troubleshooting

## Traces

- Call `trace.getTracer(name, version)` where needed instead of exporting a shared tracer, to avoid load-order problems. https://opentelemetry.io/docs/languages/js/instrumentation/#acquiring-a-tracer
- Default to `startActiveSpan` so children nest; use `startSpan` for siblings. End every span, error paths included. https://opentelemetry.io/docs/languages/js/instrumentation/#create-spans

```js
tracer.startActiveSpan('rollTheDice', (span) => {
  try { /* work */ }
  catch (err) {
    span.recordException(err);
    span.setStatus({ code: SpanStatusCode.ERROR, message: err.message });
    throw err;
  } finally { span.end(); }
});
```

https://opentelemetry.io/docs/languages/js/instrumentation/#recording-exceptions

- `BatchSpanProcessor` is the default and right choice. `SimpleSpanProcessor` exports per span, adding heavy overhead with network exporters. https://opentelemetry.io/docs/languages/js/instrumentation/#picking-the-right-span-processor
- Span processors go in the constructor: `new TracerProvider({ spanProcessors: [new BatchSpanProcessor({ exporter })] })`. https://opentelemetry.io/docs/languages/js/serverless/

## Metrics and views

- `metrics.getMeter(name, version)` at point of use; give each instrument `description`, `unit`, `valueType`. https://opentelemetry.io/docs/languages/js/instrumentation/#describing-instruments
- Views are passed to `new MeterProvider({ views: [...] })`; selectors `instrumentName` (wildcards like `http*`), `instrumentType`, `meterName`. https://opentelemetry.io/docs/languages/js/instrumentation/#configure-metric-views

```js
const limitAttributesView = { attributeKeys: ['environment'], instrumentName: '*' };
const dropView = { aggregation: { type: AggregationType.DROP }, meterName: 'pubsub' };
```

https://opentelemetry.io/docs/languages/js/instrumentation/#configure-metric-views

## Exporters

- OTLP packages: `@opentelemetry/exporter-trace-otlp-proto` (HTTP/protobuf), `-otlp-http` (HTTP/JSON), `-otlp-grpc`; metrics equivalents. https://opentelemetry.io/docs/languages/js/exporters/#otlp-dependencies
- Jaeger natively supports OTLP: set up the OTLP exporters. Prometheus: enable its OTLP receiver, or `PrometheusExporter` (port 9464, `/metrics`). https://opentelemetry.io/docs/languages/js/exporters/#jaeger , https://opentelemetry.io/docs/languages/js/exporters/#prometheus
- Meta packages (`auto-instrumentations-node`, `-web`) enlarge the dependency graph; list individual instrumentations when you know them. Disable one: `getNodeAutoInstrumentations({ '@opentelemetry/instrumentation-fs': { enabled: false } })`. https://opentelemetry.io/docs/languages/js/libraries/#registration
- Prefer frameworks with native OpenTelemetry (Next.js, SvelteKit). https://opentelemetry.io/docs/languages/js/libraries/#use-natively-instrumented-libraries

## Context, propagation, resources

- Without a context manager `context.active()` is always `ROOT_CONTEXT`. Node: `AsyncHooksContextManager` (from `@opentelemetry/context-async-hooks`; the serverless samples use `AsyncLocalStorageContextManager`); browser: `ZoneContextManager` (from `@opentelemetry/context-zone`). The SDK normally provides the manager. https://opentelemetry.io/docs/languages/js/context/#context-manager , https://opentelemetry.io/docs/languages/js/getting-started/browser/ , https://opentelemetry.io/docs/languages/js/serverless/
- Contexts are immutable: `setValue` returns a new context. `context.with(ctx, fn)` runs `fn` synchronously with `ctx` active. https://opentelemetry.io/docs/languages/js/context/#set-active-context
- Manual propagation: `propagation.inject(context.active(), carrier)` and `propagation.extract(context.active(), carrier)`, only for protocols no instrumentation covers. https://opentelemetry.io/docs/languages/js/propagation/#manual-context-propagation
- Resources: `resourceFromAttributes({ [ATTR_SERVICE_NAME]: 'svc' })`. Setting `resourceDetectors` replaces the defaults: include `envDetector` and `processDetector`. Env values override code values. https://opentelemetry.io/docs/languages/js/resources/#adding-resources-in-code , https://opentelemetry.io/docs/languages/js/resources/#container-resource-detection

## Browser and serverless

- Browser OTLP: no gRPC (use HTTP on 4318), the CSP must allow the endpoint (`connect-src`), and the Collector needs CORS (`cors.allowed_origins` under the http protocol). https://opentelemetry.io/docs/languages/js/exporters/#usage-in-the-browser
- Lambda wrapper loaded with `NODE_OPTIONS=--require lambda-wrapper`; set `disableAwsContextPropagation: true` on `@opentelemetry/instrumentation-aws-lambda`, otherwise X-Ray headers without Active Tracing give a `NonRecordingSpan` and no spans. https://opentelemetry.io/docs/languages/js/serverless/#aws-lambda-wrapper-code
- The serverless samples use old runtimes (`nodejs14.x`) and a mismatched handler name: use a supported runtime and the exported handler. https://opentelemetry.io/docs/languages/js/serverless/#deployment
