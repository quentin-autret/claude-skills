# OTLP ingestion

Sending OpenTelemetry signals to Grafana Cloud: endpoint, auth, conversion, limits.

## Contents

- Endpoint and architecture
- Region hostnames
- Auth and verification
- Metrics conversion
- Promoting resource attributes
- Metrics limits and ingestion errors
- Logs over native OTLP
- Traces limits
- Old patterns

## Endpoint and architecture

- Default: the Grafana Cloud OTLP gateway for all 3 signals, behind a Collector (Grafana Alloy or the upstream Collector) in production. SDK-direct export suits dev/test, or when no Collector can be deployed: limited transport reliability, no enrichment, sampling or redaction.
  https://grafana.com/docs/grafana-cloud/send-data/otlp/send-data-otlp/
- With the base `OTEL_EXPORTER_OTLP_ENDPOINT` the SDK appends `/v1/metrics`, `/v1/logs`, `/v1/traces`. Signal-specific variables are used as-is, so include the full path:
  ```sh
  export OTEL_EXPORTER_OTLP_TRACES_ENDPOINT=https://otlp-gateway-<REGION>.grafana.net/otlp/v1/traces
  ```
  Python: replace `Basic ` with `Basic%20` in the generated connection variables. `OTEL_SERVICE_NAME` is required and unique.
  https://grafana.com/docs/grafana-cloud/send-data/otlp/send-data-otlp/
- Wire format: OTLP/HTTP binary protobuf, plain or gzip. JSON protobuf is only for low-traffic tests.
  https://grafana.com/docs/grafana-cloud/send-data/otlp/otlp-format-considerations/
- Traces go to the OTLP gateway, never to the Tempo query URL `https://<STACK>.grafana.net/tempo`, which rejects them.
  https://grafana.com/docs/grafana-cloud/send-data/traces/set-up/verify-ingestion/

## Region hostnames

- The hostname format is fixed per stack at creation. Regions created before 2026-01-15 use the flat `<service-host>.grafana.net` (`otlp-gateway-prod-us-east-0.grafana.net`). Regions created on or after 2026-01-15 use the nested `<service-host>.<csp>-<csp-region>-<counter>.grafana.net`. Copy the exact URL from the stack's OpenTelemetry tile or the portal, never build it from a template.  https://grafana.com/docs/grafana-cloud/platform/security-and-account-management/account-management/region-url-formats/

## Auth and verification

- Basic auth: user = numeric instance ID, password = Cloud Access Policy token with `traces:write` scope (for traces). Verify with curl on `/otlp/v1/traces`: success is HTTP 200 with body `{}`. 401 = wrong ID, token or scope. 404 = wrong path or Tempo URL. Connection error = wrong region. Put recent values in `startTimeUnixNano` / `endTimeUnixNano`, wait 15-30 s, then query `{ trace:id = "<lowercase hex>" }`.
  https://grafana.com/docs/grafana-cloud/send-data/traces/set-up/verify-ingestion/

## Metrics conversion

- `.` and `-` become `_` (`http.route` -> `http_route`). Query the converted names.
  https://grafana.com/docs/grafana-cloud/send-data/otlp/otlp-format-considerations/
- Suffixes are added: unit (`_bytes`), `_per_<unit>`, `_total` for monotonic sums, `ratio` for a gauge with unit `1`. Monotonic sum `system.io` in `By` becomes `system_io_bytes_total`. Keep suffix generation on.
  https://grafana.com/docs/grafana-cloud/send-data/otlp/otlp-format-considerations/
- Resource attributes land in `target_info`. `service.namespace/service.name` become `job`, `service.instance.id` becomes `instance`. Join:
  ```promql
  requests_total * on(job, instance) group_left(service_version) target_info
  ```
  https://grafana.com/docs/grafana-cloud/send-data/otlp/otlp-format-considerations/
- Promoted by default as labels on every series: `service.*` (name, namespace, version, instance.id), `cloud.availability_zone`, `cloud.region`, `container.name`, `deployment.environment[.name]`, `k8s.cluster.name` and the `k8s.*.name` workload attributes.
  https://grafana.com/docs/grafana-cloud/send-data/otlp/otlp-format-considerations/
- Exponential histograms become native histograms, priced differently from float samples. Native histograms are on by default (`max_native_histogram_buckets: 160`). Migrating: classic and native in parallel for business-critical metrics, native only otherwise.
  https://grafana.com/docs/grafana-cloud/send-data/metrics/metrics-prometheus/native-histograms/

## Promoting resource attributes

- Result = `(defaults + promote) - neverPromote`. Lists must be disjoint and duplicate-free. Use dotted OTel names.
  ```yaml
  apiVersion: mimir.ext.grafana.com/v1alpha1
  kind: MimirOTLPConfiguration
  metadata: { name: config }
  spec:
    resourceAttributes:
      promote: [test.special]
      neverPromote: [never.promoted]
  ```
  https://grafana.com/docs/grafana-cloud/send-data/otlp/promote-resource-attributes/
- One `MimirOTLPConfiguration` per stack. A push replaces the whole `spec`. Only new data changes, so queries grouping on those labels see a discontinuity. Self-service configuration is experimental, so contact Support to configure promotion.
  https://grafana.com/docs/grafana-cloud/send-data/otlp/promote-resource-attributes/
- Never promote per-process or per-request attributes (`container.id`, `host.id`, `process.pid`). Promote small stable sets (team, tier, region). Check dashboards and rules before using `neverPromote`.
  https://grafana.com/docs/grafana-cloud/send-data/otlp/promote-resource-attributes/
- Edit with `gcx resources pull mimirotlpconfiguration/config --path <DIR> -o yaml` then `gcx resources push --path <file>`. Check `status.operatorStates.mimir-config-controller.state`. Metrics only.
  https://grafana.com/docs/grafana-cloud/send-data/otlp/promote-resource-attributes/

## Metrics limits and ingestion errors

- OTLP: metric name 2048 bytes, 40 resource and 40 metric attributes (extras dropped), attribute value 2048 bytes (truncated). Only offending metrics in a batch are rejected.
  https://grafana.com/docs/grafana-cloud/send-data/otlp/otlp-format-considerations/
- Labels: name 1024 chars, value 2048 (truncated with a hash suffix by default).
  https://grafana.com/docs/grafana-cloud/send-data/metrics/label-handling-and-limits/
- Up to 40 label names per series, keep below 30.
  https://grafana.com/docs/grafana-cloud/send-data/metrics/metrics-prometheus/ingestion-errors/
- Default 150k active series per user, soft cap 25,000 data points/s, both raisable by Support. Default out-of-order window: 2 h behind the newest sample of the series. Duplicate timestamp with another value is rejected (`err-mimir-sample-duplicate-timestamp`). Slow body upload returns HTTP 598.
  https://grafana.com/docs/grafana-cloud/send-data/metrics/metrics-prometheus/ingestion-errors/
- Debug in the "Metrics Ingestion" dashboard first: some errors never reach client logs. Alert:
  ```promql
  grafanacloud_instance_samples_discarded_per_second{reason="per_user_series_limit"} > 0
  ```
  Failed auth is not a discard: look for 401 "Invalid token" and `prometheus_remote_storage_failed_samples_total`.
  https://grafana.com/docs/grafana-cloud/send-data/metrics/metrics-prometheus/ingestion-errors/

## Logs over native OTLP

- Default: collector `otlphttp` exporter with the `basicauth` extension.
  ```yaml
  exporters:
    otlphttp:
      auth: { authenticator: basicauth/otlp }
      endpoint: http://<loki-addr>/otlp
  ```
  https://grafana.com/docs/grafana-cloud/send-data/logs/collect-logs-with-otel/
- Body becomes the line. Default resource attributes become index labels, all other attributes structured metadata. Dots become `_`, so query `service_name`.
  https://grafana.com/docs/loki/latest/send-data/otel/
- Avoid `k8s.pod.name` and `service.instance.id` as index labels (still defaults). Move them to structured metadata with `limits_config.otlp_config` (actions `index_label`, `structured_metadata`, `drop`; `log_attributes` rules are first match wins). On Grafana Cloud, contact Support or use the config self-serve API.
  https://grafana.com/docs/loki/latest/send-data/otel/
- Cloud mapping to structured metadata: SeverityText -> `severity_text`, TraceId -> `trace_id`, SpanId -> `span_id`.
  https://grafana.com/docs/grafana-cloud/send-data/otlp/otlp-format-considerations/
- Limits: line 256 KB, 15 promoted labels, structured metadata 128 entries and 64 KB, request 64 MB (413).
  https://grafana.com/docs/grafana-cloud/send-data/otlp/otlp-format-considerations/
- Oversized `exception.stacktrace` gives a non-retryable 400 (`structured_metadata_too_large`). Move it to the body:
  ```yaml
  processors:
    transform/stacktrace:
      log_statements:
        - context: log
          statements:
            - set(body, Concat([body, attributes["exception.stacktrace"]], "\n")) where attributes["exception.stacktrace"] != nil
            - delete_key(attributes, "exception.stacktrace") where attributes["exception.stacktrace"] != nil
  ```
  https://grafana.com/docs/loki/latest/send-data/otel/

## Traces limits

- 5 MB per trace, 15 MB/s ingest, 20 MB/s burst. Attribute name 1024 bytes, value 2048 (truncated).
  https://grafana.com/docs/grafana-cloud/send-data/otlp/otlp-format-considerations/
- `RESOURCE_EXHAUSTED` is retryable. Bound collector memory with `sending_queue` and `retry_on_failure` or it can OOM.
  https://grafana.com/docs/grafana-cloud/send-data/traces/set-up/troubleshoot/

## Old patterns

| Old | Current | Source |
| --- | --- | --- |
| Loki Exporter (deprecated), `otelcol.exporter.loki` (old JSON format) | `otlphttp` to the OTLP gateway | https://grafana.com/docs/grafana-cloud/send-data/otlp/adopt-new-logs-format/ |
| Hints `loki.resource.labels`, `loki.attribute.labels` | `otlp_config` and structured metadata | https://grafana.com/docs/grafana-cloud/send-data/otlp/otlp-format-considerations/ |
| `{job="shop/shoppingcart"} \| json \| severity_text="INFO"` | `{service_name="shoppingcart", service_namespace="shop"} \| severity_text="INFO"` | https://grafana.com/docs/grafana-cloud/send-data/otlp/adopt-new-logs-format/ |

Switching formats means rewriting LogQL queries and asking Support to unpin the tenant from the exporter format.
https://grafana.com/docs/grafana-cloud/send-data/otlp/adopt-new-logs-format/
