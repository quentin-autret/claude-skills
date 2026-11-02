# Collector configuration

## Contents

- Loading, merging and overriding
- Structure: a component is enabled only in a pipeline
- Exporters, batching and component names
- Connectors
- Processors and OTTL
- Environment variables, auth, TLS
- Checking and debugging a config

## Loading, merging and overriding

- Default file `/etc/<distro>/config.yaml` (`otelcol`, `otelcol-contrib`). `--config` takes a path or a URI: `file:`, `env:`, `yaml:`, `http:`, `https:`. https://opentelemetry.io/docs/collector/configuration/#location
- Several `--config` flags merge; parts may be partial but the result must be complete. Nested keys use `::`, not dots: `--config="yaml:exporters::debug::verbosity: normal"`. https://opentelemetry.io/docs/collector/configuration/#location
- `--set outer::inner=value` overrides after merging; it cannot set keys containing `.` or `=`. Use files for anything complex. https://opentelemetry.io/docs/collector/configuration/#override-settings
- Pull a section from another file: `exporters: ${file:exporters.yaml}`. https://opentelemetry.io/docs/collector/configuration/#embedding-other-configuration-providers
- When merging, a later null replaces an earlier map: write `processors: {}` rather than a bare `processors:`. https://opentelemetry.io/docs/collector/troubleshooting/#null-maps

## Structure: a component is enabled only in a pipeline

- Configuring a receiver, processor, exporter or connector does NOT enable it; it must appear in `service.pipelines`. Extensions must appear in `service.extensions`. A component that is configured but not named in the `service` section is not enabled. https://opentelemetry.io/docs/collector/configuration/#service
- IDs are `type[/name]` (`otlp`, `otlp/2`); pipelines too (`traces`, `traces/2`). https://opentelemetry.io/docs/collector/configuration/#basics
- The Collector requires one or more receivers and one or more exporters; processors are optional. Processors are optional and run in list order. https://opentelemetry.io/docs/collector/configuration/#receivers , https://opentelemetry.io/docs/collector/configuration/#exporters , https://opentelemetry.io/docs/collector/configuration/#processors
- A processor used in several pipelines gets one instance per pipeline. The same processor cannot appear twice in one pipeline. https://opentelemetry.io/docs/collector/architecture/#processors
- A receiver feeding several pipelines calls them synchronously: one blocked processor blocks the other pipelines and the receiver. https://opentelemetry.io/docs/collector/architecture/#receivers
- A component that does not support the pipeline's signal fails at load with `pipeline.ErrSignalNotSupported`. https://opentelemetry.io/docs/collector/architecture/#pipelines
- Server endpoints default to `localhost`; doc examples use `0.0.0.0` only for convenience. https://opentelemetry.io/docs/collector/configuration/#basics

## Exporters, batching and component names

- Batching is configured in the exporter `sending_queue` in current examples; agent and gateway examples use no separate `batch` processor. The sample below combines the agent example (which names its exporter `otlp`) with the `otlp_grpc` type name of the configuration page. https://opentelemetry.io/docs/collector/configuration/#basics , https://opentelemetry.io/docs/collector/deploy/other/agent-to-gateway/#example-agent-configuration-without-load-balancing

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
        sizer: items
        flush_timeout: 1s
service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter]
      exporters: [otlp_grpc]
```

https://opentelemetry.io/docs/collector/configuration/#basics , https://opentelemetry.io/docs/collector/deploy/other/agent-to-gateway/#example-agent-configuration-without-load-balancing

- Exporter type names differ across pages: the configuration page uses `otlp_grpc` / `otlp_http` and `host_metrics`; resiliency and deployment pages still use `otlp`. Check the exact type with `otelcol components` for your version. https://opentelemetry.io/docs/collector/configuration/#exporters , https://opentelemetry.io/docs/collector/troubleshooting/#check-collector-components
- Put `memory_limiter` FIRST in every pipeline: small limits on agents, larger on gateways. https://opentelemetry.io/docs/collector/deploy/other/agent-to-gateway/#recommended-processing
- Set a batch `max_size` so the backend does not reject oversized payloads. https://opentelemetry.io/docs/collector/deploy/other/agent-to-gateway/#retries
- The `otlp` exporter compresses with `gzip` by default. https://opentelemetry.io/docs/security/config-best-practices/
- Exporters using Go `net/http` honor `HTTP_PROXY`, `HTTPS_PROXY`, `NO_PROXY` set at startup. https://opentelemetry.io/docs/collector/configuration/#proxy-support

## Connectors

- A connector is the exporter of one pipeline and the receiver of another, possibly of another signal (traces in, metrics out). List it in both pipelines. https://opentelemetry.io/docs/collector/configuration/#connectors
- Count connector conditions are ORed (an event needs to match only one). In the page's example the original traces are not exported anywhere; add a trace exporter to the traces pipeline to keep them. https://opentelemetry.io/docs/collector/configuration/#connectors

```yaml
connectors:
  count:
    spanevents:
      my.prod.event.count:
        conditions:
          - 'attributes["env"] == "prod"'
service:
  pipelines:
    traces:
      receivers: [otlp]
      exporters: [count]
    metrics:
      receivers: [count]
      exporters: [otlp_grpc]
```

https://opentelemetry.io/docs/collector/configuration/#connectors

## Processors and OTTL

- `filter`: telemetry matching ANY condition is DROPPED. To keep some services, write the negation. The page examples set `error_mode: ignore`. https://opentelemetry.io/docs/collector/transforming-telemetry/#basic-filtering

```yaml
processors:
  filter/ottl:
    error_mode: ignore
    traces:
      span:
        - resource.attributes["service.name"] == "service1"
```

https://opentelemetry.io/docs/collector/transforming-telemetry/#basic-filtering

- `attributes` updates, inserts, deletes or replaces attributes on metrics or traces (actions seen in the page examples: insert, upsert, delete, hash); `resource` has an identical configuration but applies only to resource attributes. It also supports `from_attribute`. Span names are handled by the `span` processor, not `attributes`. https://opentelemetry.io/docs/collector/transforming-telemetry/#adding-or-deleting-attributes , https://opentelemetry.io/docs/collector/troubleshooting/#collector-is-not-processing-data
- `transform` (OTTL) sets a span status, and can modify span names or extract span attributes from them: https://opentelemetry.io/docs/collector/transforming-telemetry/#setting-a-span-status

```yaml
transform:
  error_mode: ignore
  trace_statements:
    - set(span.status.code, STATUS_CODE_OK) where span.attributes["http.request.status_code"] == 400
```

- `metricstransform` renames metrics and labels. https://opentelemetry.io/docs/collector/transforming-telemetry/#renaming-metrics-or-metric-labels
- `k8sattributes` needs read access to certain Kubernetes RBAC APIs. https://opentelemetry.io/docs/collector/transforming-telemetry/#enriching-telemetry-with-resource-attributes
- `transform`, `tail_sampling` and `metricstransform` are not in the core distribution. https://opentelemetry.io/docs/collector/components/processor/

## Environment variables, auth, TLS

- `${env:DB_KEY}`, default `${env:DB_KEY:-mydefault}`, literal dollar `$$`. https://opentelemetry.io/docs/collector/configuration/#environment-variables
- Auth is an extension: define it, list it in `service.extensions`, reference it under the receiver protocol (`auth: authenticator: oidc`) or exporter. Missing either step means no auth. https://opentelemetry.io/docs/collector/configuration/#authentication
- TLS: receiver `tls.cert_file` + `key_file`, add `client_ca_file` for mTLS; exporter `tls.ca_file` plus client cert and key. `insecure` and `insecure_skip_verify` are not for production. https://opentelemetry.io/docs/collector/configuration/#common-tls-settings

## Checking and debugging a config

- `otelcol validate --config=customconfig.yaml` before deploying; `otelcol components` lists components and stability levels in the binary. https://opentelemetry.io/docs/collector/configuration/#location , https://opentelemetry.io/docs/collector/troubleshooting/#check-collector-components
- `otelcol print-config` needs `--feature-gates=otelcol.printInitialConfig` (experimental); output is redacted by default. https://opentelemetry.io/docs/collector/configuration/#how-to-examine-the-final-configuration
- `debug` exporter with `verbosity: detailed` prints full payloads. https://opentelemetry.io/docs/collector/troubleshooting/#local-exporters
- zPages extension on 55679 (`/debug/tracez`) is local-only by default. https://opentelemetry.io/docs/collector/troubleshooting/#extensions
- Not receiving data: network, receiver or client config, or a receiver not listed in a pipeline. https://opentelemetry.io/docs/collector/troubleshooting/#collector-is-not-receiving-data
- Catalog components marked with a warning sign are unmaintained. https://opentelemetry.io/docs/collector/components/exporter/
