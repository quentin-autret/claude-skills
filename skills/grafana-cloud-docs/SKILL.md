---
name: grafana-cloud-docs
description: Grafana Cloud pitfalls, defaults, limits and current patterns from the Grafana, Mimir, Loki and Tempo docs, each tied to its docs URL. Use when sending OpenTelemetry data to Grafana Cloud, writing PromQL, LogQL or TraceQL, controlling cardinality and cost, designing alert rules, routing notifications, provisioning alerting, building Synthetic Monitoring checks, using the /apis dashboard API, choosing access policy tokens or service accounts, or migrating off API keys, the Loki Exporter and legacy /api routes.
metadata:
  version: "Grafana Cloud (continuous) plus OSS docs at latest"
  source: https://grafana.com/docs/
  fetched: "2026-11-02"
---

# Grafana Cloud from the official docs

Version covered: Grafana Cloud is continuously released with no version selector. OSS pages (Grafana, Mimir, Loki, Tempo) are read at latest.

Each claim carries its docs URL. Read the quick start, then one matching reference. See "Old patterns" for old config.

## Quick start

### Credentials

- Access policy tokens open telemetry and the Cloud API, never the instance HTTP API. Dashboards, folders, users and data sources need a service account token. Service account tokens and Cloud API access policy tokens go in `Authorization: Bearer <TOKEN>`.
  https://grafana.com/docs/grafana-cloud/platform/security-and-account-management/security-and-access/authentication-and-permissions/access-policies/ and https://grafana.com/docs/grafana-cloud/learn-and-build/developer-resources/api-reference/http-api/authentication/
- Neither token type expires by default. Set an expiry, one policy per task.
  https://grafana.com/docs/grafana-cloud/platform/security-and-account-management/security-and-access/authentication-and-permissions/access-policies/create-access-policies/ and https://grafana.com/docs/grafana-cloud/platform/security-and-account-management/security-and-access/authentication-and-permissions/service-accounts/
- Telemetry basic auth: user = numeric instance ID, password = access policy token with the right `*:write` or `*:read` scope.
  https://grafana.com/docs/grafana-cloud/send-data/traces/set-up/verify-ingestion/

### OTLP ingestion

- Copy the OTLP endpoint from the stack's OpenTelemetry tile. Regions created before 2026-01-15 use flat hostnames, later ones nested, so never build it from a template.
  https://grafana.com/docs/grafana-cloud/platform/security-and-account-management/account-management/region-url-formats/
- Signal-specific `OTEL_EXPORTER_OTLP_*_ENDPOINT` variables need the full `/otlp/v1/<signal>` path. The base variable gets it appended.
  https://grafana.com/docs/grafana-cloud/send-data/otlp/send-data-otlp/
- Metric names get `_` for dots and unit or `_total` suffixes. Resource attributes go to `target_info`, joined on `job` and `instance`.
  https://grafana.com/docs/grafana-cloud/send-data/otlp/otlp-format-considerations/

### Queries

- PromQL: always `[$__rate_interval]` and set the data source scrape interval to the real one (default `15s`). Multi-value variables need `=~`.
  https://grafana.com/docs/grafana/latest/datasources/prometheus/template-variables/
- Mimir label APIs: always pass `start` and `end`.
  https://grafana.com/docs/mimir/latest/query/query-metric-labels/
- LogQL: selector, then line filters, then parsers. `{app=~".*"}` is rejected, use `.+`. Filter `__error__ = ""` before `unwrap`.
  https://grafana.com/docs/loki/latest/query/bp-query/, https://grafana.com/docs/loki/latest/query/troubleshoot-query/ and https://grafana.com/docs/loki/latest/query/query_examples/
- TraceQL: conditions in one `{}` hold on one span. Scope every attribute. Prefer `&&`, as `||` is slower. Unscoped `duration` is the span's.
  https://grafana.com/docs/tempo/latest/traceql/construct-traceql-queries/ and https://grafana.com/docs/tempo/latest/traceql/tune-traceql-queries/

### Cost and limits

- Halving the scrape interval doubles data points per minute per series, and Pro includes 1 DPM per series, so the rest costs extra. `sum(foo)` reads as many samples as `foo`.
  https://grafana.com/docs/grafana-cloud/platform/pricing-and-usage/metrics/
- Loki streams take 15 labels at most by default. Put IDs in structured metadata.
  https://grafana.com/docs/grafana-cloud/send-data/logs/loki-limits/

### Alerting

- Grafana-managed rules by default. No Data and Error states fire separate `DatasourceNoData` / `DatasourceError` alerts with their own labels, so silences, mute timings and policies of the original alert may not apply to them.
  https://grafana.com/docs/grafana-cloud/observe-and-act/alert-and-measure-reliability/alerting/fundamentals/alert-rule-evaluation/nodata-and-error-states/
- Only labels route. A value in an annotation falls to the default policy.
  https://grafana.com/docs/grafana-cloud/observe-and-act/alert-and-measure-reliability/alerting/fundamentals/alert-rules/annotation-label/
- Siblings stop at the first matching policy unless "Continue matching siblings" is on. Mute timings attach to child policies only and are not inherited.
  https://grafana.com/docs/grafana-cloud/observe-and-act/alert-and-measure-reliability/alerting/fundamentals/notifications/notification-policies/ and https://grafana.com/docs/grafana-cloud/observe-and-act/alert-and-measure-reliability/alerting/configure-notifications/mute-timings/
- Cloud has no file provisioning. Provisioning the policy tree overwrites all policies.
  https://grafana.com/docs/grafana-cloud/observe-and-act/alert-and-measure-reliability/alerting/set-up/provision-alerting-resources/ and https://grafana.com/docs/grafana-cloud/observe-and-act/alert-and-measure-reliability/alerting/set-up/provision-alerting-resources/export-alerting-resources/

### Synthetic Monitoring

- Every selected probe hits the target at every interval, nearly at once.
  https://grafana.com/docs/grafana-cloud/observe-and-act/testing/synthetic-monitoring/introduction/
- MultiHTTP does not validate a 200 status automatically. Define assertions for each request.
  https://grafana.com/docs/grafana-cloud/observe-and-act/testing/synthetic-monitoring/create-checks/checks/multihttp/

### Dashboards

- The dashboard API addresses dashboards by `metadata.name` (the uid), not the `metadata.uid` field. The Cloud namespace is `stacks-<STACK_ID>`.
  https://grafana.com/docs/grafana-cloud/learn-and-build/developer-resources/api-reference/http-api/dashboard/ and https://grafana.com/docs/grafana-cloud/learn-and-build/developer-resources/api-reference/http-api/apis/

## Old patterns

| Old | Current | Source |
| --- | --- | --- |
| Instance API keys, `/api/auth/keys`, Terraform `grafana_api_key` | Service accounts and their tokens, `grafana_service_account` + `grafana_service_account_token` | https://grafana.com/docs/grafana-cloud/platform/security-and-account-management/security-and-access/authentication-and-permissions/service-accounts/migrate-api-keys/ |
| Legacy `/api` routes (deprecated in Grafana 13) | `/apis/<group>/<version>/namespaces/<ns>/...` | https://grafana.com/docs/grafana-cloud/learn-and-build/developer-resources/api-reference/http-api/apis/ |
| Loki Exporter for OTel logs | Native OTLP through `otlphttp` | https://grafana.com/docs/grafana-cloud/send-data/otlp/adopt-new-logs-format/ and https://grafana.com/docs/grafana-cloud/send-data/logs/collect-logs-with-otel/ |
| Deprecated `/api/v1/provisioning/*` endpoints (contact points, policies, template groups, mute timings, alert rules) | `/apis/notifications.alerting.grafana.app/...`, `/apis/rules.alerting.grafana.app/...` | https://grafana.com/docs/grafana-cloud/observe-and-act/alert-and-measure-reliability/alerting/set-up/provision-alerting-resources/http-api-provisioning/ and https://grafana.com/docs/grafana-cloud/learn-and-build/developer-resources/api-reference/http-api/api-legacy/alerting_provisioning/ |
| Classic condition | Reduce + Threshold | https://grafana.com/docs/grafana-cloud/observe-and-act/alert-and-measure-reliability/alerting/fundamentals/alert-rules/queries-conditions/ |
| `[[var]]` | `${var}` | https://grafana.com/docs/grafana-cloud/learn-and-build/visualizations/dashboards/variables/variable-syntax/ |
| Grafonnet (unsupported), Grizzly (removed) | Foundation SDK, Grafana CLI | https://grafana.com/docs/grafana-cloud/learn-and-build/as-code/ |
| `grafanactl` | `gcx` | https://grafana.com/docs/grafana-cloud/send-data/otlp/promote-resource-attributes/ |
| Synthetic Monitoring legacy alerts | Per-check alerts | https://grafana.com/docs/grafana-cloud/observe-and-act/testing/synthetic-monitoring/configure-alerts/ |

## Reference files

| File | Read it for |
| --- | --- |
| [references/otlp-ingestion.md](references/otlp-ingestion.md) | OTLP endpoint, metric renaming, `target_info`, ingestion errors, OTLP logs |
| [references/traces-and-traceql.md](references/traces-and-traceql.md) | trace limits, sampling, metrics-generator, exemplars, TraceQL |
| [references/metrics-promql-and-cardinality.md](references/metrics-promql-and-cardinality.md) | PromQL cost, label APIs, billing, limits, Adaptive Metrics |
| [references/logs-and-logql.md](references/logs-and-logql.md) | LogQL order, parsers, Cloud Logs limits, billing |
| [references/alert-rules.md](references/alert-rules.md) | rule types, states, No Data and Error, labels, recording rules |
| [references/notifications-and-alerting-provisioning.md](references/notifications-and-alerting-provisioning.md) | contact points, policies, mute timings, templates, provisioning |
| [references/synthetic-monitoring.md](references/synthetic-monitoring.md) | check types, MultiHTTP, k6 checks, probes, uptime, alerts |
| [references/dashboards-and-as-code.md](references/dashboards-and-as-code.md) | variables, JSON model, dashboard API, Terraform, Foundation SDK |
| [references/access-tokens-and-apis.md](references/access-tokens-and-apis.md) | access policies, scopes, service accounts, `/apis` paths |

Sources and read details are listed in [SOURCES.md](SOURCES.md).
