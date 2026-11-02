# Java SDK and API

## Contents

- Versions and BOMs
- GlobalOpenTelemetry and context
- Spans, attributes, logs
- SDK programmatic setup
- Autoconfigure properties
- OTLP exporters
- Metrics views and limits
- Declarative config, self-monitoring, testing

## Versions and BOMs

- Versions on the pages: opentelemetry-java 1.67.0, instrumentation 2.33.1, contrib 1.61.0, semconv 1.45.0. The core repo releases first each month, then instrumentation, then contrib; semconv has its own schedule. https://opentelemetry.io/docs/languages/java/intro/#repositories
- Use a BOM: `opentelemetry-bom`, `-bom-alpha`, `opentelemetry-instrumentation-bom` or `-instrumentation-bom-alpha`. They are hierarchical, so adding several is redundant and not recommended (it can give unintuitive version resolution). https://opentelemetry.io/docs/languages/java/intro/#dependencies-and-boms

```kotlin
implementation(platform("io.opentelemetry:opentelemetry-bom:1.67.0"))
implementation("io.opentelemetry:opentelemetry-api")
```

https://opentelemetry.io/docs/languages/java/intro/#dependencies-and-boms

- `-alpha` artifacts are unstable (incubator, Prometheus exporter, log appenders, several shims). Libraries should not depend on `opentelemetry-api-incubator` or `opentelemetry-semconv-incubating` (transitive version conflicts can cause runtime errors). https://opentelemetry.io/docs/languages/java/api/#incubating-api
- Default entry point: the Java agent. https://opentelemetry.io/docs/languages/java/instrumentation/

## GlobalOpenTelemetry and context

- If `GlobalOpenTelemetry.get()` runs before `set(..)`, it installs a no-op and a later `set` throws; `set` also throws when called twice. Set it once, early, or avoid the global. https://opentelemetry.io/docs/languages/java/api/#globalopentelemetry
- Under the agent, use `GlobalOpenTelemetry.getOrNoop()`. Native library instrumentation defaults to it and offers a `setOpenTelemetry(...)` override. https://opentelemetry.io/docs/languages/java/api/#globalopentelemetry
- `makeCurrent()` returns a `Scope`; not closing it is an error. Use try-with-resources. https://opentelemetry.io/docs/languages/java/api/#context
- Pool threads lose context unless wrapped: `context.wrap(runnable)`, `context.wrap(executorService)`, `wrapSupplier`, `wrapFunction`. https://opentelemetry.io/docs/languages/java/api/#context

## Spans, attributes, logs

```java
Span span = tracer.spanBuilder("span").startSpan();
try (Scope s = span.makeCurrent()) {
  // work
  span.recordException(new RuntimeException("error")); // a span event
  span.setStatus(StatusCode.OK, "status description");
} finally {
  span.end();
}
```

https://opentelemetry.io/docs/languages/java/api/#span

- Without `makeCurrent()`, children need an explicit `setParent`. https://opentelemetry.io/docs/languages/java/api/#span
- Pre-allocate `AttributeKey` and static `Attributes` constants on hot paths. https://opentelemetry.io/docs/languages/java/api/#attributes
- `LoggerProvider`/`Logger` is the bridge API for appender authors; apps keep Log4j, Logback or SLF4J with an appender, and use the MDC shim for trace correlation. https://opentelemetry.io/docs/languages/java/instrumentation/#log-instrumentation
- Baggage enriches telemetry only when the SDK is configured to add entries as attributes; contrib `BaggageSpanProcessor` (alpha) does it for spans. https://opentelemetry.io/docs/languages/java/sdk/#spanprocessor

## SDK programmatic setup

- Use one `Resource` for all providers: `Resource.getDefault().toBuilder().put(ServiceAttributes.SERVICE_NAME, "my-service").build()`. With autoconfigure, the default `otel.service.name` is `unknown_service:java`. https://opentelemetry.io/docs/languages/java/sdk/#resource , https://opentelemetry.io/docs/languages/java/configuration/#properties-resource
- Default sampler `ParentBased(root=AlwaysOn)`; change with `.setSampler(Sampler.parentBasedBuilder(Sampler.traceIdRatioBased(0.25)).build())`. https://opentelemetry.io/docs/languages/java/sdk/#sampler
- Exporters are paired with processors, typically `BatchSpanProcessor` (`SimpleSpanProcessor` exports each span). Batch defaults: queue 2048, delay 5000 ms, batch 512, timeout 30000 ms; log batch delay 1000 ms. https://opentelemetry.io/docs/languages/java/configuration/#properties-traces
- Span limits: 128 attributes, events, links; no value length limit unless `otel.attribute.value.length.limit`. https://opentelemetry.io/docs/languages/java/configuration/#properties-general
- `LoggingSpanExporter` (`console`) is a debug format; stdout OTLP exporters are experimental. https://opentelemetry.io/docs/languages/java/sdk/#spanexporter
- Autoconfigure registers JVM shutdown hooks; `java.util.logging` output can be lost during them (JDK-8161253). https://opentelemetry.io/docs/languages/java/configuration/#zero-code-sdk-autoconfigure

## Autoconfigure properties

- Default: `AutoConfiguredOpenTelemetrySdk.initialize().getOpenTelemetrySdk()` (artifact `opentelemetry-sdk-extension-autoconfigure`) over hand-built SDKs. https://opentelemetry.io/docs/languages/java/configuration/#zero-code-sdk-autoconfigure
- Property to env var: uppercase, `.` and `-` to `_`. A system property wins over the env var. https://opentelemetry.io/docs/languages/java/configuration/#environment-variables-and-system-properties
- Restricting `otel.java.enabled.resource-providers` without `EnvironmentResourceProvider` makes `OTEL_SERVICE_NAME` and `OTEL_RESOURCE_ATTRIBUTES` ignored. https://opentelemetry.io/docs/languages/java/configuration/#properties-resource
- Customize with `addSamplerCustomizer`, `addResourceCustomizer`, `addSpanExporterCustomizer` and SPIs (`ConfigurableSamplerProvider`, `ResourceProvider`) found via `ServiceLoader`. https://opentelemetry.io/docs/languages/java/configuration/#programmatic-customization

## OTLP exporters

- SDK autoconfigure defaults to `grpc` on 4317; the agent 2.x and Spring starter default to `http/protobuf` on 4318. https://opentelemetry.io/docs/languages/java/configuration/#properties-exporters
- With `http/protobuf`, a per-signal endpoint must include `/v1/<signal>`. Timeout 10000 ms, compression option `gzip` (default none), retries on by default. https://opentelemetry.io/docs/languages/java/configuration/#properties-exporters
- Default sender is OkHttp (pulls Kotlin stdlib). Switch by excluding it and adding `opentelemetry-exporter-sender-jdk` (Java 11+, http/protobuf only). https://opentelemetry.io/docs/languages/java/sdk/#senders
- Rotating tokens: `setHeaders(supplier)`. mTLS: `otel.exporter.otlp.client.key` and `client.certificate`. https://opentelemetry.io/docs/languages/java/sdk/#authentication

## Metrics views and limits

- Views do not merge: two matching views (rename, aggregation) give 2 streams. Use narrow selectors. https://opentelemetry.io/docs/languages/java/sdk/#views
- Cardinality limit 2000 per instrument per cycle (`otel.java.metrics.cardinality.limit`). Export interval 60 s. Exemplar filter `TRACE_BASED`. https://opentelemetry.io/docs/languages/java/sdk/#sdkmeterprovider , https://opentelemetry.io/docs/languages/java/configuration/#properties-metrics
- Memory mode `reusable_data` by default (`otel.java.exporter.memory_mode`, other value `immutable_data`). It applies to the OTLP gRPC/HTTP, OTLP stdout and `PrometheusHttpServer` exporters. https://opentelemetry.io/docs/languages/java/configuration/#properties-exporters
- Histogram buckets: set them with `setExplicitBucketBoundariesAdvice()` when migrating second-scale histograms (defaults are millisecond-scale). https://opentelemetry.io/docs/compatibility/prometheus/client-libraries/#classic-explicit-histogram

## Declarative config, self-monitoring, testing

- SDK: `otel.experimental.config.file` with `opentelemetry-sdk-extension-incubator` (alpha). With a file, env vars, system properties, programmatic customizers and SPIs are all ignored. https://opentelemetry.io/docs/languages/java/configuration/#declarative-configuration
- Internal logs: `java.util.logging`, `io.opentelemetry.level = FINE` to debug. Self-monitoring names: `otel.experimental.sdk.telemetry.version` `legacy` (default) or `latest`. https://opentelemetry.io/docs/languages/java/sdk/#sdk-self-monitoring-metrics
- Tests: `opentelemetry-sdk-testing`, `OpenTelemetryExtension` (JUnit 5) resets telemetry per test. https://opentelemetry.io/docs/languages/java/sdk/#testing

```java
@RegisterExtension static final OpenTelemetryExtension otelTesting = OpenTelemetryExtension.create();
assertThat(otelTesting.getSpans()).satisfiesExactly(s -> assertThat(s).hasName("name"));
```

https://opentelemetry.io/docs/languages/java/sdk/#testing
