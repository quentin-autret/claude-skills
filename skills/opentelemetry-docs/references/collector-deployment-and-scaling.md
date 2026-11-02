# Collector deployment, resiliency and scaling

## Contents

- Choosing a pattern
- Gateways and load balancing
- Resiliency: queues, retries, persistence
- Scaling and when not to
- Internal telemetry
- Common failures

## Choosing a pattern

- Run a Collector next to services in production: apps offload fast, the Collector does retries, batching, encryption and filtering. Direct-to-backend is fine for trials and dev. https://opentelemetry.io/docs/collector/#when-to-use-a-collector
- Agent (sidecar or DaemonSet): easy, 1:1 mapping, limited scalability. Gateway: one endpoint per cluster or region, central credentials and policy, one more thing that can fail. https://opentelemetry.io/docs/collector/deploy/agent/ , https://opentelemetry.io/docs/collector/deploy/gateway/
- Use agent-to-gateway only when you need central processing (tail sampling, transforms), network isolation, or cost control at scale. Skip it for small deployments without host telemetry or tail sampling. https://opentelemetry.io/docs/collector/deploy/other/agent-to-gateway/#when-to-use-this-pattern
- Use OTLP between agents and gateways, with Kubernetes service names as endpoints (`otel-gateway:4317`). https://opentelemetry.io/docs/collector/deploy/other/agent-to-gateway/#protocol-selection
- Keep backend endpoints and API keys at the gateway, read from env: `headers: api-key: ${env:BACKEND_API_KEY}`. https://opentelemetry.io/docs/collector/deploy/other/agent-to-gateway/#example-gateway-configuration

## Gateways and load balancing

- For OTLP/gRPC use an L7, gRPC-aware load balancer. L4 balancers and plain Kubernetes Services pin long-lived connections to one pod. https://opentelemetry.io/docs/collector/scaling/#scaling-stateless-collectors-and-using-load-balancers
- Alternatives: headless service with client-side balancing, service mesh, OTLP/HTTP, or connection recycling (new pods get traffic only as old connections expire): https://opentelemetry.io/docs/guidance/blueprints/managed-telemetry-platforms-for-k8s-workloads/#appendix-2

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        keepalive:
          server_parameters:
            max_connection_age: 60s
            max_connection_age_grace: 10s
```

- Stateful processing (tail sampling, spanmetrics, `cumulativetodelta`) needs all data of a trace or series in one Collector. Put a tier with the `load_balancing` exporter in front. `routing_key: traceID` for tail sampling, `service` for spanmetrics. https://opentelemetry.io/docs/collector/deploy/gateway/#load-balancing-exporter , https://opentelemetry.io/docs/collector/deploy/other/agent-to-gateway/#other-processing-considerations (cumulativetodelta)

```yaml
exporters:
  load_balancing:
    routing_key: traceID
    protocol:
      otlp:
    resolver:
      dns:
        hostname: otel-gateway-headless
        port: 4317
```

https://opentelemetry.io/docs/collector/deploy/other/agent-to-gateway/#example-agent-configuration-with-load-balancing

- Lower the DNS resolver interval in elastic environments. Monitor `otelcol_loadbalancer_num_backends` and `otelcol_loadbalancer_backend_latency`. https://opentelemetry.io/docs/collector/scaling/#scaling-stateful-collectors , https://opentelemetry.io/docs/collector/deploy/gateway/#load-balancing-exporter
- Single-writer rule: each metric stream needs one writer and a unique identity. Symptoms of violation: gaps, jumps, Prometheus `Error on ingesting out-of-order samples`. Add identity with `k8sattributes` and `resourcedetection`. https://opentelemetry.io/docs/collector/deploy/gateway/#multiple-collectors-and-the-single-writer-principle
- Chained Collectors hide errors: a gateway answers 200 before the backend accepts, so upstream sees success while data is rejected. https://opentelemetry.io/docs/guidance/reference-implementations/adobe/#the-chain-collector-problem

## Resiliency: queues, retries, persistence

- Every network exporter has `sending_queue` and `retry_on_failure`. Defaults: retry up to 300 s (`max_elapsed_time`), queue often 1000 batches. A full queue drops new data; after `max_elapsed_time` the oldest is dropped. https://opentelemetry.io/docs/collector/resiliency/#sending-queue-in-memory-buffering
- Persistent queue: `file_storage` extension, referenced in `sending_queue.storage`, listed in `service.extensions`. Survives restarts, not disk failure. https://opentelemetry.io/docs/collector/resiliency/#persistent-storage-write-ahead-log---wal

```yaml
extensions:
  file_storage:
    directory: /var/lib/otelcol/storage
exporters:
  otlp:
    endpoint: otlp.example.com:4317
    sending_queue:
      storage: file_storage
      queue_size: 5_000
    retry_on_failure:
      max_elapsed_time: 10m
service:
  extensions: [file_storage]
```

https://opentelemetry.io/docs/collector/resiliency/

- On Kubernetes `file_storage` forces a StatefulSet with PVCs and delays backpressure (`memory_limiter` triggers later). https://opentelemetry.io/docs/guidance/blueprints/managed-telemetry-platforms-for-k8s-workloads/#appendix-4
- Put resilience on network hops (agent to gateway, gateway to backend). SDK to a local agent usually needs none. https://opentelemetry.io/docs/collector/resiliency/#recommendations-for-preventing-data-loss
- Kafka between tiers only for critical hops; the producer still needs its own sending queue. https://opentelemetry.io/docs/collector/resiliency/#message-queues

## Scaling and when not to

- Scale up when `otelcol_processor_refused_spans` rises (with `memory_limiter`) or the exporter queue reaches 60-70% of capacity. Keep a few replicas minimum. https://opentelemetry.io/docs/collector/scaling/#when-to-scale
- Do NOT scale when the backend or network is the bottleneck or `otelcol_exporter_send_failed_spans` keeps rising: more Collectors only add pressure. https://opentelemetry.io/docs/collector/scaling/#when-not-to-scale
- Agents scale vertically (they run per host). Gateways scale both vertically and horizontally. The managed-platform blueprint advises avoiding default CPU-based autoscaling for gateways in favor of pipeline queue depth or memory consumption; the agent-to-gateway page itself mentions HPA on CPU or memory. https://opentelemetry.io/docs/collector/deploy/other/agent-to-gateway/#agents , https://opentelemetry.io/docs/guidance/blueprints/managed-telemetry-platforms-for-k8s-workloads/#guideline-3
- Replicas with the same Prometheus receiver config scrape the same targets twice. Shard per instance or use the Operator Target Allocator. https://opentelemetry.io/docs/collector/scaling/#scaling-the-scrapers

## Internal telemetry

- Defaults: Prometheus metrics on `127.0.0.1:8888/metrics`, logs on stderr. Its config follows the SDK declarative schema, still in development. https://opentelemetry.io/docs/collector/internal-telemetry/
- `service::telemetry::metrics::address` is ignored since v0.123.0. Use `readers`: https://opentelemetry.io/docs/collector/internal-telemetry/#service-address

```yaml
service:
  telemetry:
    metrics:
      readers:
        - periodic:
            exporter:
              otlp:
                protocol: http/protobuf
                endpoint: https://backend:4318
```

- Prometheus names: `_total` and unit suffixes are OFF by default (`without_type_suffix`, `without_units`); only component metrics carry `otelcol_`. https://opentelemetry.io/docs/collector/internal-telemetry/#metric-names
- Repeated internal log lines are sampled by default. Metric level `normal` by default. https://opentelemetry.io/docs/collector/internal-telemetry/#configure-internal-logs
- Watch `otelcol_exporter_queue_size` vs `_queue_capacity`, `otelcol_exporter_send_failed_*`, `otelcol_receiver_refused_*` (clients got errors), and the log `Dropping data because sending_queue is full`. https://opentelemetry.io/docs/collector/internal-telemetry/#use-internal-telemetry-to-monitor-the-collector

## Common failures

- Dropped data: undersized Collector or slow destination. Restarts: missing `memory_limiter`, queue larger than memory, container limits. https://opentelemetry.io/docs/collector/troubleshooting/#collector-exits-or-restarts
- Systemd installs read options from `/etc/otelcol/otelcol.conf` (`OTELCOL_OPTIONS`); restart after edits. https://opentelemetry.io/docs/collector/install/binary/linux/#automatic-service-configuration
- The Docker examples run an explicit version tag (`otel/opentelemetry-collector:<version>`, replace it with the version you want); mount config at `/etc/otelcol/config.yaml`. https://opentelemetry.io/docs/collector/install/docker/
- The Kubernetes example manifest is a starting point; use Helm charts or the Operator in production. https://opentelemetry.io/docs/collector/install/kubernetes/
