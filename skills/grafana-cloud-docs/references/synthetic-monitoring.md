# Synthetic Monitoring

Checks, probes, k6 scripts, secrets, uptime math, alerts, Terraform and pricing for Grafana Cloud Synthetic Monitoring.

## Contents

- Execution model
- Choosing a check type
- Limits and defaults
- MultiHTTP
- k6 scripted and browser checks
- Secrets
- Probes and blocked traffic
- Uptime and reachability
- Alerts
- Troubleshooting
- Provisioning and API
- Pricing
- Old patterns

## Execution model

- Every selected probe runs the check at every interval, all at nearly the same time. 5 probes at 1 min = 5 hits per minute: targets must tolerate concurrent sessions and single-use auth.
  https://grafana.com/docs/grafana-cloud/observe-and-act/testing/synthetic-monitoring/introduction/
- Type identifiers (API `settings`, Terraform block, `check_name`): `http`, `ping`, `dns`, `tcp`, `traceroute`, `multihttp`, `scripted`, `browser`.
  https://grafana.com/docs/grafana-cloud/observe-and-act/testing/synthetic-monitoring/create-checks/checks/

## Choosing a check type

Default: the simplest type that proves the outcome, one HTTP check per independent target. Escalate only for an ordered journey. Auth: basic, bearer or client cert = HTTP with secrets. Secret in a custom header = scripted. Token or cookie from sign-in = MultiHTTP. HTML login form = browser.
https://grafana.com/docs/grafana-cloud/observe-and-act/testing/synthetic-monitoring/create-checks/checks/

## Limits and defaults

- HTTP, DNS, TCP, Ping, MultiHTTP: every 10-3600 s, timeout 1-60 s. Scripted and browser: 60-3600 s, timeout up to 180 s. Traceroute: 120 s minimum. 10 custom labels.
  https://grafana.com/docs/grafana-cloud/observe-and-act/testing/synthetic-monitoring/create-checks/checks/
- HTTP: request body 64KB max ("unexpected EOF" beyond). Valid status 2xx, SSL "ignore", IP `Any` prefers IPv6 and uses IPv4 only when no AAAA record exists, with no retry on another IP version if the connection fails. "Publish full set of metrics" defaults to false.
  https://grafana.com/docs/grafana-cloud/observe-and-act/testing/synthetic-monitoring/create-checks/checks/http/
- Regex validations fail the check when they MATCH. Use "invert match" to require a match.
  https://grafana.com/docs/grafana-cloud/observe-and-act/testing/synthetic-monitoring/create-checks/checks/http/

## MultiHTTP

- No automatic 200 check. Add assertions per request or failures never affect uptime.
  https://grafana.com/docs/grafana-cloud/observe-and-act/testing/synthetic-monitoring/create-checks/checks/multihttp/
- Variables are unique across requests, used as `${name}`. Private probe firewalls must allow `jslib.k6.io`.
  https://grafana.com/docs/grafana-cloud/observe-and-act/testing/synthetic-monitoring/create-checks/checks/multihttp/

## k6 scripted and browser checks

- One iteration of `default` per run. `vus`, `duration`, `stages`, `iterations` are ignored, thresholds unsupported. The k6 CLI cannot run or upload scripts to Synthetic Monitoring.
  https://grafana.com/docs/grafana-cloud/observe-and-act/testing/synthetic-monitoring/create-checks/checks/k6/
- No local files (`open`, `fs`): build the script with a bundler, then copy the output into the script option.
  https://grafana.com/docs/grafana-cloud/observe-and-act/testing/synthetic-monitoring/create-checks/checks/k6/
- Test-wide k6 tags (`options.tags`) appear on metrics k6 generates (`probe_checks_total`), not on agent metrics such as `probe_success`. A tag named like a reserved system label is not validated and its value is overwritten on metrics.
  https://grafana.com/docs/grafana-cloud/observe-and-act/testing/synthetic-monitoring/create-checks/checks/k6/
- If startup fails, `probe_duration_seconds` shows the full time up to the timeout.
  https://grafana.com/docs/grafana-cloud/observe-and-act/testing/synthetic-monitoring/create-checks/checks/k6/
- Register the wait before the action, never a fixed delay or `networkidle`:
  ```javascript
  const ratingsPromise = page.waitForResponse(/\/api\/ratings$/);
  await page.goto(ratingsUrl);
  const response = await ratingsPromise;
  if (response.status() !== 200 || (await response.json()).ratings.length !== 0) {
    fail('Expected an empty ratings list');
  }
  ```
  https://grafana.com/docs/grafana-cloud/observe-and-act/testing/synthetic-monitoring/create-checks/write-resilient-check-scripts/
- `check()` does not stop the script, `fail()` does. Close pages in `try/finally`. Rethrow caught errors.
  https://grafana.com/docs/grafana-cloud/observe-and-act/testing/synthetic-monitoring/create-checks/write-resilient-check-scripts/

## Secrets

- Checks reference a secret by name, and its value is not exposed in the UI, scripts or logs. HTTP supports secrets for password, bearer token, CA and client cert or key. Not for the username or custom headers.
  https://grafana.com/docs/grafana-cloud/observe-and-act/testing/synthetic-monitoring/create-checks/manage-secrets/
  ```js
  import secrets from 'k6/secrets';
  export default async function main() { const v = await secrets.get('test-secret-name'); }
  ```
  Locally: `k6 run --secret-source=mock=username=default,password=12345678 mytest.js`.
  https://grafana.com/docs/grafana-cloud/observe-and-act/testing/synthetic-monitoring/create-checks/manage-secrets/
- Deleting a secret breaks every check using it until one with the same name exists.
  https://grafana.com/docs/grafana-cloud/observe-and-act/testing/synthetic-monitoring/create-checks/manage-secrets/

## Probes and blocked traffic

A check failing from some locations only can mean a WAF, CDN or CAPTCHA blocks those probes' source IPs. Work with the owner of the blocking rule to allowlist the current probe source IP ranges, and keep the list current.
https://grafana.com/docs/grafana-cloud/observe-and-act/testing/synthetic-monitoring/create-checks/public-probes/

## Uptime and reachability

- Uptime = time points where at least 1 probe succeeded. Reachability = successful executions over all. 3 probes x 4 points can give 75% uptime and 50% reachability: high uptime with low reachability points to regional paths.
  https://grafana.com/docs/grafana-cloud/observe-and-act/testing/synthetic-monitoring/analyze-results/uptime-and-reachability/
- Time points without data are excluded, so uptime says nothing about a gap.
  https://grafana.com/docs/grafana-cloud/observe-and-act/testing/synthetic-monitoring/analyze-results/uptime-and-reachability/

## Alerts

- Default: per-check alerts (Grafana-managed): failed checks, TLS expiry (HTTP, TCP), latency (HTTP, DNS, Ping). Windows `5m` to `1h`, at least the frequency. Each pushes a threshold series.
  https://grafana.com/docs/grafana-cloud/observe-and-act/testing/synthetic-monitoring/configure-alerts/configure-per-check-alerts/
- All carry `namespace = synthetic_monitoring`: route with a child policy on it. Group by `instance` and `job` for one notification per check.
  https://grafana.com/docs/grafana-cloud/observe-and-act/testing/synthetic-monitoring/configure-alerts/configure-per-check-alerts/
- Custom rule, custom labels kept in `by ()`:
  ```promql
  1 - (sum by (instance, job) (rate(probe_all_success_sum{}[10m])) / sum by (instance, job) (rate(probe_all_success_count{}[10m]))) > 0.1
  ```
  https://grafana.com/docs/grafana-cloud/observe-and-act/testing/synthetic-monitoring/configure-alerts/configure-custom-alerts/

## Troubleshooting

- Read the messages before "Check failed" for the real failing step. The newest time point may still be pending.
  https://grafana.com/docs/grafana-cloud/observe-and-act/testing/synthetic-monitoring/analyze-results/troubleshoot-checks/
- A timeout alone does not identify the failing system, and a longer timeout may not fix it. Do not weaken an assertion solely to make the check succeed. Final logs can land later: widen the range.
  ```logql
  {job="YOUR_JOB", instance="YOUR_INSTANCE", probe="YOUR_PROBE"} | logfmt | execution_id="YOUR_EXECUTION_ID"
  ```
  https://grafana.com/docs/grafana-cloud/observe-and-act/testing/synthetic-monitoring/analyze-results/troubleshoot-checks/

## Provisioning and API

- Terraform `grafana_synthetic_monitoring_check` with the type block in `settings`:
  ```hcl
  settings { scripted { script = file("${path.module}/script.js") } }
  ```
  Export starting config from the plugin Config tab.
  https://grafana.com/docs/grafana-cloud/observe-and-act/testing/synthetic-monitoring/set-up/provision-synthetic-monitoring-resources/
- SM access tokens never expire and cannot be scoped: treat them as long-lived secrets.
  https://grafana.com/docs/grafana-cloud/observe-and-act/testing/synthetic-monitoring/set-up/provision-synthetic-monitoring-resources/
- REST: `GET https://<backend-address>/api/v1/check`, `Authorization: Bearer <SM token>`. The regional backend address is under Configuration > General.
  https://grafana.com/docs/grafana-cloud/observe-and-act/testing/synthetic-monitoring/api-reference/

## Pricing

Per 10,000 executions, API and browser apart. Monthly = probes x checks x run minutes x (43,200 / frequency): 1 check, 1 min, 3 probes = 129,600. Metrics and logs bill on top.
https://grafana.com/docs/grafana-cloud/platform/pricing-and-usage/synthetic-monitoring/

## Old patterns

| Old | Current | Source |
| --- | --- | --- |
| Legacy alerts (data source-managed) | Per-check alerts | https://grafana.com/docs/grafana-cloud/observe-and-act/testing/synthetic-monitoring/configure-alerts/ |
| `label_` custom label prefix | Un-prefixed labels (dual-write can duplicate notifications) | https://grafana.com/docs/grafana-cloud/observe-and-act/testing/synthetic-monitoring/configure-alerts/configure-custom-alerts/ |
