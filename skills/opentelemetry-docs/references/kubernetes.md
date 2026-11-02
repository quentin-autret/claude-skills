# Kubernetes

## Contents

- What to deploy
- Kubernetes components and their pitfalls
- Collector Helm chart
- Operator install and CRs
- Auto-instrumentation with the Operator
- Autoscaling and the Target Allocator

## What to deploy

- Two Collectors: a DaemonSet agent per node (OTLP from apps, kubeletstats, filelog, k8sattributes) and a single-replica Deployment for cluster-wide data (k8s_cluster, k8sobjects events). More replicas of cluster-wide receivers duplicate data. https://opentelemetry.io/docs/platforms/kubernetes/getting-started/
- The Kubernetes blueprint builds on `opentelemetry-kube-stack` (Operator plus Collector CRs), configured through `collectors.*` presets. There the cluster-wide receivers run on the DaemonSet with leader election, or on a separate cluster Collector when leader election is not available. https://opentelemetry.io/docs/guidance/blueprints/k8s-observability-blueprint/#1-deploy-the-opentelemetry-kube-stack
- Send app telemetry to a Collector, not straight to a backend. https://opentelemetry.io/docs/platforms/kubernetes/operator/automatic/

## Kubernetes components and their pitfalls

- `k8sattributes`: any Collector receiving app data should use it. It associates pods by request IP by default; rules are ordered. RBAC: get/watch/list on `pods`, `namespaces` and `replicasets` (more resources for more extracted attributes, for example `deployments` for `k8s.deployment.name`). https://opentelemetry.io/docs/platforms/kubernetes/collector/components/
- Missing RBAC silently skips enrichment. Behind a proxy the pod IP is lost: enable pass-through on the proxy, or send `k8s.pod.uid` and use association rules. In a gateway each pod caches cluster metadata, so memory grows with cluster size. https://opentelemetry.io/docs/guidance/blueprints/managed-telemetry-platforms-for-k8s-workloads/#appendix-5
- `kubeletstats` collects pod and node metrics by default; add `container` in `metric_groups`. Prefer it to `hostmetrics` for CPU and memory; with both, disable the `cpu` and `memory` scrapers of `hostmetrics`. https://opentelemetry.io/docs/platforms/kubernetes/collector/components/ , https://opentelemetry.io/docs/guidance/blueprints/k8s-observability-blueprint/#3-prefer-kubeletstats-for-resource-usage-use-hostmetrics-only-for-gaps
- `hostmetrics` for the node: mount host `/` at `/hostfs` and set `root_path: /hostfs`. https://opentelemetry.io/docs/platforms/kubernetes/collector/components/
- `k8s_cluster` reports only the `Ready` node condition by default: add others in `node_conditions_to_report`. Status-reason metrics (`k8s.pod.status_reason`) are opt-in. https://opentelemetry.io/docs/platforms/kubernetes/collector/components/ , https://opentelemetry.io/docs/guidance/blueprints/k8s-observability-blueprint/#2-collect-cluster-state-metrics-with-the-clustermetrics-preset
- `filelog` reads `/var/log/pods/*/*/*.log` with the `container` operator; exclude the Collector's own logs. Prefer the `logsCollection` preset. https://opentelemetry.io/docs/platforms/kubernetes/collector/components/
- The Prometheus receiver rejects `remote_write`, `remote_read`, `rule_files` and alerting blocks. It does not shard across replicas. https://opentelemetry.io/docs/platforms/kubernetes/collector/components/
- Scrape node-local Prometheus endpoints from the DaemonSet (`spec.nodeName=${OTEL_K8S_NODE_NAME}`, `prometheus.io/scrape=true`). Do not also scrape them via Target Allocator or ServiceMonitor: double counts. https://opentelemetry.io/docs/guidance/blueprints/k8s-observability-blueprint/#6-scrape-prometheus-native-components-on-each-node-via-daemonset-mode
- Add tolerations so tainted, GPU and spot nodes get an agent. https://opentelemetry.io/docs/guidance/blueprints/k8s-observability-blueprint/#3-prefer-kubeletstats-for-resource-usage-use-hostmetrics-only-for-gaps

## Collector Helm chart

- `mode` is required (`daemonset`, `deployment`, `statefulset`). Use image `otel/opentelemetry-collector-k8s`. https://opentelemetry.io/docs/platforms/kubernetes/helm/collector/
- The chart only ships the `debug` exporter: data goes nowhere until you add an exporter to every pipeline. Overriding a pipeline means listing all its components, defaults included. Remove a default with `null`. https://opentelemetry.io/docs/platforms/kubernetes/helm/collector/

```yaml
mode: daemonset
image:
  repository: otel/opentelemetry-collector-k8s
presets:
  kubernetesAttributes:
    enabled: true
  kubeletMetrics:
    enabled: true
  logsCollection:
    enabled: true
config:
  exporters:
    otlp_http:
      endpoint: https://example.com:55681
  service:
    pipelines:
      logs:
        exporters: [otlp_http]
```

https://opentelemetry.io/docs/platforms/kubernetes/getting-started/ , https://opentelemetry.io/docs/platforms/kubernetes/helm/collector/

- `clusterMetrics` and `kubernetesEvents` presets are recommended with `mode: deployment` (or a statefulset) and a single replica (`replicaCount: 1`). https://opentelemetry.io/docs/platforms/kubernetes/getting-started/
- `logsCollection` with `includeCollectorLogs: true` and the `debug` exporter re-ingests its own output (log explosion). Presets are a start: configure the component by hand when you need more. https://opentelemetry.io/docs/platforms/kubernetes/helm/collector/

## Operator install and CRs

- Needs cert-manager for its webhooks, unless the Helm chart uses `admissionWebhooks.certManager.enabled=false` and `admissionWebhooks.autoGenerateCert.enabled=true`. https://opentelemetry.io/docs/platforms/kubernetes/helm/operator/
- `OpenTelemetryCollector` is `opentelemetry.io/v1beta1` with `config` as a YAML map (the Target Allocator page still shows `v1alpha1` with a string config). `Instrumentation` is `v1alpha1`. Default mode `deployment`; in that mode the Operator creates a Service `<name>-collector`. https://opentelemetry.io/docs/platforms/kubernetes/operator/ , https://opentelemetry.io/docs/platforms/kubernetes/operator/automatic/
- Default operator image is core `opentelemetry-collector`; build your own with ocb for other components. The Helm charts use `opentelemetry-collector-k8s`. https://opentelemetry.io/docs/platforms/kubernetes/operator/

## Auto-instrumentation with the Operator

- Create the `Instrumentation` BEFORE deploying the workload, or no init container is injected; redeploy afterwards. https://opentelemetry.io/docs/platforms/kubernetes/operator/troubleshooting/automatic/
- Annotate `spec.template.metadata.annotations` (not the Deployment's own metadata): `instrumentation.opentelemetry.io/inject-java: "true"` (also `dotnet`, `go`, `nodejs`, `python`, `sdk`). `"true"` searches the pod's namespace only; otherwise use `"<namespace>/<name>"`. https://opentelemetry.io/docs/platforms/kubernetes/operator/automatic/
- `spec.exporter.endpoint` defaults to `http://localhost:4317`, silently dropping data. Set it; Python, Java, .NET and Go use `http/protobuf` on 4318 (Python supports only 4318), Node.js `grpc` on 4317. https://opentelemetry.io/docs/platforms/kubernetes/operator/troubleshooting/automatic/

```yaml
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: demo-instrumentation
spec:
  exporter:
    endpoint: http://demo-collector:4318
  propagators: [tracecontext, baggage]
  sampler:
    type: parentbased_traceidratio
    argument: "1"
```

https://opentelemetry.io/docs/platforms/kubernetes/operator/automatic/

- Injection overrides `JAVA_TOOL_OPTIONS`, `PYTHONPATH`, `NODE_OPTIONS` set in the image or a ConfigMap. https://opentelemetry.io/docs/platforms/kubernetes/operator/troubleshooting/automatic/
- Only the first container is instrumented; choose others with `instrumentation.opentelemetry.io/container-names`. Several languages per pod need `enable-multi-instrumentation`. https://opentelemetry.io/docs/platforms/kubernetes/operator/automatic/
- Go (eBPF sidecar): feature gate `enable-go-instrumentation` is off; needs `instrumentation.opentelemetry.io/otel-go-auto-target-exe` and a privileged root sidecar. https://opentelemetry.io/docs/platforms/kubernetes/operator/automatic/
- Python: logs auto-instrumentation is disabled by default (set `OTEL_PYTHON_LOGGING_AUTO_INSTRUMENTATION_ENABLED` in `spec.python.env`); musl images need `instrumentation.opentelemetry.io/otel-python-platform: "musl"`. .NET on musl-based images: `otel-dotnet-auto-runtime: linux-musl-x64` (default `linux-x64`). https://opentelemetry.io/docs/platforms/kubernetes/operator/automatic/
- Resource precedence: `OTEL_RESOURCE_ATTRIBUTES`/`OTEL_SERVICE_NAME`, then `resource.opentelemetry.io/<attr>` annotations, then `app.kubernetes.io/*` labels (only with `defaults.useLabelsForResourceAttributes=true`), then attributes calculated from pod metadata, then `spec.resource.resourceAttributes` of the `Instrumentation`. First found wins, per attribute. https://opentelemetry.io/docs/platforms/kubernetes/operator/automatic/

## Autoscaling and the Target Allocator

- `spec.autoscaler` creates the HPA (modes `deployment`, `statefulset`); requires `spec.resources` and a Metrics Server (absent by default on EKS). https://opentelemetry.io/docs/platforms/kubernetes/operator/horizontal-pod-autoscaling/
- Target Allocator: `spec.targetAllocator.enabled: true` (the block alone does nothing), `prometheusCR.enabled: true` for ServiceMonitor/PodMonitor. It removes existing `static_configs` and injects `http_sd_configs`. https://opentelemetry.io/docs/platforms/kubernetes/operator/target-allocator/
- As of `v1beta1`, include `podMonitorSelector: {}` and `serviceMonitorSelector: {}` even if unused, or the Target Allocator discovers no targets from ServiceMonitors and PodMonitors. Selectors match the monitor's labels; `endpoints[].port` must be a NAMED Service port. https://opentelemetry.io/docs/platforms/kubernetes/operator/troubleshooting/target-allocator/
