# Compatibility and migration

## Contents

- From OpenTracing, OpenCensus and Jaeger clients
- From Prometheus client libraries
- Exporting OTLP to Prometheus
- Deprecated components and their replacements
- Upgrade habits

## From OpenTracing, OpenCensus and Jaeger clients

- OpenTelemetry merges OpenTracing and OpenCensus and aims for backward compatibility. Jaeger clients are deprecated; Jaeger accepts OTLP since v1.35, so switch from the Jaeger exporter to OTLP. https://opentelemetry.io/docs/compatibility/migration/
- Every SDK ships an OpenTracing shim, disabled by default: Go `go.opentelemetry.io/otel/bridge/opentracing`, JavaScript `@opentelemetry/shim-opentracing`, Java `opentracing-shim`. OpenTelemetry needs newer runtimes (Java 8, Python 3.6, Go 1.16 minimum). https://opentelemetry.io/docs/compatibility/migration/opentracing/#language-version-support
- Step 1: swap the SDK alone (OpenTelemetry SDK plus shim), export in the old format or OTLP to a Collector, and confirm dashboards and alerts. https://opentelemetry.io/docs/compatibility/migration/opentracing/#step-1-install-the-opentelemetry-sdk
- Step 2: write new code with the OpenTelemetry API, replace OpenTracing instrumentation libraries one at a time, and build new dashboards BEFORE production. Optionally emit both shapes for a while via Collector rules. https://opentelemetry.io/docs/compatibility/migration/opentracing/#step-2-progressively-replace-instrumentation
- Baggage set through the OpenTracing API is invisible to the OpenTelemetry API and not injected by its propagators: switch all baggage calls at once. https://opentelemetry.io/docs/compatibility/migration/opentracing/#baggage
- JavaScript: OpenTelemetry uses context managers while OpenTracing passes spans explicitly. Migrate whole code paths so one trace uses one API. https://opentelemetry.io/docs/compatibility/migration/opentracing/#context-management-in-javascript

## From Prometheus client libraries

- No upfront label declaration: attribute keys and values come with each measurement. https://opentelemetry.io/docs/compatibility/prometheus/client-libraries/#label-names-attributes
- Use dotted names without an app prefix; the Meter name gives the namespace. The exporter translates: counter `hvac.on` with unit `s` becomes `hvac_on_seconds_total`. https://opentelemetry.io/docs/compatibility/prometheus/client-libraries/#naming-conventions
- Prometheus `Gauge` maps to `Gauge` (non-additive, temperature) or `UpDownCounter` (additive, active connections). `GaugeFunc`/`CounterFunc` map to asynchronous instruments. https://opentelemetry.io/docs/compatibility/prometheus/client-libraries/#stateful-and-callback-instruments
- `inc()` becomes `add(value)` with an explicit value. https://opentelemetry.io/docs/compatibility/prometheus/client-libraries/#counter
- Default histogram buckets are millisecond-scale `[0, 5, 10, 25, 50, 75, 100, 250, 500, 750, 1000, 2500, 5000, 7500, 10000]`: wrong for seconds. Set boundaries (Java `setExplicitBucketBoundariesAdvice()`, Go `WithExplicitBucketBoundaries`) or a view. https://opentelemetry.io/docs/compatibility/prometheus/client-libraries/#classic-explicit-histogram
- Native histograms: `MaxScale: 3` on `AggregationBase2ExponentialHistogram` approximates bucket factor 1.1. https://opentelemetry.io/docs/compatibility/prometheus/client-libraries/#native-base2-exponential-histogram
- Summary has no equivalent: use a base2 exponential histogram and `histogram_quantile()` at query time. https://opentelemetry.io/docs/compatibility/prometheus/client-libraries/#summary
- The Prometheus exporter forces cumulative temporality. https://opentelemetry.io/docs/compatibility/prometheus/client-libraries/#otel-aggregation-temporality
- Every point gets `otel_scope_name` and `otel_scope_version`. `without_scope_info` removes them only safely when each metric name comes from one scope, otherwise series collide. https://opentelemetry.io/docs/compatibility/prometheus/client-libraries/#otel-instrumentation-scope
- Resource attributes become `job`, `instance` and a `target_info` metric; promote chosen ones with `with_resource_constant_labels`. Stop emitting `target_info` by hand. https://opentelemetry.io/docs/compatibility/prometheus/client-libraries/#otel-resource-attributes

## Exporting OTLP to Prometheus

- Prometheus can receive OTLP over HTTP (enable its OTLP receiver). Good for simple setups; in production put a Collector in between. https://opentelemetry.io/docs/compatibility/prometheus/otlp-metrics-export/

```bash
export OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf
export OTEL_EXPORTER_OTLP_METRICS_ENDPOINT=http://localhost:9090/api/v1/otlp
export OTEL_TRACES_EXPORTER=none
export OTEL_LOGS_EXPORTER=none
export OTEL_METRIC_EXPORT_INTERVAL=15000
export OTEL_SERVICE_NAME="my-example-service"
export OTEL_RESOURCE_ATTRIBUTES="service.instance.id=$(uuidgen)"
```

https://opentelemetry.io/docs/compatibility/prometheus/otlp-metrics-export/#use-environment-variables

- Set `service.name` and a per-instance `service.instance.id`, or instances are indistinguishable. https://opentelemetry.io/docs/compatibility/prometheus/otlp-metrics-export/#use-environment-variables
- Alerts break silently when scrapers use different label names (`pod_name` vs `pod`). https://opentelemetry.io/docs/guidance/blueprints/k8s-observability-blueprint/#common-challenges
- Drop SDK HTTP/RPC metrics but keep spans with views, not by disabling the instrumentation (Java `OTEL_EXPERIMENTAL_METRICS_VIEW_CONFIG` file): https://opentelemetry.io/docs/guidance/reference-implementations/skyscanner/#spans-yes-metrics-no-by-default

```yaml
- selector:
    instrument_name: http.*
  view:
    aggregation: drop
```

## Deprecated components and their replacements

| Old | Current |
| :-- | :-- |
| Collector `logging` exporter | `debug` exporter (since v0.86.0) https://opentelemetry.io/docs/collector/configuration/#exporters |
| `service::telemetry::metrics::address` | `readers` with a `pull` or `periodic` exporter (ignored since v0.123.0) https://opentelemetry.io/docs/collector/internal-telemetry/#service-address |
| `spanmetrics` / `servicegraph` processors | spanmetrics / servicegraph connectors https://opentelemetry.io/docs/collector/extend/custom-component/connector/#why-use-a-connector |
| routing processor | routing connector https://opentelemetry.io/docs/guidance/reference-implementations/adobe/#navigating-component-deprecations |
| Jaeger client libraries and exporter | OpenTelemetry SDK with OTLP https://opentelemetry.io/docs/compatibility/migration/ |
| JMX Metric Gatherer | JMX scraper https://opentelemetry.io/docs/languages/java/jmx/ |
| Python `ConsoleLogExporter` | `ConsoleLogRecordExporter` (1.39.0+) https://opentelemetry.io/docs/languages/python/instrumentation/#logs |
| Python `OTEL_PYTHON_LOGGING_AUTO_INSTRUMENTATION_ENABLED` | `OTEL_PYTHON_LOG_AUTO_INSTRUMENTATION`, on by default (1.40.0+) https://opentelemetry.io/docs/zero-code/python/configuration/#logging |

## Upgrade habits

- Upgrade often so breaking changes arrive in small batches; six-month jumps bundle many. https://opentelemetry.io/docs/guidance/reference-implementations/mastodon/#advice-for-small-teams , https://opentelemetry.io/docs/guidance/reference-implementations/skyscanner/#lessons-and-pain-points
- Start a Collector simple: memory limiter from day one, filters early, no over-engineered resiliency. https://opentelemetry.io/docs/guidance/reference-implementations/skyscanner/#advice-for-others
