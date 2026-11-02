# Traces and TraceQL

Span design, sampling, metrics-generator, exemplars and TraceQL rules for Grafana Cloud Traces (Tempo).

## Contents

- Span size and attributes
- Sampling
- Metrics-generator
- Exemplars
- TraceQL semantics
- TraceQL performance
- Query patterns
- Deleting leaked data
- Old patterns

## Span size and attributes

- Cloud truncates attributes above 2048 bytes and partially discards traces over the per-trace limit (`trace_too_large`). Only Support can raise it.
  https://grafana.com/docs/grafana-cloud/send-data/traces/configure/reduce-trace-size/
- SDK limits default to 128 attributes, events and links per span, with no value length limit. Set `OTEL_ATTRIBUTE_VALUE_LENGTH_LIMIT=2048`, one of the most effective size controls.
  https://grafana.com/docs/grafana-cloud/send-data/traces/configure/reduce-trace-size/
- Avoid: one span per loop iteration or retry (cap retries, record the count on the parent), unbounded fan-out, full bodies or SQL as attributes, one span event per log line.
  https://grafana.com/docs/grafana-cloud/send-data/traces/configure/reduce-trace-size/
- Each attribute costs. When unsure, leave it out. Never put metrics or logs in attributes.
  https://grafana.com/docs/grafana-cloud/send-data/traces/tracing-best-practices/
- Trim in the collector (filter, attributes, transform processors) to keep whole traces. Adaptive Traces drop policies only drop whole traces.
  https://grafana.com/docs/grafana-cloud/send-data/traces/configure/reduce-trace-size/
- Find oversized traces: `count() > 10000 | rate() by (resource.service.name)`, then open traces from the exemplars.
  https://grafana.com/docs/grafana-cloud/send-data/traces/configure/reduce-trace-size/
- Retention is 30 days by default. Cloud Traces users contact Support to modify overrides.
  https://grafana.com/docs/grafana-cloud/send-data/traces/remove-trace-info/ and https://grafana.com/docs/grafana-cloud/send-data/traces/tracing-best-practices/

## Sampling

- Tail sampling pitfalls: a long decision wait delays spans, composite AND samplers can drop most traces.
  https://grafana.com/docs/grafana-cloud/send-data/traces/configure/sampling/
- Adaptive Traces (managed tail sampling) applies policy updates in about 10 s. Its diversity policy keeps at least 1 trace per fingerprint (`service.name`, `http.route`, `http.response.status_code`) every 15 min, so the probabilistic rate can go lower.
  https://grafana.com/docs/grafana-cloud/send-data/traces/configure/sampling/

## Metrics-generator

- Off by default. Enable with the Application Observability defaults, or ask Support for custom settings. Defaults cover `SERVER` and `CONSUMER` spans only. `CLIENT` and `PRODUCER` are opt-in, `INTERNAL` needs Support.
  https://grafana.com/docs/grafana-cloud/send-data/traces/configure/metrics-generator/
- Generated series are billed as metrics, aggregated by Adaptive Metrics, default collection interval 60 s. Down-sampling before Tempo lowers generated metrics.
  https://grafana.com/docs/grafana-cloud/send-data/traces/configure/metrics-generator/
- Spans ending more than the slack (30 s) ago are discarded by the generator: they stay searchable but RED metrics skew. TraceQL metrics are unaffected. Fix: shorter tail decision wait plus batch timeout (25 s + 10 s overshoots), pipeline near the edge, a larger slack via Support.
  https://grafana.com/docs/grafana-cloud/send-data/traces/configure/metrics-generator/
- Watch `grafanacloud_traces_instance_metrics_generator_discarded_spans_per_second` (reason `outside_metrics_ingestion_slack`) and `..._label_values_limited_per_second` (values become `__cardinality_overflow__`) in `grafanacloud-usage`.
  https://grafana.com/docs/grafana-cloud/send-data/traces/configure/metrics-generator/
- TraceQL metrics run on `local_blocks`, limited to a 24 h time range. Support can increase it.
  https://grafana.com/docs/grafana-cloud/send-data/traces/configure/metrics-generator/

## Exemplars

- Emit via an official Prometheus client in OpenMetrics format (Go: `EnableOpenMetrics`, `ObserveWithExemplar`, `AddWithExemplar`). Check: `curl -H "Accept: application/openmetrics-text" http://<app>/metrics | grep -i traceid`.
  https://grafana.com/docs/grafana-cloud/send-data/traces/configure/exemplars/
- Alloy sets `send_exemplars = true` in the `prometheus.remote_write` `endpoint` block. Prometheus needs `--enable-feature=exemplar-storage`.
  https://grafana.com/docs/grafana-cloud/send-data/traces/configure/exemplars/
- Exemplar links giving 404 are expected when tail sampling later dropped the trace.
  https://grafana.com/docs/grafana-cloud/send-data/traces/configure/exemplars/

## TraceQL semantics

- Conditions inside one `{}` must hold on the same span. Separate `{}` joined by `&&` may match different spans: `{ resource.cloud.region = "us-east-1" } && { resource.cloud.region = "us-west-1" }` works, one `{}` never matches.
  https://grafana.com/docs/tempo/latest/traceql/construct-traceql-queries/
- Results are first-match, not newest. Experimental `with (most_recent=true)` returns newest first.
  https://grafana.com/docs/tempo/latest/traceql/construct-traceql-queries/
- Regex is Go syntax and fully anchored: `=~ "bar"` means `^bar$`, use `.*bar.*`. `= nil` means missing.
  https://grafana.com/docs/tempo/latest/traceql/construct-traceql-queries/
- Unscoped `duration` is `span:duration`, not `trace:duration`. Wrong-looking percentiles often come from this.
  https://grafana.com/docs/tempo/latest/traceql/construct-traceql-queries/
- Intrinsics use a colon (`span:status`, `trace:rootService`), attributes a dot (`span.http.method`). Quote odd names: `span."attribute name with space"`.
  https://grafana.com/docs/tempo/latest/traceql/construct-traceql-queries/
- Array attributes: `=` matches if any element matches, `!=` only if none does.
  https://grafana.com/docs/tempo/latest/traceql/construct-traceql-queries/
- Structural `>>`, `<<`, `>`, `<`, `~` return the right side. `&>>` returns both. Negated forms (`!>>`) are experimental with false positives.
  https://grafana.com/docs/tempo/latest/traceql/construct-traceql-queries/

## TraceQL performance

- Queries made only of `&&` conditions are much faster: the engine pushes filtering down into the Parquet layer (predicate pushdown). Queries with `||` or structural operators are slower. Use `=~ "us-east-1|us-west-1"` instead of `||` on one attribute.
  https://grafana.com/docs/tempo/latest/traceql/tune-traceql-queries/
- Always scope attributes (`span.`, `resource.`). Unscoped `.http.status_code` searches several places. Reference few attributes.
  https://grafana.com/docs/tempo/latest/traceql/tune-traceql-queries/
- Prefer trace intrinsics and resource attributes: much smaller columns.
  https://grafana.com/docs/tempo/latest/traceql/tune-traceql-queries/
- Approximate large metrics queries: `{ resource.service.name = "api" } | quantile_over_time(duration, 0.9) with(sample=true)`.
  https://grafana.com/docs/tempo/latest/traceql/tune-traceql-queries/

## Query patterns

```traceql
{ trace:rootService = "gateway" && trace:rootName = "HTTP GET /health" }
{ resource.service.name = "frontend" } >> { span:status = error }
{ span:status = error } | select(span.http.response.status_code, span.url.full)
{ span:status = error } | count() by (resource.service.name)
{ span:duration < 5ms } | count_over_time() by(resource.service.name) | topk(10)
```
https://grafana.com/docs/grafana-cloud/send-data/traces/traces-query-editor/traceql-cookbook/ and https://grafana.com/docs/tempo/latest/traceql/construct-traceql-queries/

`select()` is cheap: fields are fetched after filtering.
https://grafana.com/docs/tempo/latest/traceql/construct-traceql-queries/

## Deleting leaked data

An org admin opens a Support issue with stack, UTC range and trace IDs or a TraceQL query (1,000 traces max). Fix the source first.
https://grafana.com/docs/grafana-cloud/send-data/traces/remove-trace-info/

## Old patterns

| Old | Current | Source |
| --- | --- | --- |
| Unscoped `duration` read as the trace duration | `trace:duration` for the whole trace, `span:duration` for one span (unscoped `duration` is an alias for `span:duration`) | https://grafana.com/docs/tempo/latest/traceql/construct-traceql-queries/ |
| Graph panel for exemplars | Time series panel, Exemplars on | https://grafana.com/docs/grafana-cloud/send-data/traces/configure/exemplars/ |
| Prometheus text format exemplars | OpenMetrics | https://grafana.com/docs/grafana-cloud/send-data/traces/configure/exemplars/ |
