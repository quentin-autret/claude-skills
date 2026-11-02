# Logs and LogQL

LogQL rules for Grafana Loki and Grafana Cloud Logs: query order, syntax traps, metric queries, limits, HTTP access, billing.

## Contents

- Query order and speed
- Syntax traps
- Parsers
- Pipeline errors
- Metric queries
- Grafana Cloud limits
- HTTP access and errors
- Diagnosing write failures
- Billing

## Query order and speed

- Loki runs left to right: time range, stream selector, line filters, parsers, label filters. Parsers go after line filters:
  ```logql
  {job="mysql"} |= "error" | json | line_format "{{.err}}"
  ```
  https://grafana.com/docs/loki/latest/query/bp-query/
- Set the range outside the query (picker or `start`/`end`). One index file per day, so multi-day queries read several.
  https://grafana.com/docs/loki/latest/query/bp-query/
- Default `|=` and `!=`. Regex filters only as a last resort. Exact label matchers over regex.
  https://grafana.com/docs/loki/latest/query/bp-query/
- One regex instead of many ORs: `{cluster="prod", namespace=~"api|web"}`. Sample huge exploratory queries with `| sample 0.1`.
  https://grafana.com/docs/loki/latest/query/troubleshoot-query/

## Syntax traps

- A selector needs a positive matcher that cannot match empty. `{app=~".*"}`, `{foo!="bar"}` are rejected. Use `{app=~".+"}`.
  https://grafana.com/docs/loki/latest/query/troubleshoot-query/
- Selector regex is fully anchored and `.` skips newlines (use `(?s)`). Line-filter regex is not anchored and `.` matches newlines. RE2, case-sensitive, `(?i)` to ignore case. Backticks avoid escaping.
  https://grafana.com/docs/loki/latest/query/log_queries/
- Label filter predicates evaluate left to right, `and` has no priority over `or`: use parentheses, `| duration >= 20ms or (method="GET" and size <= 20KB)`. For duration and bytes comparisons the label value must carry a unit suffix.
  https://grafana.com/docs/loki/latest/query/log_queries/
- `label_format dst=src` drops `src`, `dst="{{.src}}"` keeps it. One name once per expression.
  https://grafana.com/docs/loki/latest/query/log_queries/
- `keep` never drops `__error__` and `__error_details__`. Use `drop`.
  https://grafana.com/docs/loki/latest/query/log_queries/
- Range durations accept `ns`, `us`, `ms`, `s`, `m`, `h`, `d`, `w`, `y`. `5minutes` fails to parse.
  https://grafana.com/docs/loki/latest/query/troubleshoot-query/

## Parsers

- Default `json` or `logfmt`. `pattern` beats `regexp` for odd formats. `regexp` needs a named group `(?P<name>re)`.
  https://grafana.com/docs/loki/latest/query/log_queries/
- Extracted keys are sanitized (`{"a.b": {"c": "d"}}` gives `a_b_c`). A clash with a stream label adds `_extracted`.
  https://grafana.com/docs/loki/latest/query/log_queries/
- `| json` skips arrays: `| json first_server="servers[0]"`.
  https://grafana.com/docs/loki/latest/query/log_queries/
- `logfmt` is best effort. `| logfmt --strict host, fwd_ip="fwd"` puts flags right after the parser.
  https://grafana.com/docs/loki/latest/query/log_queries/
- `pattern` is anchored at line start unless it begins with `<_>`. A pattern is invalid with no named capture or with two consecutive captures not separated by whitespace. Valid: `<_> msg="<method> <path> (<status>) <latency>"`.
  https://grafana.com/docs/loki/latest/query/log_queries/

## Pipeline errors

A failing stage keeps the line and sets `__error__`. Filter it with a label filter. Only label filters are allowed after `unwrap`, so the filter can go before or after it:
```logql
quantile_over_time(0.99, {container="ingress-nginx"} | json | __error__ = "" | unwrap request_time [1m]) by (path)
```
https://grafana.com/docs/loki/latest/query/query_examples/
https://grafana.com/docs/loki/latest/query/log_queries/

## Metric queries

- `offset` goes right after the range: `count_over_time({job="mysql"}[5m] offset 5m)`.
  https://grafana.com/docs/loki/latest/query/metric_queries/
- `sum_over_time` and siblings need `unwrap`. `count_over_time` must not have it.
  https://grafana.com/docs/loki/latest/query/troubleshoot-query/
- Unwrapped `sum_over_time`, `absent_over_time`, `rate` and `rate_counter` do not support `by`/`without`.
  https://grafana.com/docs/loki/latest/query/metric_queries/
- Empty alert query to 0: `sum(count_over_time({namespace="traefik"}[5m])) or vector(0)`. Missing logs: `absent_over_time`.
  https://grafana.com/docs/loki/latest/query/metric_queries/
- Result order is unspecified. `sort` works only on instant queries. `topk` keeps original labels.
  https://grafana.com/docs/loki/latest/query/metric_queries/
- Log queries cannot use `/loki/api/v1/query`: use `query_range` or wrap in a metric function. The series API takes stream selectors only.
  https://grafana.com/docs/loki/latest/query/troubleshoot-query/

## Grafana Cloud limits

- Line 256 KB (`line_too_long`, 400). Setting `max_line_size_truncate` to `true` through the self-serve configuration API truncates instead, which can break JSON parsing.
  https://grafana.com/docs/grafana-cloud/send-data/logs/loki-limits/
  https://grafana.com/docs/grafana-cloud/send-data/logs/troubleshoot/
- 15 labels per stream, fixed. Put IDs, IPs and trace IDs in structured metadata (128 entries, 64 KB per line).
  https://grafana.com/docs/grafana-cloud/send-data/logs/loki-limits/
- 5 MB/s per tenant (429 `rate_limited`), 5,000 active streams (429 `stream_limit`), both scale with usage. Push 64 MB decompressed.
  https://grafana.com/docs/grafana-cloud/send-data/logs/loki-limits/
- Query range 30 days and 1 hour, 500 unique series per metric query. The 5 min query timeout cannot be changed.
  https://grafana.com/docs/grafana-cloud/send-data/logs/loki-limits/
- Entries older than 1 h behind the newest of the same stream are rejected (`entry too far behind`). Several hosts on one stream: add a label to split it. Replays go oldest first.
  https://grafana.com/docs/grafana-cloud/send-data/logs/troubleshoot/
- Pushes take up to 1 s: send big batches in parallel (Alloy `loki.write`).
  https://grafana.com/docs/grafana-cloud/send-data/logs/troubleshoot/
- Self-managed defaults worth knowing: `max_entries_limit_per_query` 5000, `query_timeout` 1m (raise `server.http_server_*_timeout` with it).
  https://grafana.com/docs/loki/latest/query/troubleshoot-query/

## HTTP access and errors

- Token with `logs:read` (`logs:delete` for deletes). User = Loki instance ID from the Loki card.
  ```bash
  curl -s -u "<LOKI_INSTANCE_ID>:<TOKEN>" -G "<LOKI_URL>/loki/api/v1/query_range" --data-urlencode 'query={job="varlogs"}'
  ```
  https://grafana.com/docs/grafana-cloud/send-data/logs/authenticate-to-query-logs/
- 401 = wrong instance ID or bad token. 403 = missing scope or a label policy excluding the labels.
  https://grafana.com/docs/grafana-cloud/send-data/logs/authenticate-to-query-logs/
- Parse error 400, timeout 504 (narrow and retry), `no org id` (multi-tenancy without tenant ID: send `X-Scope-OrgID`). An empty 200 is no error: test with `{job=~".+"}`.
  https://grafana.com/docs/loki/latest/query/troubleshoot-query/

## Diagnosing write failures

In Explore, data source `grafanacloud-<instance>-usage-insights`: `{instance_type="logs"} |= "push request failed"`. Oversized requests and client OOM never appear there. `logcli series '{cluster="x"}' --analyze-labels` lists labels with their unique values.
https://grafana.com/docs/grafana-cloud/send-data/logs/troubleshoot/

## Billing

- Dimensions: GB processed (before Adaptive Telemetry), written, retained, queried above fair use. Queries up to 100x written volume per month are free.
  https://grafana.com/docs/grafana-cloud/platform/pricing-and-usage/logs/
- Retention minimum 14 days free, 30 days paid. Changes go through Support and are not retroactive. Adaptive drops lower written, not processed volume.
  https://grafana.com/docs/grafana-cloud/platform/pricing-and-usage/logs/
