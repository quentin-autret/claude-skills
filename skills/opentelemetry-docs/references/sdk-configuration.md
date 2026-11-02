# SDK configuration

## Contents

- Environment variables
- OTLP exporter endpoints and protocol
- Headers, timeout, metrics options
- Declarative configuration (YAML)

## Environment variables

- Support varies by language: check the environment variable compliance matrix before relying on one. https://opentelemetry.io/docs/languages/sdk-configuration/ (matrix: https://github.com/open-telemetry/opentelemetry-specification/blob/main/spec-compliance-matrix.md#environment-variables)
- Always set `OTEL_SERVICE_NAME`; without it `service.name` is `unknown_service`. https://opentelemetry.io/docs/languages/sdk-configuration/general/#otel_service_name
- `OTEL_TRACES_EXPORTER`, `OTEL_METRICS_EXPORTER`, `OTEL_LOGS_EXPORTER` default to `otlp`; `none` disables a signal's exporter. Traces also accept `zipkin`, `console`; metrics `prometheus`, `console`; logs `console`. https://opentelemetry.io/docs/languages/sdk-configuration/general/#otel_traces_exporter
- `OTEL_TRACES_SAMPLER` default `parentbased_always_on`; `OTEL_PROPAGATORS` default `tracecontext,baggage`. https://opentelemetry.io/docs/languages/sdk-configuration/general/
- Metrics push interval default is 60 s (`OTEL_METRIC_EXPORT_INTERVAL` in ms). A shorter interval (the page's example is 15000 ms) gives more responsive metrics and faster alerting at the cost of higher network and processing overhead. https://opentelemetry.io/docs/compatibility/prometheus/otlp-metrics-export/#use-environment-variables
- Go does NOT support `OTEL_SDK_DISABLED`. https://opentelemetry.io/docs/languages/go/exporters/#automatic-exporter-configuration-with-environment-variables

## OTLP exporter endpoints and protocol

- Defaults: gRPC `http://localhost:4317`, HTTP `http://localhost:4318`. Port 4317 is the gRPC default and 4318 the HTTP default, so keep protocol and port consistent. https://opentelemetry.io/docs/languages/sdk-configuration/otlp-exporter/#otel_exporter_otlp_endpoint
- `OTEL_EXPORTER_OTLP_PROTOCOL` (plus `_TRACES_`, `_METRICS_`, `_LOGS_`, `_PROFILES_` variants): `grpc`, `http/protobuf`, `http/json`. The default depends on the SDK: Java SDK autoconfigure `grpc`, Java agent 2.x and Spring starter `http/protobuf`, Node.js operator injection `grpc`. Set it explicitly. https://opentelemetry.io/docs/languages/sdk-configuration/otlp-exporter/#protocol-configuration , https://opentelemetry.io/docs/languages/java/configuration/#properties-exporters , https://opentelemetry.io/docs/platforms/kubernetes/operator/automatic/
- Base endpoint over HTTP: the SDK constructs `/v1/traces`, `/v1/metrics`, `/v1/logs` URLs from it. https://opentelemetry.io/docs/languages/sdk-configuration/otlp-exporter/#otel_exporter_otlp_endpoint
- Signal-specific endpoint over HTTP typically ends with the signal path (default `http://localhost:4318/v1/traces`; likewise `v1/metrics`, `v1/logs`, `v1/profiles`), so include it. https://opentelemetry.io/docs/languages/sdk-configuration/otlp-exporter/#otel_exporter_otlp_traces_endpoint
- gRPC endpoints never take a `/v1/...` path. https://opentelemetry.io/docs/languages/sdk-configuration/otlp-exporter/#otel_exporter_otlp_endpoint

```sh
export OTEL_EXPORTER_OTLP_ENDPOINT="http://my-api-endpoint/"                    # HTTP base
export OTEL_EXPORTER_OTLP_TRACES_ENDPOINT="http://my-api-endpoint/v1/traces"    # HTTP per signal
export OTEL_EXPORTER_OTLP_ENDPOINT="https://my-api-endpoint:443"                # gRPC
```

https://opentelemetry.io/docs/languages/sdk-configuration/otlp-exporter/#endpoint-configuration

- Exporters configured in code (JavaScript `url`, Python HTTP `endpoint`) take the full signal path too. https://opentelemetry.io/docs/languages/js/exporters/#usage-with-nodejs , https://opentelemetry.io/docs/languages/python/exporters/#usage
- SDK OTLP exporters retry on HTTP 429/503 or gRPC UNAVAILABLE but have no Collector-style queue and drop batches that keep failing: send to a local Collector fast. https://opentelemetry.io/docs/guidance/blueprints/managed-telemetry-platforms-for-k8s-workloads/#2-include-organization-standards-into-default-extensible-application-level-configuration-action-2

## Headers, timeout, metrics options

- `OTEL_EXPORTER_OTLP_HEADERS` uses W3C Baggage format `key1=value1,key2=value2`; semicolon metadata is not supported. Header values are credentials: inject them from a secret, never commit them. https://opentelemetry.io/docs/languages/sdk-configuration/otlp-exporter/#header-configuration

```sh
export OTEL_EXPORTER_OTLP_HEADERS="api-key=key,other-config-value=value"
```

https://opentelemetry.io/docs/languages/sdk-configuration/otlp-exporter/#header-configuration

- `OTEL_EXPORTER_OTLP_TIMEOUT` is in milliseconds, default 10000. https://opentelemetry.io/docs/languages/sdk-configuration/otlp-exporter/#timeout-configuration
- `OTEL_EXPORTER_OTLP_METRICS_TEMPORALITY_PREFERENCE` default `cumulative`. `delta`: delta for Counter, async Counter and Histogram, cumulative for UpDownCounters. `lowmemory`: delta only for synchronous Counter and Histogram. https://opentelemetry.io/docs/languages/sdk-configuration/otlp-exporter/#otel_exporter_otlp_metrics_temporality_preference
- `OTEL_EXPORTER_OTLP_METRICS_DEFAULT_HISTOGRAM_AGGREGATION` default `explicit_bucket_histogram`; alternative `base2_exponential_bucket_histogram`. https://opentelemetry.io/docs/languages/sdk-configuration/otlp-exporter/#otel_exporter_otlp_metrics_default_histogram_aggregation

## Declarative configuration (YAML)

- Enable with `OTEL_CONFIG_FILE=/path/to/otel-config.yaml`. The schema is stable; keys suffixed `/development` are experimental and may not be supported by every SDK. SDK support is still experimental, Java being the language listed. https://opentelemetry.io/docs/languages/sdk-configuration/declarative-configuration/
- Once a config file is used, the SDK ignores ALL environment variables, standard `OTEL_*` ones included, except those referenced with `${VAR}` or `${VAR:-default}` in the file. https://opentelemetry.io/docs/languages/sdk-configuration/declarative-configuration/#environment-variables
- With `otlp_http` and the base variable you append `/v1/<signal>` yourself. `otlp_grpc` takes no path. https://opentelemetry.io/docs/languages/sdk-configuration/declarative-configuration/#endpoint-per-signal , https://opentelemetry.io/docs/languages/sdk-configuration/declarative-configuration/#grpc-exporter
- Keep resource attributes in the env var with `attributes_list: ${OTEL_RESOURCE_ATTRIBUTES}` (works with the Kubernetes Operator). https://opentelemetry.io/docs/languages/sdk-configuration/declarative-configuration/#resource-attributes
- Starting file recommended by the docs (metrics and logs follow the same shape with `meter_provider.readers[].periodic` and `logger_provider.processors[].batch`):

```yaml
file_format: "1.2"
resource:
  attributes_list: ${OTEL_RESOURCE_ATTRIBUTES}
  detection/development:
    detectors:
      - service:
      - host:
      - process:
      - container:
propagator:
  composite:
    - tracecontext:
    - baggage:
tracer_provider:
  sampler:
    parent_based:
      root:
        always_on:
  processors:
    - batch:
        exporter:
          otlp_http:
            endpoint: ${OTEL_EXPORTER_OTLP_ENDPOINT:-http://localhost:4318}/v1/traces
```

https://opentelemetry.io/docs/languages/sdk-configuration/declarative-configuration/#getting-started

- The `service` detector sets `service.instance.id` and `service.name` from `OTEL_SERVICE_NAME`. https://opentelemetry.io/docs/languages/sdk-configuration/declarative-configuration/#getting-started
- Migrating from env vars: start from `otel-sdk-migration-config.yaml` in the opentelemetry-configuration repo; the kitchen-sink example lists every option. https://opentelemetry.io/docs/languages/sdk-configuration/declarative-configuration/#migration-configuration
- Types prefixed `Experimental` in the schema can break without notice. https://opentelemetry.io/docs/specs/otel-config/types/
- Java agent (2.26.0 and later) enables it with `-Dotel.config.file=/path/to/otel-config.yaml` and accepts durations only in milliseconds (`5000`, not `5s`). https://opentelemetry.io/docs/zero-code/java/agent/declarative-configuration/
