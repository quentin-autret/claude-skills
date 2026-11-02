# Instrumenting a library

## Contents

- Dependencies and versions
- Getting a tracer
- What to trace
- Context propagation in a library
- Performance and testing

## Dependencies and versions

- A library uses ONLY the OpenTelemetry API, never the SDK. Without an SDK in the app, every call is a no-op. https://opentelemetry.io/docs/concepts/instrumentation/libraries/#opentelemetry-api
- Depend on the earliest stable API (1.0.*) and do not bump it unless you need a new feature, to minimize dependency conflicts. https://opentelemetry.io/docs/concepts/instrumentation/libraries/#libraries-should-only-use-the-opentelemetry-api
- While the instrumentation stabilizes, ship it as a separate package (own repo or contrib) so it never affects users who do not use it. Semantic conventions are "stable, but subject to evolution" and need periodic updates. https://opentelemetry.io/docs/concepts/instrumentation/libraries/#libraries-should-only-use-the-opentelemetry-api
- Register the instrumentation in the OpenTelemetry registry. https://opentelemetry.io/docs/concepts/instrumentation/libraries/#instrumentation-registry

## Getting a tracer

- Allow injecting a `TracerProvider` (easier tests) or use the global one, whichever is idiomatic. Pass the library name AND version when getting the tracer. https://opentelemetry.io/docs/concepts/instrumentation/libraries/#getting-a-tracer
- Set `schema_url` to record the semantic conventions version. Keep span names meaningful and low-cardinality. If no convention fits, propose one upstream. https://opentelemetry.io/docs/concepts/instrumentation/libraries/#semantic-conventions

## What to trace

- Design spans from the library user's perspective, not the internals. https://opentelemetry.io/docs/concepts/instrumentation/libraries/#defining-spans
- Trace public methods doing network calls or significant fallible local work, and handlers processing requests or messages. https://opentelemetry.io/docs/concepts/instrumentation/libraries/#public-apis
- A DB client creates a span for the logical DB request only. The physical network request belongs to the library that does it. Serialization and similar steps become span events, not extra spans. https://opentelemetry.io/docs/concepts/instrumentation/libraries/#defining-spans
- Do not instrument when all apply: thin proxy over self-explanatory APIs, OpenTelemetry already instruments the underlying network calls, no conventions to enrich. "When in doubt, don't instrument." https://opentelemetry.io/docs/concepts/instrumentation/libraries/#when-not-to-instrument
- When OpenTelemetry does not trace your network client: if your library wraps a public, documented RPC API, instrument it and trace individual network tries. If those spans would be too verbose or costly, use logs or span events on the public API span; if they must be spans, put them behind an option disabled by default. https://opentelemetry.io/docs/concepts/instrumentation/libraries/#nested-network-and-other-spans
- Do not duplicate network spans OpenTelemetry already produces, except to support users without auto-instrumentation, custom or legacy propagation, or enriching RPC spans. https://opentelemetry.io/docs/concepts/instrumentation/libraries/#nested-network-and-other-spans
- Attach events to the span YOUR instrumentation created, not the "active span", which you do not control. https://opentelemetry.io/docs/concepts/instrumentation/libraries/#events

## Context propagation in a library

- Server side: extract with the propagator from the incoming request, use it as parent of a `SERVER` span, and make that span current. Several received messages become span links. https://opentelemetry.io/docs/concepts/instrumentation/libraries/#extracting-context

```java
Context extractedContext = propagator.extract(Context.current(), httpExchange, getter);
Span span = tracer.spanBuilder("receive").setSpanKind(SpanKind.SERVER).setParent(extractedContext).startSpan();
```

https://opentelemetry.io/docs/concepts/instrumentation/libraries/#extracting-context

- Client side: start a `CLIENT` span, make it current, then `propagator.inject(Context.current(), transportLayer, setter)`. https://opentelemetry.io/docs/concepts/instrumentation/libraries/#injecting-context
- Skip injection when the downstream rejects unknown metadata. With a custom correlation protocol, use a custom propagator or stamp correlation IDs on the span. https://opentelemetry.io/docs/concepts/instrumentation/libraries/#injecting-context
- In-process: capture the user's active context at the public API surface as early as possible, then propagate it explicitly inside the library, because the active span can change during callbacks, threads and background work. https://opentelemetry.io/docs/concepts/instrumentation/libraries/#in-process
- With a protocol that has no metadata field, the receiver must extract AND remove the context before processing the payload. https://opentelemetry.io/docs/concepts/context-propagation/#custom-context-propagation

## Performance and testing

- Pass sampling-relevant attributes at span creation. Compute expensive attributes only if the span is recording. https://opentelemetry.io/docs/concepts/instrumentation/libraries/#performance

```java
Span span = tracer.spanBuilder(String.format("SELECT %s.%s", dbName, collectionName))
        .setSpanKind(SpanKind.CLIENT)
        .setAttribute("db.name", dbName)
        .startSpan();
if (span.isRecording()) {
    span.setAttribute("db.statement", sanitize(query.statement()));
}
```

https://opentelemetry.io/docs/concepts/instrumentation/libraries/#performance

- Unit-test with an in-memory exporter (Java `InMemorySpanExporter` from `opentelemetry-sdk-testing`, behind a `SimpleSpanProcessor`), and try the library in a typical app with all tracing enabled. https://opentelemetry.io/docs/concepts/instrumentation/libraries/#testing
