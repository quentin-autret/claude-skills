# Context propagation and sampling

## Contents

- Propagation basics
- Propagator configuration
- Security of propagated context
- Sampling: when and how
- SDK head samplers
- Collector sampling

## Propagation basics

- The default propagator is W3C TraceContext, header `traceparent`, format `<version>-<trace-id>-<parent-id>-<trace-flags>`, e.g. `00-a0892f3577b34da6a3ce929d0e0e4736-f03067aa0ba902b7-01`. https://opentelemetry.io/docs/concepts/context-propagation/#propagation
- Instrumentation libraries propagate transparently. Use the Propagators API (inject on sender, extract on receiver) only for protocols no library covers. https://opentelemetry.io/docs/concepts/context-propagation/#custom-context-propagation
- Span context (trace ID, span ID, trace flags, trace state) is immutable and is what gets serialized. https://opentelemetry.io/docs/concepts/signals/traces/#span-context
- SDKs can stamp trace and span IDs onto log records, correlating logs with traces automatically. https://opentelemetry.io/docs/concepts/context-propagation/#logs

## Propagator configuration

- `OTEL_PROPAGATORS` default `tracecontext,baggage`. Values: `tracecontext`, `baggage`, `b3`, `b3multi`, `jaeger`, `xray`, `ottrace`, `none`. https://opentelemetry.io/docs/languages/sdk-configuration/general/#otel_propagators
- The value is the complete list: setting `OTEL_PROPAGATORS="b3"` alone drops W3C `tracecontext` and `baggage`. List all you need, e.g. `tracecontext,baggage,b3`. (Follows from the list semantics and the default; the page does not warn.) https://opentelemetry.io/docs/languages/sdk-configuration/general/#otel_propagators
- For a managed platform, ship `tracecontext` as the primary propagator, legacy formats as secondary; priority follows the configured order. https://opentelemetry.io/docs/guidance/blueprints/managed-telemetry-platforms-for-k8s-workloads/#2-include-organization-standards-into-default-extensible-application-level-configuration-action-2
- Language setup: Go `otel.SetTextMapPropagator(propagation.TraceContext{})`; a propagator must be registered with the API to send trace context over the wire (B3 is also supported via `go.opentelemetry.io/contrib/propagators/b3`). https://opentelemetry.io/docs/languages/go/instrumentation/#propagators-and-context
- Browser and web clients inject `traceparent` and `tracestate`; backends must propagate them. A server can hand its trace to a page with `<meta name="traceparent" content="00-<traceId>-<spanId>-01">`. https://opentelemetry.io/docs/platforms/client-apps/ , https://opentelemetry.io/docs/languages/js/getting-started/browser/#example-application

## Security of propagated context

- Incoming context from untrusted sources can be forged: ignore or sanitize it at the edge. https://opentelemetry.io/docs/concepts/context-propagation/#external-services
- Outgoing trace IDs, span IDs and baggage reveal internal architecture: configure propagators not to send context to external or public endpoints. https://opentelemetry.io/docs/concepts/context-propagation/#external-services
- Avoid putting credentials, API keys or PII in baggage: it may be logged or sent to untrusted downstream services. https://opentelemetry.io/docs/concepts/context-propagation/#baggage

## Sampling: when and how

- Terms: "sampled" means processed and exported; "not sampled" means not exported. https://opentelemetry.io/docs/concepts/sampling/#terminology
- Consider sampling if any of these hold: 1000 or more traces per second, mostly healthy traffic with little variation, or clear criteria (errors, high latency, domain rules) for interesting traces. Avoid it with tens of small traces per second or fewer, when you only use aggregates, or when regulation forbids dropping data. https://opentelemetry.io/docs/concepts/sampling/#when-to-sample , https://opentelemetry.io/docs/concepts/sampling/#when-not-to-sample
- Head sampling decides as early as possible, without inspecting the whole trace. The most common form is consistent probability sampling on the trace ID, which keeps whole traces and can run at any point in the pipeline, but cannot guarantee keeping traces with errors. https://opentelemetry.io/docs/concepts/sampling/#head-sampling
- Tail sampling decides after seeing most spans (errors, latency, attributes). Almost always needed in large systems, but stateful, hard to operate and monitor. https://opentelemetry.io/docs/concepts/sampling/#tail-sampling
- Combine: head sampling to protect the pipeline for very high-volume services, then tail sampling later. https://opentelemetry.io/docs/concepts/sampling/#tail-sampling

## SDK head samplers

- `OTEL_TRACES_SAMPLER` default `parentbased_always_on`. Values: `always_on`, `always_off`, `traceidratio`, `parentbased_always_on`, `parentbased_always_off`, `parentbased_traceidratio`, `parentbased_jaeger_remote`, `jaeger_remote`, `xray`. https://opentelemetry.io/docs/languages/sdk-configuration/general/#otel_traces_sampler
- `OTEL_TRACES_SAMPLER_ARG` is read only when `OTEL_TRACES_SAMPLER` is set; for ratio samplers it is a probability in [0..1], default 1.0. Invalid or unrecognized input is logged as an error. https://opentelemetry.io/docs/languages/sdk-configuration/general/#otel_traces_sampler_arg

```sh
export OTEL_TRACES_SAMPLER="parentbased_traceidratio"
export OTEL_TRACES_SAMPLER_ARG="0.1"
```

https://opentelemetry.io/docs/languages/sdk-configuration/general/#otel_traces_sampler_arg

- Go: the default is `ParentBased` wrapping `AlwaysSample`; in production consider `ParentBased` wrapping `TraceIDRatioBased`. `ParentBased` follows the parent decision; the wrapped sampler only decides for root spans. https://opentelemetry.io/docs/languages/go/sampling/
- Jaeger remote arg: `endpoint=http://localhost:14250,pollingIntervalMs=5000,initialSamplingRate=0.25`; `initialSamplingRate` only applies until a strategy is fetched. https://opentelemetry.io/docs/languages/sdk-configuration/general/#otel_traces_sampler_arg
- Do not set a sampler at all if you do not need to manage volume: the default samples 100%. https://opentelemetry.io/docs/languages/js/sampling/#default-behavior
- Custom samplers must copy the parent tracestate into the result, respect the parent decision if you want that (wrap in `ParentBased` or check `IsSampled()`), and stay cheap: `ShouldSample` runs synchronously for every span. https://opentelemetry.io/docs/languages/go/sampling/#custom-samplers
- Java sampler decisions: DROP (no data, `onEnd` not called), RECORD_ONLY (processors called, not exported), RECORD_AND_SAMPLE (exported). https://opentelemetry.io/docs/languages/java/sdk/#sampler

## Collector sampling

- Processors: `probabilistic_sampler` (contrib, core and K8s distributions) and `tail_sampling` (contrib and K8s distributions only). https://opentelemetry.io/docs/concepts/sampling/#collector , https://opentelemetry.io/docs/collector/components/processor/
- Use the same hash seed on every Collector running probabilistic sampling (option `hash_seed`, e.g. `hash_seed: 22`). https://opentelemetry.io/docs/collector/deploy/other/agent-to-gateway/#sampling-considerations , https://opentelemetry.io/docs/collector/configuration/
- Configure tail sampling on gateways only. Agents use the `load_balancing` exporter (`routing_key: traceID`) to send all spans of a trace to one gateway instance. Prefer one well-resourced tail-sampling gateway unless you have robust sticky routing. https://opentelemetry.io/docs/collector/deploy/other/agent-to-gateway/#sampling-considerations

```yaml
tail_sampling:
  policies:
    - name: errors-policy
      type: status_code
      status_code: { status_codes: [ERROR] }
    - name: probabilistic-policy
      type: probabilistic
      probabilistic: { sampling_percentage: 10 }
```

https://opentelemetry.io/docs/collector/deploy/other/agent-to-gateway/#example-gateway-configuration
