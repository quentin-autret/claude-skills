# Semantic conventions and resources

## Contents

- Semantic conventions
- Opting in to stable conventions
- Resources and service identity
- Resource detectors
- Where an attribute belongs

## Semantic conventions

- Conventions give common names for spans, metrics, log attributes, profiles and resources. The benefit is one common naming scheme across a codebase, libraries and platforms. https://opentelemetry.io/docs/concepts/semantic-conventions/
- Current HTTP span attribute names: `http.request.method`, `url.path`, `url.query`, `url.scheme`, `server.address`, `server.port`, `http.route`, `http.response.status_code`, `client.address`, `network.protocol.version`, `user_agent.original`. https://opentelemetry.io/docs/concepts/observability-primer/#span-attributes
- Names like `http.method`, `http.target`, `http.flavor`, `net.peer.ip` and the metric `http.server.duration` are the old experimental conventions; several doc samples still show them. https://opentelemetry.io/docs/concepts/signals/traces/#spans , https://opentelemetry.io/docs/languages/js/getting-started/nodejs/ , https://opentelemetry.io/docs/languages/python/instrumentation/#semantic-convention-stability
- Prefer an HTTP instrumentation library that already uses the STABLE HTTP conventions and covers traces and metrics; add a manual span only if none exists. https://opentelemetry.io/docs/getting-started/reference-application-specification/
- When replacing existing instrumentation (the page covers OpenTracing), the attributes produced may differ from the old tags: update dashboards and alerts that relied on the old data. https://opentelemetry.io/docs/compatibility/migration/opentracing/#semantic-conventions
- Java: stable constants in `io.opentelemetry.semconv:opentelemetry-semconv`, experimental ones in `opentelemetry-semconv-incubating` (alpha permanently). Libraries may use incubating in tests but must not depend on it. https://opentelemetry.io/docs/languages/java/api/#semantic-attributes
- Go: the semconv package version is in the import path; the docs show `go.opentelemetry.io/otel/semconv/v1.43.0`. https://opentelemetry.io/docs/languages/go/instrumentation/
- Never invent a custom Kubernetes label when the K8s conventions already define one. https://opentelemetry.io/docs/guidance/blueprints/k8s-observability-blueprint/#5-use-k8sattributesprocessor-to-enrich-telemetry-with-required-and-optional-organizational-labels

## Opting in to stable conventions

- Instrumentations that predate stable HTTP and DB conventions keep emitting old names until you opt in with `OTEL_SEMCONV_STABILITY_OPT_IN`. Python values: `http`, `http/dup`, `database`, `database/dup`, `gen_ai_latest_experimental`. https://opentelemetry.io/docs/languages/python/instrumentation/#semantic-convention-stability
- `http` emits only the new names; `http/dup` emits both during a migration. If both are given, `/dup` wins. https://opentelemetry.io/docs/languages/python/instrumentation/#semantic-convention-stability

```sh
export OTEL_SEMCONV_STABILITY_OPT_IN=http/dup,database
```

https://opentelemetry.io/docs/languages/python/instrumentation/#semantic-convention-stability

- Java agent: database client metrics exist only when opting in to DB convention stability. https://opentelemetry.io/docs/zero-code/java/agent/supported-libraries/

## Resources and service identity

- A resource describes the entity producing telemetry. It is attached to the provider at creation and cannot change afterwards. https://opentelemetry.io/docs/concepts/resources/
- `service.name` defaults to `unknown_service`, so set it explicitly, in code or with `OTEL_SERVICE_NAME`. https://opentelemetry.io/docs/concepts/resources/#semantic-attributes-with-sdk-provided-default-value (The Java Spring Boot starter default is `unknown_service:java`: https://opentelemetry.io/docs/zero-code/java/spring-boot-starter/sdk-configuration/)
- `OTEL_SERVICE_NAME` wins over `service.name` inside `OTEL_RESOURCE_ATTRIBUTES`. https://opentelemetry.io/docs/languages/sdk-configuration/general/#otel_service_name

```sh
export OTEL_SERVICE_NAME="checkout"
export OTEL_RESOURCE_ATTRIBUTES="service.version=1.1,deployment.environment.name=staging"
```

https://opentelemetry.io/docs/languages/sdk-configuration/general/#otel_resource_attributes

- The environment attribute is `deployment.environment.name`. The non-Kubernetes blueprint still writes `deployment.environment`. https://opentelemetry.io/docs/concepts/resources/#custom-resources , https://opentelemetry.io/docs/guidance/blueprints/instrumenting-applications-and-processes-on-nonk8s-environments/
- Minimum organization-specific resource attributes recommended by the Kubernetes platform blueprint: `service.name`, `service.version`, `service.namespace` (or `service.owner`), `deployment.environment.name`. https://opentelemetry.io/docs/guidance/blueprints/managed-telemetry-platforms-for-k8s-workloads/#2-include-organization-standards-into-default-extensible-application-level-configuration-action-2
- `service.instance.id` must be unique per instance and regenerated when a resource attribute changes: a new UUID at each startup. Never share one value across workers or replicas (single writer for metrics). https://opentelemetry.io/docs/compatibility/prometheus/otlp-metrics-export/#use-environment-variables , https://opentelemetry.io/docs/zero-code/python/troubleshooting/#workarounds
- In JavaScript an attribute set both in code and in `OTEL_RESOURCE_ATTRIBUTES` takes the environment value. https://opentelemetry.io/docs/languages/js/resources/#adding-resources-in-code

## Resource detectors

- Most SDKs ship detectors for OS, host, process and runtime, container, Kubernetes and cloud. https://opentelemetry.io/docs/concepts/resources/#resource-detectors
- A custom detector list can lose the attributes set through environment variables. JavaScript: when setting `resourceDetectors`, also add `envDetector` (and `processDetector`). Java: when using `otel.java.enabled.resource-providers`, include `EnvironmentResourceProvider`. https://opentelemetry.io/docs/languages/js/resources/#container-resource-detection , https://opentelemetry.io/docs/languages/java/configuration/#properties-resource
- Collector side: `resourcedetection` (`detectors: [env, system, gcp, ec2, azure]`, adjust to the environment) and `k8sattributes` enrich telemetry with infrastructure metadata. https://opentelemetry.io/docs/collector/transforming-telemetry/#enriching-telemetry-with-resource-attributes

## Where an attribute belongs

- Resource: the producing entity (service, environment, region). Scope: the instrumenting library. Measurement or span attributes: the individual operation. https://opentelemetry.io/docs/concepts/signals/metrics/#what-the-limit-does-not-apply-to
- High-cardinality values (`user_id`, raw URL paths) impact backend performance, storage and Metric SDK memory. https://opentelemetry.io/docs/concepts/glossary/#cardinality , https://opentelemetry.io/docs/concepts/signals/metrics/#cardinality-limits
- An instrumentation scope is `(name, version, schema_url, attributes)`; spans from one tracer share it, and backends can filter by it to find which library version adds latency. https://opentelemetry.io/docs/concepts/instrumentation-scope/
