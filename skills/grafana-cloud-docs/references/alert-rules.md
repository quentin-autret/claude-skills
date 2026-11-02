# Alert rules

Grafana-managed alert and recording rules in Grafana Cloud: conditions, evaluation, No Data and Error, labels, quality policy, limits.

## Contents

- Rule type
- Conditions and expressions
- Evaluation and states
- No Data, Error, missing series
- Connectivity alerts
- Labels and annotations
- Recording rules
- Limits and performance
- Alert Advisor
- Cardinality alerts
- Old patterns

## Rule type

- Default: Grafana-managed rules. Data source-managed rules (Mimir, Loki) lack expressions, No Data/Error states, stale handling, images, RBAC, history and Terraform.
  https://grafana.com/docs/grafana-cloud/observe-and-act/alert-and-measure-reliability/alerting/alerting-rules/create-data-source-managed-rule/
- Grafana-managed rules query only data sources whose plugin sets `"backend": true, "alerting": true`. Query ranges must be fixed relative (`now-24hr: now`), never `now/d` or absolute.
  https://grafana.com/docs/grafana-cloud/observe-and-act/alert-and-measure-reliability/alerting/alerting-rules/create-grafana-managed-rule/
- Grafana-managed rules in a group evaluate concurrently with one timestamp. When alerts read recording rule output, define a rule sequence.
  https://grafana.com/docs/grafana-cloud/observe-and-act/alert-and-measure-reliability/alerting/fundamentals/alert-rule-evaluation/evaluation-within-a-group/

## Conditions and expressions

- One query or expression is the condition. Each series becomes an instance. Reduce series to one number. Tables need exactly one numeric column.
  https://grafana.com/docs/grafana-cloud/observe-and-act/alert-and-measure-reliability/alerting/fundamentals/alert-rules/queries-conditions/
- Use Math, Reduce, Resample, Threshold. Math joins series by identical labels. Threshold returns 1 to fire.
  https://grafana.com/docs/grafana-cloud/observe-and-act/alert-and-measure-reliability/alerting/fundamentals/alert-rules/queries-conditions/
- Recovery threshold stops flapping: fire at 1000 ms, recover at 900 ms, so 990 stays firing.
  https://grafana.com/docs/grafana-cloud/observe-and-act/alert-and-measure-reliability/alerting/fundamentals/alert-rules/queries-conditions/
- Smooth instead of raw thresholds:
  ```promql
  avg_over_time(cpu_usage[5m]) > 90
  ```
  https://grafana.com/docs/grafana-cloud/observe-and-act/alert-and-measure-reliability/alerting/guides/best-practices/

## Evaluation and states

- Notifications go out only on entering Alerting and on resolve. Recovering back to Alerting sends nothing new.
  https://grafana.com/docs/grafana-cloud/observe-and-act/alert-and-measure-reliability/alerting/fundamentals/alert-rule-evaluation/
- Editing a rule (except annotations and interval) resets instances to Normal. Pending 0 skips Pending, keep firing 0 skips Recovering.
  https://grafana.com/docs/grafana-cloud/observe-and-act/alert-and-measure-reliability/alerting/fundamentals/alert-rule-evaluation/
- Pause stops evaluation. Mute timings only stop notifications. A deleted UI-created resource cannot be retrieved (only admins can restore deleted rules): provision for backup.
  https://grafana.com/docs/grafana-cloud/observe-and-act/alert-and-measure-reliability/alerting/alerting-rules/create-grafana-managed-rule/

## No Data, Error, missing series

- No Data = zero series or all null. Error = failure or timeout (defaults: 30 s `evaluation_timeout`, 3 `max_attempts`). Both respect the pending period.
  https://grafana.com/docs/grafana-cloud/observe-and-act/alert-and-measure-reliability/alerting/fundamentals/alert-rule-evaluation/nodata-and-error-states/
- The default raises separate `DatasourceNoData` / `DatasourceError` alerts with labels `alertname`, `datasource_uid`, `rulename`. They are independent of the original alert, so its silences, mute timings and policies may not apply. Route them with one policy on `alertname=DatasourceError` grouped by `datasource_uid`.
  https://grafana.com/docs/grafana-cloud/observe-and-act/alert-and-measure-reliability/alerting/fundamentals/alert-rule-evaluation/nodata-and-error-states/
- Keep last state can hide prolonged data source outages: add alert rules that detect them, such as a dedicated data source health rule.
  https://grafana.com/docs/grafana-cloud/observe-and-act/alert-and-measure-reliability/alerting/fundamentals/alert-rule-evaluation/nodata-and-error-states/
- A series missing 2 evaluations (default) goes Normal (MissingSeries), resolves, then is evicted.
  https://grafana.com/docs/grafana-cloud/observe-and-act/alert-and-measure-reliability/alerting/fundamentals/alert-rule-evaluation/stale-alert-instances/
- Detect vanished series while keeping labels:
  ```promql
  present_over_time(http_request_latency_seconds{}[24h]) unless present_over_time(http_request_latency_seconds{}[10m])
  ```
  When no data is fine: `your_metric_query OR on() vector(0)`. Late data: query To `now-1m`.
  https://grafana.com/docs/grafana-cloud/observe-and-act/alert-and-measure-reliability/alerting/guides/missing-data/

## Connectivity alerts

- `up == 0` for 5m is fragile: one good scrape resets it. Smooth, then shorten pending to 0-1m:
  ```promql
  avg_over_time(up[10m]) < 0.8 or avg_over_time(probe_success[10m]) < 0.8
  ```
  https://grafana.com/docs/grafana-cloud/observe-and-act/alert-and-measure-reliability/alerting/guides/connectivity-errors/
- Health rule: cheap query, error handling Alerting, pending 5m. App rules then use Normal or Keep last state on error.
  https://grafana.com/docs/grafana-cloud/observe-and-act/alert-and-measure-reliability/alerting/guides/connectivity-errors/

## Labels and annotations

- Only labels route and silence. `team=backend` in an annotation falls to the default policy.
  https://grafana.com/docs/grafana-cloud/observe-and-act/alert-and-measure-reliability/alerting/fundamentals/alert-rules/annotation-label/
- `alertname`, `grafana_folder` and the `grafana_` prefix are reserved. Two rules with identical label sets lose one instance.
  https://grafana.com/docs/grafana-cloud/observe-and-act/alert-and-measure-reliability/alerting/fundamentals/alert-rules/annotation-label/
- Avoid query values in labels, use annotations: each distinct label set is a separate instance, and a templated label whose value changes maps to a different instance (the previous one goes stale). Templates use `$labels` and `$values`:
  ```go
  CPU usage for {{ $labels.instance }} has exceeded 80% ({{ $values.A.Value }}).
  ```
  https://grafana.com/docs/grafana-cloud/observe-and-act/alert-and-measure-reliability/alerting/alerting-rules/templates/

## Recording rules

- They store nothing locally. With a blank target, Cloud writes to `grafanacloud-prom`: select the target explicitly. Intervals snap to 10 s multiples.
  https://grafana.com/docs/grafana-cloud/observe-and-act/alert-and-measure-reliability/alerting/alerting-rules/create-recording-rules/create-grafana-managed-recording-rules/

## Limits and performance

- Free: 500 rules, 1000 instances per rule. Paid: soft 2000 rules, unlimited instances.
  https://grafana.com/docs/grafana-cloud/observe-and-act/alert-and-measure-reliability/alerting/alerting-rules/create-grafana-managed-rule/
- A rule returning more results than the limit fails with `query evaluation returned too many results: 12345 (limit: 10000)` and enters Error. Aggregate, filter or split. Self-managed: `alerting_rule_evaluation_results` in `[quota]`, default `-1` (unlimited). In Cloud, Grafana Labs manages the limit.
  https://grafana.com/docs/grafana-cloud/observe-and-act/alert-and-measure-reliability/alerting/set-up/performance-limitations/
- The Grafana Alertmanager receives only Grafana-managed alerts.
  https://grafana.com/docs/grafana-cloud/observe-and-act/alert-and-measure-reliability/alerting/set-up/performance-limitations/

## Alert Advisor

One quality policy per org, checks presence only. Detect-only first, then Enforced. Enforcement covers Terraform and the Alerting provisioning HTTP API (plus `kubectl` writes only with specific manifest fields). It is bypassed by `X-Disable-Provenance`, `disable_provenance = true` and the UI editor.
https://grafana.com/docs/grafana-cloud/observe-and-act/alert-and-measure-reliability/alerting/alerting-rules/alert-rule-quality/

## Cardinality alerts

```promql
(grafanacloud_instance_active_series
  / on (id) grafanacloud_instance_metrics_limits{limit_name="max_global_series_per_user"})
* on (id) group_left(name) grafanacloud_instance_info > 0.9
```
Wrap `count({__name__!=""})` in a recording rule. For one instance per metric, copy `__name__` with `label_replace`.
https://grafana.com/docs/grafana-cloud/observe-and-act/alert-and-measure-reliability/alerting/examples/high-cardinality-alerts/

## Old patterns

| Old | Current | Source |
| --- | --- | --- |
| Classic condition (one instance total) | Reduce + Threshold per series | https://grafana.com/docs/grafana-cloud/observe-and-act/alert-and-measure-reliability/alerting/fundamentals/alert-rules/queries-conditions/ |
| Recording rule `url`, `basic_auth_*` (gone in 12.1) | `[recording_rules] default_datasource_uid` or per-rule target | https://grafana.com/docs/grafana-cloud/observe-and-act/alert-and-measure-reliability/alerting/alerting-rules/create-recording-rules/create-grafana-managed-recording-rules/ |
| `alertingSaveStateCompressed` off (removed in 13.2) | Enable it and run one cycle before upgrading | https://grafana.com/docs/grafana-cloud/observe-and-act/alert-and-measure-reliability/alerting/set-up/performance-limitations/ |
