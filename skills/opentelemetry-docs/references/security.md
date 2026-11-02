# Security

## Contents

- Binding and exposure
- Secrets, auth, privileges
- Scrubbing sensitive data
- SDK-side privacy defaults
- Security model and reporting

## Binding and exposure

- Bind server-like receivers and extensions to specific interfaces, not `0.0.0.0` (CWE-1327): `localhost`, a pod IP, or `127.0.0.1` / `::1` if `localhost` resolves oddly. Check both loopbacks in dual-stack setups. https://opentelemetry.io/docs/security/config-best-practices/#protect-against-denial-of-service-attacks
- Since Collector v0.110.0 component servers default to `localhost`; earlier versions default to `0.0.0.0` (feature gate `component.UseLocalHostAsDefaultHost`). https://opentelemetry.io/docs/security/config-best-practices/#protect-against-denial-of-service-attacks
- Containers: `localhost` often does not work. Docker: bind to the container hostname and publish with `-p 127.0.0.1:4567:4317`. Compose: the service name. Kubernetes DaemonSet: inject the pod IP with the Downward API. https://opentelemetry.io/docs/security/config-best-practices/#docker

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: ${env:MY_POD_IP}:4317
      http:
        endpoint: ${env:MY_POD_IP}:4318
```

https://opentelemetry.io/docs/security/config-best-practices/#docker

- Do not expose a Collector to the internet for browser telemetry; put a reverse proxy in front for TLS and CORS. https://opentelemetry.io/docs/languages/js/exporters/#securely-expose-your-collector
- Java autoconfigure property `otel.exporter.prometheus.host` (address the Prometheus metric server binds to) defaults to `0.0.0.0`. https://opentelemetry.io/docs/languages/java/configuration/#properties-exporters

## Secrets, auth, privileges

- The Collector config may hold tokens and TLS keys: store it on an encrypted filesystem or secret store and inject sensitive values with `${env:...}`. https://opentelemetry.io/docs/security/config-best-practices/#store-your-configuration-securely
- Enable authentication (for example bearer token or basic auth extensions) on server-like receivers and restrict the IPs the Collector runs on. https://opentelemetry.io/docs/security/hosting-best-practices/#control-access-to-server-like-components
- The Collector configuration should include both encryption (certificates) and authentication. https://opentelemetry.io/docs/security/config-best-practices/#use-encryption-and-authentication
- Default settings are not considered secure against availability attacks: require authentication on endpoints. https://opentelemetry.io/docs/security/security-response/#security-model
- Do not run the Collector as root. Grant privileged access only for what the data location needs (specific node log mounts), with RBAC. https://opentelemetry.io/docs/security/config-best-practices/#set-permissions-carefully , https://opentelemetry.io/docs/security/hosting-best-practices/#apply-the-principle-of-least-privilege
- Tune buffer, queue, payload and worker settings with care: improperly set values can expose the Collector to additional attack vectors. https://opentelemetry.io/docs/security/config-best-practices/#configure-with-care
- Protect Collector resources by batching, limiting available memory, sizing queues (`sending_queue.queue_size`), setting timeout and retry limits, and filtering unwanted telemetry. https://opentelemetry.io/docs/security/config-best-practices/#safeguard-resource-utilization
- Restrict access to directories holding stored telemetry. https://opentelemetry.io/docs/security/hosting-best-practices/#store-data-securely
- Java: read static keys from env, and for expiring tokens pass a `Supplier<Map<String,String>>` to `setHeaders`. https://opentelemetry.io/docs/languages/java/sdk/#authentication
- The Android example sets a bearer token through `httpExport { baseHeaders }` (`"Authorization" to "Bearer <token>"`). https://opentelemetry.io/docs/platforms/client-apps/android/

## Scrubbing sensitive data

- The implementer is responsible for privacy compliance, including what third-party instrumentation emits. The best protection is not collecting data. https://opentelemetry.io/docs/security/handling-sensitive-data/#your-responsibility
- `redaction` processor: deletes non-allowed keys FIRST, then masks values matching `blocked_values`. https://opentelemetry.io/docs/security/config-best-practices/#scrub-sensitive-data

```yaml
processors:
  redaction:
    allow_all_keys: false
    allowed_keys: [description, group, id, name]
    blocked_values:
      - '4[0-9]{12}(?:[0-9]{3})?'
    summary: debug
```

https://opentelemetry.io/docs/security/config-best-practices/#scrub-sensitive-data

- Hash or delete fields with `attributes` (`action: hash`, `action: delete`); replace with OTTL: https://opentelemetry.io/docs/security/handling-sensitive-data/#replacing-userid-with-userhash

```yaml
transform:
  trace_statements:
    - context: span
      statements:
        - set(attributes["user.hash"], SHA256(attributes["user.id"]))
        - delete_key(attributes, "user.id")
```

- Hashing small predictable inputs (numeric IDs) is reversible. Prefer truncation: keep year/month, the email domain, drop the last IPv4 octet: `replace_pattern(attributes["client.address"], "\\.\\d+$", ".0")`. https://opentelemetry.io/docs/security/handling-sensitive-data/#truncating-ip-addresses

## SDK-side privacy defaults

- Java agent DB statement sanitization is on (string and number values in the statement become `?`; property `otel.instrumentation.common.db-statement-sanitizer.enabled`, default true). https://opentelemetry.io/docs/zero-code/java/agent/instrumentation/#db-statement-sanitization
- Java `enduser.id`, `enduser.role`, `enduser.scope` capture is off by default. https://opentelemetry.io/docs/zero-code/java/agent/instrumentation/#capturing-enduser-attributes
- Python Redis query sanitization is opt-in (`OTEL_PYTHON_INSTRUMENTATION_SANITIZE_REDIS=true`). Captured headers can be masked with `OTEL_INSTRUMENTATION_HTTP_CAPTURE_HEADERS_SANITIZE_FIELDS=".*session.*,set-cookie"`. https://opentelemetry.io/docs/zero-code/python/configuration/#python-specific-configuration , https://opentelemetry.io/docs/zero-code/python/example/#sanitization-of-captured-headers
- Sensitive baggage items can reach unintended parties (third-party APIs): automatic instrumentation includes baggage in most network requests, in HTTP headers visible to anyone inspecting traffic. https://opentelemetry.io/docs/concepts/signals/baggage/#baggage-security-considerations

## Security model and reporting

- Only the latest minor version gets community support and security fixes: upgrade to receive them. https://opentelemetry.io/docs/security/security-response/#supported-versions
- DoS by properly authenticated clients is not treated as a vulnerability. https://opentelemetry.io/docs/security/security-response/#security-model
- Report through the repository's "Report a vulnerability" button or security@opentelemetry.io, never a public issue. https://opentelemetry.io/docs/security/security-response/#reporting-methods
- Linux system packages are unsigned and their install snippets disable signature checks (`[trusted=yes]`, `gpgcheck=0`); not for production. https://opentelemetry.io/docs/platforms/linux/#status-and-limitations , snippets at https://opentelemetry.io/docs/platforms/linux/installation/
