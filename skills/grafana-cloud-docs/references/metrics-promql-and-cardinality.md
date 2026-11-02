# Metrics, PromQL and cardinality

Querying Grafana Cloud Metrics (Mimir), the Prometheus data source editor, billing, limits and Adaptive Metrics.

## Contents

- Query cost rules
- Label APIs
- HTTP API access
- Query editor and `$__rate_interval`
- Billing
- Default limits
- Adaptive Metrics
- Old patterns

## Query cost rules

- Filter and aggregate in PromQL (`sum by (job) (metric)`), not in Grafana. Prefer `label="value"` over regex. Avoid long look-backs on frequent queries (`rate(m[30d])` every minute).
  https://grafana.com/docs/mimir/latest/query/query-best-practices/
- Recording rules for expensive frequent dashboard queries and long ranges, not for rarely evaluated ones.
  https://grafana.com/docs/mimir/latest/query/query-best-practices/
- Functions do not reduce samples read: `foo` and `sum(foo)` cost the same, `sum(foo)/count(foo)` reads twice. A step shorter than the scrape interval re-counts samples. Each sample read is 10 MQU.
  https://grafana.com/docs/grafana-cloud/platform/pricing-and-usage/metrics/

## Label APIs

- Default: `/api/v1/labels` and `/api/v1/label/{name}/values`. Use the cardinality APIs only to count values or series.
  https://grafana.com/docs/mimir/latest/query/query-metric-labels/
- Always pass `start` and `end`, otherwise Mimir queries the whole retention period or up to `-store.max-labels-query-length`. Ranges round to block boundaries (2 h recent, up to 24 h older).
  https://grafana.com/docs/mimir/latest/query/query-metric-labels/
- Cardinality APIs take no time range: they read only in-memory ingester series.
  https://grafana.com/docs/mimir/latest/query/query-metric-labels/
- Slow label values with a selector matching a few thousand series: use `/api/v1/series`.
  https://grafana.com/docs/mimir/latest/query/query-metric-labels/

## HTTP API access

- Basic auth: user = metrics instance ID, password = token with `metrics:read`. Prometheus calls use the query endpoint from the Prometheus card. Rules go through `https://<STACK_URL>/api/prometheus/<DATASOURCE_UID>/api/v1/rules` (UID can be `grafanacloud-prom`).
  https://grafana.com/docs/grafana-cloud/send-data/metrics/metrics-prometheus/query-http-api/
- Mimir-only endpoints: take the remote write URL, strip `/api/prom/push`, append `/prometheus`.
  ```bash
  curl -s -u "$LOGIN" "$PROMETHEUS_HTTP_PREFIX/api/v1/cardinality/label_names"
  ```
  https://grafana.com/docs/grafana-cloud/send-data/metrics/metrics-prometheus/query-http-api/

## Query editor and `$__rate_interval`

- Query type defaults to "Both" (2 queries). Pick Range for graphs, Instant for stat, gauge and single-row tables. Instant queries show no exemplars.
  https://grafana.com/docs/grafana/latest/datasources/prometheus/query-editor/
- Alert queries do not accept `$variables`. Ranges align to the step, which can leave a gap at the right edge.
  https://grafana.com/docs/grafana/latest/datasources/prometheus/query-editor/
- `increase()` returning fractions is interpolation, not a bug. A `rate()` that grows forever means several targets write one series without a unique label.
  https://grafana.com/docs/grafana/latest/datasources/prometheus/query-editor/
- Always `[$__rate_interval]` in `rate()` and `increase()`, never fixed `[5m]` or `$__interval`. It equals `max($__interval + scrape_interval, 4 * scrape_interval)`.
  https://grafana.com/docs/grafana/latest/datasources/prometheus/template-variables/
- The data source scrape interval defaults to `15s`. With a real 60 s scrape, `rate()` returns nothing: set the real value.
  https://grafana.com/docs/grafana/latest/datasources/prometheus/template-variables/
- Recording rules use a fixed range such as `[5m]`.
  https://grafana.com/docs/grafana/latest/datasources/prometheus/template-variables/
- Multi-value variables become a regex: `{job=~"$job"}`, never `=`. The Filters variable applies to every query of the data source.
  https://grafana.com/docs/grafana/latest/datasources/prometheus/template-variables/
- Variable from a range query: `query_result(topk(5, sum(rate(http_requests_total[$__range])) by (instance)))` with Regex `/"([^"]+)"/`.
  https://grafana.com/docs/grafana/latest/datasources/prometheus/template-variables/

## Billing

- Active series = sample in the last 20 min, see `grafanacloud_instance_active_series` in `grafanacloud-usage`. Billed on max(active series, DPM / included DPM) at the 95th percentile.
  https://grafana.com/docs/grafana-cloud/platform/pricing-and-usage/metrics/
- 60 s scrape = 1 DPM, 15 s = 4 DPM: halving the interval doubles cost. Native histogram buckets count 0.25 series each.
  https://grafana.com/docs/grafana-cloud/platform/pricing-and-usage/metrics/
- Cost Management gauges are month-to-date, so `increase()` on them can go negative.
  https://grafana.com/docs/grafana-cloud/platform/pricing-and-usage/metrics/
- Cardinality management dashboards cover one stack and the last 2 h. "Unused" = not queried in 30 days and in no dashboard or rule.
  https://grafana.com/docs/grafana-cloud/platform/cost-management-and-billing/analyze-costs/metrics-costs/prometheus-metrics-costs/cardinality-management/

## Default limits

Exposed as `grafanacloud_instance_metrics_limits{limit_name=...}`. Time limits are recorded in nanoseconds (`max_partial_query_length` is exported in seconds).
https://grafana.com/docs/grafana-cloud/platform/pricing-and-usage/usage-limits/

| Limit | Default |
| --- | --- |
| `ingestion_rate` / `ingestion_burst_size` | 10,000 samples/s / 200,000 |
| `max_global_series_per_user` | 150,000 |
| `max_global_exemplars_per_user` | 100,000 |
| `out_of_order_time_window` | 2 h |
| `max_partial_query_length`, `max_labels_query_length` | 32 days (clipped) |
| `max_fetched_chunks_per_query` | 2,000,000 |
| `max_query_expression_size_bytes` | 128 KiB after variable substitution |
| `ruler_max_rule_groups_per_tenant` / `ruler_max_rules_per_rule_group` | 35 / 20 |

Ingestion, series, metadata and ruler limits autoscale. Others need Support.
https://grafana.com/docs/grafana-cloud/platform/pricing-and-usage/usage-limits/

## Adaptive Metrics

- On by default. Recommends aggregations from 30 days of usage. With no metric metadata and no type inferable from name or usage, the default recommendation supports only sum, count, avg and sum(rate()).
  https://grafana.com/docs/grafana-cloud/observe-and-act/adaptive-telemetry/adaptive-metrics/introduction/
- By default, samples later than 90 s behind the newest sample of the aggregated series drop (`aggregator-sample-too-old`). Raise `aggregation_delay` for lagging clients.
  https://grafana.com/docs/grafana-cloud/observe-and-act/adaptive-telemetry/adaptive-metrics/introduction/
- Rule:
  ```json
  {"metric": "http_requests_total_", "match_type": "prefix", "drop_labels": ["instance"], "aggregations": ["sum:counter"]}
  ```
  `drop_labels` XOR `keep_labels`. Use `sum:counter` for counters: `sum` is not useful for them. Exact match beats prefix, then first rule wins. Set `aggregation_interval` and `aggregation_delay` together.
  https://grafana.com/docs/grafana-cloud/observe-and-act/adaptive-telemetry/adaptive-metrics/create-rules/define-metrics-aggregation-rules/
- Aggregate rather than `"drop": true`: a drop hides the metric and its impact can no longer be computed.
  https://grafana.com/docs/grafana-cloud/observe-and-act/adaptive-telemetry/adaptive-metrics/create-rules/define-metrics-aggregation-rules/
- Querying aggregated metrics: the bare name errors, list labels with `group without () (metric)`. `rate`, `irate`, `increase` need `sum:counter`. Selectors without a metric name, such as `count({job="x"})`, can fail when they match aggregated data. Raw-plus-aggregated ranges return nothing during transition. Bypass with `__aggregation__!="none"` or header `X-Query-Aggregations: false`.
  https://grafana.com/docs/grafana-cloud/observe-and-act/adaptive-telemetry/adaptive-metrics/troubleshoot-your-aggregated-metrics-query/

## Old patterns

| Old | Current | Source |
| --- | --- | --- |
| Classic query variable `label_values(metric, label)` | Label values, Label names, Metrics, Query result types | https://grafana.com/docs/grafana/latest/datasources/prometheus/template-variables/ |
| `[[var]]` (legacy, still supported) | `$var` or `${var}` | https://grafana.com/docs/grafana/latest/datasources/prometheus/template-variables/ |
| Ad hoc filters | Filters variable | https://grafana.com/docs/grafana/latest/datasources/prometheus/template-variables/ |
