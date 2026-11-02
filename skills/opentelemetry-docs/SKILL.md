---
name: opentelemetry-docs
description: Teaches current OpenTelemetry practices from the official documentation. Use when instrumenting apps or libraries with the SDKs and APIs (JavaScript, Python, Go, Java), configuring the Collector (pipelines, connectors, OTTL), exporting over OTLP, sampling, context propagation, baggage, semantic conventions, Kubernetes Operator or Helm, zero-code agents, OTEL_* variables or declarative YAML, securing a Collector, or migrating from OpenTracing, OpenCensus or Jaeger. Triggers on OpenTelemetry, OTel, OTLP, otelcol, OTEL_SERVICE_NAME, TracerProvider, NodeSDK, tail_sampling, traceparent.
metadata:
  version: "OpenTelemetry documentation as published on opentelemetry.io, tracking latest releases (no version selector)"
  source: https://opentelemetry.io/docs/
  fetched: "2026-11-02"
---

# OpenTelemetry

This skill covers the OpenTelemetry documentation at https://opentelemetry.io/docs/: concepts, language SDKs, zero-code instrumentation, the Collector, Kubernetes and security. It holds surprising defaults, pitfalls, deprecated components and recommended patterns. Each claim ends with its source page.

## Version covered

The site documents the latest releases and has no version selector. Component names and stability levels change between Collector releases: confirm them with `otelcol components` on the target binary. https://opentelemetry.io/docs/collector/troubleshooting/#check-collector-components

## How to navigate

1. Read the quick rules below. They catch the most frequent mistakes.
2. Open the one reference file that matches the task, from the table at the end.
3. For code using old names (`logging` exporter, Jaeger exporter, `service::telemetry::metrics::address`), read `references/compatibility-and-migration.md` first.

## Quick start

Zero-code Node.js service exporting to a local Collector over OTLP/HTTP:

```shell
npm install --save @opentelemetry/api @opentelemetry/auto-instrumentations-node
export OTEL_SERVICE_NAME="checkout"
export OTEL_EXPORTER_OTLP_PROTOCOL="http/protobuf"
export OTEL_EXPORTER_OTLP_ENDPOINT="http://localhost:4318"
export NODE_OPTIONS="--require @opentelemetry/auto-instrumentations-node/register"
node app.js
```

https://opentelemetry.io/docs/zero-code/js/#configuring-the-module , https://opentelemetry.io/docs/languages/sdk-configuration/otlp-exporter/#otel_exporter_otlp_protocol

Minimal Collector receiving OTLP and forwarding it:

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: localhost:4317
      http:
        endpoint: localhost:4318
processors:
  memory_limiter:
    check_interval: 1s
    limit_mib: 512
exporters:
  otlp_grpc:
    endpoint: otelcol:4317
    sending_queue:
      batch:
service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter]
      exporters: [otlp_grpc]
```

https://opentelemetry.io/docs/collector/configuration/#basics

## Quick rules

### Instrumentation

- Always set `OTEL_SERVICE_NAME`. Without it `service.name` is `unknown_service`. https://opentelemetry.io/docs/languages/sdk-configuration/general/#otel_service_name
- Libraries depend on the API only, never the SDK. Without an SDK every API call is a silent no-op. https://opentelemetry.io/docs/concepts/instrumentation/libraries/#opentelemetry-api
- Initialize the SDK before any other module loads (Node.js `--import ./instrumentation.mjs`). https://opentelemetry.io/docs/languages/js/getting-started/nodejs/
- `Unset` span status already means success. Set `Error` on failure and record the exception too; in Go `RecordError` does not set the status. https://opentelemetry.io/docs/concepts/signals/traces/#span-status , https://opentelemetry.io/docs/languages/go/instrumentation/#record-errors
- Add attributes at span creation so samplers see them. https://opentelemetry.io/docs/concepts/signals/traces/#attributes
- Use `BatchSpanProcessor`; `SimpleSpanProcessor` is for debugging. https://opentelemetry.io/docs/languages/js/instrumentation/#picking-the-right-span-processor
- Apps keep their logging library plus a bridge or appender; the Logs Bridge API is for appender authors. https://opentelemetry.io/docs/concepts/signals/logs/#log-appender--bridge
- Metric cardinality limit is 2000 combinations per stream. Beyond it, extra combinations fold into one overflow data point marked `otel.metric.overflow=true`, so queries that filter or group by an attribute undercount. https://opentelemetry.io/docs/concepts/signals/metrics/#cardinality-limits

### Export and configuration

- Port 4317 is gRPC, 4318 is HTTP. The default protocol differs by SDK (Java agent `http/protobuf`, Java SDK `grpc`): set `OTEL_EXPORTER_OTLP_PROTOCOL`. https://opentelemetry.io/docs/languages/sdk-configuration/otlp-exporter/#protocol-configuration , https://opentelemetry.io/docs/zero-code/java/agent/configuration/#sdk-configuration
- A per-signal HTTP endpoint must include `/v1/traces`; the base endpoint gets it appended; gRPC endpoints take no path. https://opentelemetry.io/docs/languages/sdk-configuration/otlp-exporter/#otel_exporter_otlp_traces_endpoint
- `OTEL_PROPAGATORS` is the full list: `b3` alone drops `tracecontext,baggage`. https://opentelemetry.io/docs/languages/sdk-configuration/general/#otel_propagators
- With a declarative config file, the SDK ignores all `OTEL_*` variables except those referenced as `${VAR}`. https://opentelemetry.io/docs/languages/sdk-configuration/declarative-configuration/#environment-variables
- Never put credentials or PII in baggage: auto-instrumentation sends it on outgoing requests. https://opentelemetry.io/docs/concepts/signals/baggage/#baggage-security-considerations

### Collector

- A configured component is enabled only when listed in `service.pipelines` (extensions in `service.extensions`). https://opentelemetry.io/docs/collector/configuration/#service
- Put `memory_limiter` first in every pipeline. https://opentelemetry.io/docs/collector/deploy/other/agent-to-gateway/#recommended-processing
- The `filter` processor DROPS what matches. https://opentelemetry.io/docs/collector/transforming-telemetry/#basic-filtering
- Bind receivers to `localhost` or a pod IP, not `0.0.0.0`. https://opentelemetry.io/docs/security/config-best-practices/#protect-against-denial-of-service-attacks
- Tail sampling and span-to-metrics need all spans of a trace on one Collector: route with the `load_balancing` exporter. https://opentelemetry.io/docs/collector/scaling/#scaling-stateful-collectors
- OTLP/gRPC behind an L4 load balancer can pin clients to one Collector instance: use an L7 balancer that understands gRPC. https://opentelemetry.io/docs/collector/scaling/#scaling-stateless-collectors-and-using-load-balancers

### Kubernetes

- Create the `Instrumentation` CR before the workload, annotate `spec.template.metadata.annotations`, and set `spec.exporter.endpoint` (default `localhost:4317` drops data). https://opentelemetry.io/docs/platforms/kubernetes/operator/troubleshooting/automatic/
- The Collector Helm chart ships only the `debug` exporter, and `mode` is required. https://opentelemetry.io/docs/platforms/kubernetes/helm/collector/

## Old patterns

| Old | Current |
| :-- | :-- |
| Collector `logging` exporter | `debug` exporter |
| `service::telemetry::metrics::address` | `service::telemetry::metrics::readers` |
| `spanmetrics`, `servicegraph`, routing processors | the matching connectors |
| Jaeger exporter and client libraries | OTLP exporter and OpenTelemetry SDK |
| Collector servers on `0.0.0.0` by default | `localhost` by default since v0.110.0 |

Details and sources: `references/compatibility-and-migration.md`.

## Reference files

| File | Read it for |
| :-- | :-- |
| [references/concepts-and-signals.md](references/concepts-and-signals.md) | Signal stability, API vs SDK, spans, metrics, cardinality, logs, baggage |
| [references/library-instrumentation.md](references/library-instrumentation.md) | Instrumenting a library: API-only dependency, what to trace, tests |
| [references/context-propagation-and-sampling.md](references/context-propagation-and-sampling.md) | `traceparent`, propagators, head and tail sampling |
| [references/semantic-conventions-and-resources.md](references/semantic-conventions-and-resources.md) | Attribute names, stability opt-in, `service.*`, resource detectors |
| [references/sdk-configuration.md](references/sdk-configuration.md) | `OTEL_*` variables, OTLP endpoints, declarative YAML |
| [references/collector-configuration.md](references/collector-configuration.md) | Pipelines, queues, connectors, filter, transform, auth, TLS |
| [references/collector-deployment-and-scaling.md](references/collector-deployment-and-scaling.md) | Agent vs gateway, load balancing, persistence, internal telemetry |
| [references/collector-extending.md](references/collector-extending.md) | Distributions, ocb, custom components, OpAMP |
| [references/security.md](references/security.md) | Binding, secrets, redaction, privacy defaults |
| [references/javascript.md](references/javascript.md) | Node.js SDK, ESM, zero-code, views, browser, Lambda |
| [references/python.md](references/python.md) | SDK setup, `opentelemetry-instrument`, env vars, pre-fork servers |
| [references/go.md](references/go.md) | SDK bootstrap, errors, views, logs bridges, otelc |
| [references/java-sdk.md](references/java-sdk.md) | BOMs, `GlobalOpenTelemetry`, autoconfigure, views, testing |
| [references/java-agent-and-spring.md](references/java-agent-and-spring.md) | Java agent, `@WithSpan`, extensions, Spring Boot, Quarkus |
| [references/kubernetes.md](references/kubernetes.md) | DaemonSet plus gateway, Helm, Operator CRs, Target Allocator |
| [references/platforms.md](references/platforms.md) | AWS Lambda layers, Linux packages, Android, client apps |
| [references/compatibility-and-migration.md](references/compatibility-and-migration.md) | OpenTracing and Prometheus migration, deprecated components |
