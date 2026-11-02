# Go

## Contents

- Status
- SDK bootstrap
- Traces
- Metrics and views
- Logs
- Resources, sampling, exporters
- Zero-code: otelc, OBI and the Auto SDK

## Status

- Traces and metrics stable, logs release candidate (still experimental, breaking changes possible). `stdoutlog`, `otlploghttp`, `otlploggrpc` and the Prometheus exporter are experimental. https://opentelemetry.io/docs/languages/go/ , https://opentelemetry.io/docs/languages/go/getting-started/ , https://opentelemetry.io/docs/languages/go/exporters/
- Getting started needs Go 1.23+; `otelc` needs Go 1.25+. https://opentelemetry.io/docs/languages/go/getting-started/ , https://opentelemetry.io/docs/zero-code/go/compile-time/getting-started/
- The docs know of no Go library with native OpenTelemetry support; use contrib instrumentation libraries (`go.opentelemetry.io/contrib/instrumentation/...`). https://opentelemetry.io/docs/languages/go/libraries/

## SDK bootstrap

- Apps initialize the SDK; libraries depend only on the API. Without a registered MeterProvider or LoggerProvider the metrics and logs APIs are no-ops. https://opentelemetry.io/docs/languages/go/instrumentation/
- Register propagator and providers globally, keep every shutdown function and call them on exit (the docs join errors with `errors.Join`). https://opentelemetry.io/docs/languages/go/getting-started/#initialize-the-opentelemetry-sdk

```go
otel.SetTextMapPropagator(propagation.NewCompositeTextMapPropagator(
	propagation.TraceContext{}, propagation.Baggage{}))
tp := trace.NewTracerProvider(trace.WithBatcher(exp), trace.WithResource(res))
otel.SetTracerProvider(tp)
defer tp.Shutdown(ctx)
```

https://opentelemetry.io/docs/languages/go/getting-started/

- Without a propagator, trace context does not cross process boundaries. https://opentelemetry.io/docs/languages/go/instrumentation/#propagators-and-context
- Defaults: batch timeout 5 s, periodic metric reader interval 1 min. https://opentelemetry.io/docs/languages/go/getting-started/
- Logger provider: the full getting-started example uses `global.SetLoggerProvider` from `go.opentelemetry.io/otel/log/global` (the instrumentation page writes `otel.SetLoggerProvider`). https://opentelemetry.io/docs/languages/go/getting-started/
- In servers, pass the signal-aware ctx with `BaseContext: func(net.Listener) context.Context { return ctx }`, then `srv.Shutdown` before the OTel shutdown. https://opentelemetry.io/docs/languages/go/getting-started/
- `OTEL_SDK_DISABLED` is not supported. https://opentelemetry.io/docs/languages/go/exporters/#automatic-exporter-configuration-with-environment-variables

## Traces

- `ctx, span := tracer.Start(ctx, "name")` returns a new context: pass it to children and outgoing calls, `defer span.End()`. A finished span is immutable. https://opentelemetry.io/docs/languages/go/instrumentation/#creating-spans
- `span.RecordError(err)` does NOT set the status. Also call `span.SetStatus(codes.Error, "...")`. https://opentelemetry.io/docs/languages/go/instrumentation/#record-errors

```go
if err != nil {
	span.SetStatus(codes.Error, "operationThatCouldFail failed")
	span.RecordError(err)
}
```

https://opentelemetry.io/docs/languages/go/instrumentation/#record-errors

- Precompute attribute keys: `var myKey = attribute.Key("myCoolAttribute")`. https://opentelemetry.io/docs/languages/go/instrumentation/#span-attributes
- `net/http` server: wrap the whole mux with `otelhttp.NewHandler(mux, "/")` for a server span plus `http.server.request.duration`. The operation string becomes the span name. Spans from `r.Context()` become children. https://opentelemetry.io/docs/languages/go/getting-started/#instrument-the-http-server

## Metrics and views

- The examples create instruments in package-level variables or `init()`, with a description and a unit (`s`, `By`, `{call}`, `{roll}`). https://opentelemetry.io/docs/languages/go/getting-started/ , https://opentelemetry.io/docs/languages/go/instrumentation/#using-counters
- Guard expensive attribute computation with `Enabled`: https://opentelemetry.io/docs/languages/go/instrumentation/#performance-optimization-for-synchronous-instruments

```go
if apiCounter.Enabled(ctx) {
    apiCounter.Add(ctx, 1, metric.WithAttributes(attributes...))
}
```

- Instrument creation returns `(Instrument, error)`: handle the error. https://opentelemetry.io/docs/languages/go/instrumentation/#using-counters
- `Int64Gauge.Record` is the synchronous gauge; observable gauges (callback invoked once per export cycle) are for non-additive values. `meter.RegisterCallback` returns a `Registration` to unregister before closing the observed resource (the registration example uses observable up-down counters). https://opentelemetry.io/docs/languages/go/instrumentation/#using-gauges , https://opentelemetry.io/docs/languages/go/instrumentation/#using-observable-async-updown-counters
- A matching view REPLACES the default view; several matching views produce several metrics. Custom `View` functions must copy Name, Description and Unit into the returned `Stream`. https://opentelemetry.io/docs/languages/go/instrumentation/#registering-views

```go
metric.NewView(metric.Instrument{Name: "latency"}, metric.Stream{Name: "request.latency"})
metric.Stream{Aggregation: metric.AggregationDrop{}}
metric.Stream{AttributeFilter: attribute.NewDenyKeysFilter("http.request.method")}
```

https://opentelemetry.io/docs/languages/go/instrumentation/#registering-views

## Logs

- There is no user-facing logs API: use a bridge (slog, logrus, zap, logr). Pass the span ctx (`logger.InfoContext(ctx, ...)` with `otelslog.NewLogger(name)`) to correlate records with traces. https://opentelemetry.io/docs/languages/go/instrumentation/#logs , https://opentelemetry.io/docs/languages/go/getting-started/#add-custom-instrumentation

## Resources, sampling, exporters

- Merge with the default resource to keep SDK attributes: `resource.Merge(resource.Default(), resource.NewWithAttributes(semconv.SchemaURL, semconv.ServiceName("svc")))`. https://opentelemetry.io/docs/languages/go/instrumentation/#getting-a-tracer
- `resource.New(ctx, resource.WithFromEnv(), resource.WithProcess(), resource.WithHost(), ...)`: `ErrPartialResource` and `ErrSchemaURLConflict` are non-fatal, log them and keep the resource. https://opentelemetry.io/docs/languages/go/resources/
- Default sampler `ParentBased(AlwaysSample())`; production: `sdktrace.WithSampler(sdktrace.ParentBased(sdktrace.TraceIDRatioBased(0.1)))`. https://opentelemetry.io/docs/languages/go/sampling/
- OTLP packages `otlptracehttp`, `otlptracegrpc`, `otlpmetrichttp`, `otlpmetricgrpc`, `otlploghttp`, `otlploggrpc`; they read most `OTEL_EXPORTER_OTLP_*` variables (endpoint, headers, timeout, compression). https://opentelemetry.io/docs/languages/go/exporters/
- `go.opentelemetry.io/contrib/exporters/autoexport` picks the exporter from `OTEL_TRACES_EXPORTER`, `OTEL_METRICS_EXPORTER`, `OTEL_LOGS_EXPORTER`: `autoexport.NewSpanExporter(ctx)`, `autoexport.NewMetricReader(ctx)`. https://opentelemetry.io/docs/languages/go/exporters/

## Zero-code: otelc, OBI and the Auto SDK

- `otelc` (compile-time, stable since v1.0.0): no privileges, any OS, deep in-process spans; needs the build pipeline, changes need rebuild. OBI (eBPF): separate process, no build change, Linux 5.8+ with BTF and elevated privileges, boundary-only. They can be combined. https://opentelemetry.io/docs/zero-code/go/choosing-auto-instrumentation/ , stable status: https://opentelemetry.io/docs/zero-code/go/compile-time/
- Build with `otelc go build -o myapp .`, or keep `go build` after `otelc setup` and `GOFLAGS="${GOFLAGS} '-toolexec=otelc toolexec'"`. In containers only the build stage needs otelc. https://opentelemetry.io/docs/zero-code/go/compile-time/getting-started/
- Flags go before the subcommand: `otelc --rules my-rules.yaml --debug go build -o myapp .`. https://opentelemetry.io/docs/zero-code/go/compile-time/configuration/
- Runtime: standard `OTEL_*` vars plus `OTEL_GO_ENABLED_INSTRUMENTATIONS=nethttp,grpc` / `OTEL_GO_DISABLED_INSTRUMENTATIONS`. Runtime metrics are on by default (`runtimemetrics` disables them). https://opentelemetry.io/docs/zero-code/go/compile-time/configuration/ , https://opentelemetry.io/docs/zero-code/go/compile-time/supported-libraries/
- No telemetry: check the binary was built with otelc, the library and its version are supported, `OTEL_EXPORTER_OTLP_ENDPOINT` is right (`OTEL_LOG_LEVEL=debug`). `otelc --debug` writes `debug.log` under `.otelc-build`; `otelc cleanup` resets odd builds. https://opentelemetry.io/docs/zero-code/go/compile-time/troubleshooting/#no-telemetry-is-produced
- With eBPF instrumentation, do NOT register a global TracerProvider: it breaks the Auto SDK correlation of manual and eBPF spans. Just call `otel.Tracer(...).Start(...)`; if needed `otel.SetTracerProvider(autosdk.TracerProvider())`. https://opentelemetry.io/docs/zero-code/go/autosdk/
- Since Go v1.36.0 the Auto SDK (`go.opentelemetry.io/auto/sdk`) is an indirect dependency of the API. https://opentelemetry.io/docs/zero-code/go/autosdk/#how-do-i-use-it
