# Concepts and signals

## Contents

- Stability by signal and language
- API, SDK and instrumentation approaches
- Traces and spans
- Metrics and cardinality limits
- Logs
- Baggage
- Profiles

## Stability by signal and language

- Spec level: tracing API, SDK and protocol stable. Metrics API and protocol stable, SDK "mixed". Logs bridge API, SDK and protocol stable. Baggage API and SDK stable, with no OTLP and no Collector component. Profiles protocol in development (the profiles page says Alpha). https://opentelemetry.io/docs/specs/status/ , https://opentelemetry.io/docs/concepts/signals/profiles/
- The Collector status for a signal equals the OTLP status for that signal. https://opentelemetry.io/docs/specs/status/
- Languages are developed independently: check the language repo README and the spec compliance matrix for real support. https://opentelemetry.io/docs/specs/status/
- Traces / Metrics / Logs per language: C++, .NET, PHP stable/stable/stable. Java stable/stable/stable (profiles in development). Go stable/stable/release candidate. JavaScript and Python stable/stable/development. Erlang/Elixir, Ruby, Swift stable/development/development. Rust traces Beta, metrics and logs stable. Kotlin development for all. https://opentelemetry.io/docs/languages/#status-and-releases
- Whatever the API/SDK status, instrumentation built on Experimental semantic conventions can break the data flow. https://opentelemetry.io/docs/languages/#status-and-releases

## API, SDK and instrumentation approaches

- Code-based and zero-code instrumentation complement each other and can run together. Zero-code typically instruments the libraries you use (HTTP, DB, queues), not your own code. https://opentelemetry.io/docs/concepts/instrumentation/ , https://opentelemetry.io/docs/concepts/instrumentation/zero-code/
- A library or component consumed by an app depends ONLY on the API. A runnable service depends on API and SDK. Without an SDK the API is a no-op and very performant. https://opentelemetry.io/docs/concepts/instrumentation/code-based/ , https://opentelemetry.io/docs/concepts/instrumentation/libraries/#opentelemetry-api , https://opentelemetry.io/docs/concepts/instrumentation/libraries/#performance
- In most applications a provider (TracerProvider, MeterProvider) is initialized once; its lifecycle matches the application and its initialization includes Resource and exporter setup. The LoggerProvider is part of the Logs Bridge API, for logging library authors. https://opentelemetry.io/docs/concepts/signals/traces/#tracer-provider , https://opentelemetry.io/docs/concepts/signals/metrics/#meter-provider , https://opentelemetry.io/docs/concepts/signals/logs/#logger-provider
- Name a tracer or meter after what is instrumented (library: its fully qualified name, app: class or module) and pass a version. These form the instrumentation scope `(name, version, schema_url, attributes)`. https://opentelemetry.io/docs/concepts/instrumentation-scope/
- Layout required by the reference application specification: SDK initialization in a separate file imported by the app file when possible; the library file depends only on the API. https://opentelemetry.io/docs/getting-started/reference-application-specification/
- OTLP exporters emit OTel data without loss of information and are supported by Jaeger, Prometheus and most vendors; OTLP is the protocol for sending to the Collector. Using the Collector in production is a best practice. https://opentelemetry.io/docs/concepts/components/#exporters
- The API never throws and swallows invalid arguments, so instrumentation bugs stay hidden: test instrumentation with an in-memory exporter. https://opentelemetry.io/docs/concepts/instrumentation/libraries/#error-handling
- Diagnostic logging of SDK components: `OTEL_LOG_LEVEL` where implemented. https://opentelemetry.io/docs/getting-started/reference-application-specification/

## Traces and spans

- Prefer adding attributes at span creation (adding later is allowed) so SDK sampling can use them. https://opentelemetry.io/docs/concepts/signals/traces/#attributes
- Attribute keys are non-null strings. Values are non-null string, boolean, float, integer, or arrays of these. https://opentelemetry.io/docs/concepts/signals/traces/#attributes
- Status: `Unset` (default) already means success. `Ok` is a final override that nothing else should reinterpret. In most cases do not set `Ok`; set `Error` on failure. https://opentelemetry.io/docs/concepts/signals/traces/#span-status
- Record an exception and set `Error` status together (Java `recordException` + `setStatus(StatusCode.ERROR, ...)`, end in `finally`). https://opentelemetry.io/docs/concepts/instrumentation/libraries/#instrumentation-example
- Span kind default is `Internal`. `Client`/`Server` are synchronous remote calls ("synchronous" means not queued, unrelated to async/await). `Producer` creates a job processed later; `Consumer` processes it, possibly long after the producer ended. https://opentelemetry.io/docs/concepts/signals/traces/#span-kind
- Event vs attribute: if the timestamp matters, use a span event; an operation with start and end is a span. https://opentelemetry.io/docs/concepts/signals/traces/#when-to-use-span-events-versus-span-attributes
- Links express causality between spans, for example follow-up work queued and executed asynchronously (traces page), or several messages received at once (libraries page). https://opentelemetry.io/docs/concepts/signals/traces/#span-links , https://opentelemetry.io/docs/concepts/instrumentation/libraries/#extracting-context
- The JSON span examples on the traces page are illustrative, not OTLP/JSON (the page says so). The sample span also uses attribute names such as `http.method` and `net.peer.ip`; check current semantic conventions before copying them. https://opentelemetry.io/docs/concepts/signals/traces/#spans

## Metrics and cardinality limits

- Instruments: Counter, UpDownCounter, Gauge, Histogram, plus asynchronous Counter, UpDownCounter and Gauge. Asynchronous ones are collected once per export; use them when you only see the current or aggregated value. https://opentelemetry.io/docs/concepts/signals/metrics/#metric-instruments
- Views select instruments, change aggregation and filter reported attributes. https://opentelemetry.io/docs/concepts/signals/metrics/#views
- The SDK keeps one data point in memory per attribute combination. Default cardinality limit: 2000 combinations per metric stream, overridable with a View. https://opentelemetry.io/docs/concepts/signals/metrics/#cardinality-limits
- On overflow, extra combinations fold into one point with `otel.metric.overflow=true`. The ENTIRE attribute set is replaced: `{url.path=/checkout, success=false}` becomes `{otel.metric.overflow=true}`, so an error-rate alert on `success=false` can silently stop firing while totals stay right. Keep high-cardinality attributes (user IDs, raw paths) off metrics. https://opentelemetry.io/docs/concepts/signals/metrics/#cardinality-limits
- The limit does not apply to Resource or scope attributes. Do not move measurement attributes into the Resource to dodge it; constant per-process context (service, environment, region) belongs there. https://opentelemetry.io/docs/concepts/signals/metrics/#what-the-limit-does-not-apply-to
- With cumulative temporality, once the limit is reached new combinations overflow until process restart. With delta, the limit only bounds combinations active in one cycle. https://opentelemetry.io/docs/concepts/signals/metrics/#temporality-and-cardinality-limits

## Logs

- Application developers do NOT call the Logs Bridge API (LoggerProvider, Logger). Keep your logging library and add an appender or bridge. https://opentelemetry.io/docs/concepts/signals/logs/#log-appender--bridge
- With an SDK or autoinstrumentation active, existing logs get the active trace and span IDs. https://opentelemetry.io/docs/concepts/signals/logs/#opentelemetry-logs-for-applications
- An event is a log record with an event name and well-known structure. https://opentelemetry.io/docs/concepts/glossary/#event
- Structured means a stable schema, not just valid JSON. Prefer structured logs in production. https://opentelemetry.io/docs/concepts/signals/logs/#structured-logs
- Do not guard log calls with `logger.Enabled(...)` by default: the API is cheap when disabled. Guard only expensive, side-effect-free argument evaluation, and never cache the result. https://opentelemetry.io/docs/concepts/signals/logs/#checking-whether-logging-is-enabled
- Collector side: `filelogreceiver` reads files, `transformprocessor` reshapes records. https://opentelemetry.io/docs/concepts/signals/logs/#opentelemetry-logs-in-the-opentelemetry-collector

## Baggage

- Baggage is a key-value store propagated with context, for values known at request start (account ID, origin). https://opentelemetry.io/docs/concepts/signals/baggage/#what-should-otel-baggage-be-used-for
- Baggage is NOT copied onto spans, metrics or logs. Read it and add it explicitly, or use a Baggage Span Processor where the language ships one. https://opentelemetry.io/docs/concepts/signals/baggage/#baggage-is-not-the-same-as-attributes
- Auto-instrumentation sends baggage as HTTP headers on most outgoing requests, including third-party APIs, with no integrity check. Never put credentials or PII in it; treat incoming baggage as untrusted. https://opentelemetry.io/docs/concepts/signals/baggage/#baggage-security-considerations

## Profiles

- Profiles (Alpha) are stack-trace samples, optionally linked to a trace ID and sharing the Resource with other signals. Types: on-CPU, off-CPU, heap, allocations. https://opentelemetry.io/docs/concepts/signals/profiles/
- Collect with the official eBPF profiling agent for Linux (no code change), export via OTLP. Language integrations (JFR, pprof) are not available yet. https://opentelemetry.io/docs/concepts/signals/profiles/#collecting-profiles
