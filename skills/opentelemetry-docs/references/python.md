# Python

## Contents

- Status and packages
- Manual SDK setup
- Traces and metrics API
- Propagation and context
- Exporters
- Zero-code agent
- Python-specific environment variables
- Pre-fork servers and other pitfalls

## Status and packages

- Traces and metrics stable, logs in development (underscore modules `opentelemetry.sdk._logs`, `opentelemetry._logs`). Python 3.10+. https://opentelemetry.io/docs/languages/python/
- Packages `opentelemetry-api`, `opentelemetry-sdk`, `opentelemetry-exporter-*`, `opentelemetry-instrumentation-*` (contrib, versioned with a `b` suffix such as `0.65b0`). Libraries install only the API. https://opentelemetry.io/docs/languages/python/#installation , https://opentelemetry.io/docs/languages/python/instrumentation/
- mypy needs `namespace_packages = true`. https://opentelemetry.io/docs/languages/python/mypy/

## Manual SDK setup

```python
from opentelemetry import trace, metrics
from opentelemetry.sdk.resources import Resource, SERVICE_NAME
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.sdk.metrics import MeterProvider
from opentelemetry.sdk.metrics.export import PeriodicExportingMetricReader
from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter
from opentelemetry.exporter.otlp.proto.http.metric_exporter import OTLPMetricExporter

resource = Resource.create(attributes={SERVICE_NAME: "your-service-name"})
tracer_provider = TracerProvider(resource=resource)
tracer_provider.add_span_processor(BatchSpanProcessor(OTLPSpanExporter(endpoint="http://localhost:4318/v1/traces")))
trace.set_tracer_provider(tracer_provider)
reader = PeriodicExportingMetricReader(OTLPMetricExporter(endpoint="http://localhost:4318/v1/metrics"))
metrics.set_meter_provider(MeterProvider(resource=resource, metric_readers=[reader]))
```

https://opentelemetry.io/docs/languages/python/exporters/#usage , https://opentelemetry.io/docs/languages/python/instrumentation/#acquire-tracer

- The global tracer provider can be set only once. For a second provider, pass it explicitly: `trace.get_tracer("tracer.two", tracer_provider=other)`. https://opentelemetry.io/docs/languages/python/cookbook/#using-multiple-tracer-providers-with-different-resource
- Logs: stdlib `logging` is the application API. Attach a `LoggingHandler` bound to a `LoggerProvider` with `BatchLogRecordProcessor`. https://opentelemetry.io/docs/languages/python/instrumentation/#logs

## Traces and metrics API

- Default to `with tracer.start_as_current_span("name") as span:` (current and auto-closed); `start_span` does not make the span current. Decorator form `@tracer.start_as_current_span("do_work")`. https://opentelemetry.io/docs/languages/python/instrumentation/#creating-spans
- Errors: `span.set_status(Status(StatusCode.ERROR))` plus `span.record_exception(ex)`; catch specific exceptions. `Unset` already means success. https://opentelemetry.io/docs/languages/python/instrumentation/#record-exceptions-in-spans
- `trace.get_current_span().set_attribute(...)` enriches the auto-instrumented span. https://opentelemetry.io/docs/languages/python/instrumentation/#get-the-current-span
- Create instruments once at module level. Async callbacks return `Iterable[Observation]` and must respect `options.timeout_millis`. https://opentelemetry.io/docs/languages/python/instrumentation/#creating-and-using-asynchronous-instruments
- `OTEL_EXPORTER_OTLP_METRICS_TEMPORALITY_PREFERENCE`: unknown values log a warning and fall back to `CUMULATIVE`. https://opentelemetry.io/docs/languages/python/exporters/#console
- Semantic attributes come from `opentelemetry-semantic-conventions`; the page snippet contains a `//` comment, which is invalid Python. https://opentelemetry.io/docs/languages/python/instrumentation/#add-semantic-attributes

## Propagation and context

- Defaults `tracecontext,baggage`. Env vars override propagators set in code (`set_global_textmap(B3Format())`). https://opentelemetry.io/docs/languages/python/instrumentation/#using-sdk-apis
- Let instrumentations (Flask, Django, Celery) propagate. Manual: build a carrier dict, `TraceContextTextMapPropagator().extract(carrier=carrier)`, then `tracer.start_span("name", context=ctx)`. Use `.get()` on headers: the docs index `headers['Traceparent']` directly, which raises `KeyError` when absent. https://opentelemetry.io/docs/languages/python/propagation/#manual-context-propagation
- `context.attach(ctx)` returns a token: always `context.detach(token)` in `finally`. https://opentelemetry.io/docs/languages/python/cookbook/#manually-setting-span-context
- `baggage.set_baggage` returns a new Context; pass it with `context=`. https://opentelemetry.io/docs/languages/python/cookbook/#capturing-baggage-at-different-contexts

## Exporters

- `opentelemetry-exporter-otlp-proto-http` or `-proto-grpc`. HTTP endpoints in code include `/v1/<signal>`; gRPC takes `host:port`. https://opentelemetry.io/docs/languages/python/exporters/#otlp-dependencies
- Prometheus: prefer its OTLP receiver. `PrometheusMetricReader` with `start_http_server(port=9464)` is the alternative. https://opentelemetry.io/docs/languages/python/exporters/#prometheus
- Zipkin packages: `opentelemetry-exporter-zipkin-proto-http` or `-zipkin-json` (the tab labelled "Thrift" installs JSON). https://opentelemetry.io/docs/languages/python/exporters/#zipkin-dependencies

## Zero-code agent

```sh
pip install opentelemetry-distro opentelemetry-exporter-otlp
opentelemetry-bootstrap -a install
opentelemetry-instrument --service_name checkout --metrics_exporter none python myapp.py
```

https://opentelemetry.io/docs/zero-code/python/#setup

- A distro package is required. Install app dependencies before `opentelemetry-bootstrap`, which installs instrumentations for packages it finds. https://opentelemetry.io/docs/zero-code/python/#setup
- The `otlp` exporter uses gRPC (default `localhost:4317` per getting-started, `0.0.0.0:4317` per the configuration page; HTTP `4318`); `--exporter_otlp_protocol http/protobuf` switches to HTTP. https://opentelemetry.io/docs/languages/python/getting-started/ , https://opentelemetry.io/docs/zero-code/python/configuration/#configuration-properties
- A metrics exporter is currently required: pass `none` if you do not export metrics. https://opentelemetry.io/docs/zero-code/python/configuration/#configuration-properties
- Every CLI flag maps to an env var: uppercase plus `OTEL_` (`exporter_otlp_endpoint` to `OTEL_EXPORTER_OTLP_ENDPOINT`). https://opentelemetry.io/docs/zero-code/python/configuration/#environment-variables
- The agent pipeline is set only by env vars or flags. Programmatic instrumentation (`FlaskInstrumentor().instrument_app(app)`) and `opentelemetry-instrument` are mutually exclusive. https://opentelemetry.io/docs/zero-code/python/example/
- Agent spans cover edges; manual spans become their children. https://opentelemetry.io/docs/languages/python/getting-started/#add-manual-instrumentation-to-automatic-instrumentation
- Logs auto-instrumentation is on by default since SDK 1.40.0, controlled by `OTEL_PYTHON_LOG_AUTO_INSTRUMENTATION`. https://opentelemetry.io/docs/zero-code/python/configuration/#logging

## Python-specific environment variables

- `OTEL_PYTHON_EXCLUDED_URLS="client/.*/info,healthcheck"` (or per library, `OTEL_PYTHON_FLASK_EXCLUDED_URLS`) drops health checks. https://opentelemetry.io/docs/zero-code/python/configuration/#python-specific-configuration
- `OTEL_PYTHON_DISABLED_INSTRUMENTATIONS=redis,kafka,grpc_client` uses entry-point names, which can differ from package names. By default every installable instrumentation runs. https://opentelemetry.io/docs/zero-code/python/configuration/#python-specific-configuration
- `OTEL_INSTRUMENTATION_HTTP_CAPTURE_HEADERS_SERVER_REQUEST` captures headers as `http.request.header.<name>`; mask with `..._SANITIZE_FIELDS`. https://opentelemetry.io/docs/zero-code/python/example/#capture-http-request-and-response-headers
- `OTEL_PYTHON_LOG_CORRELATION`, `OTEL_PYTHON_LOG_FORMAT` inject trace context into logs. https://opentelemetry.io/docs/zero-code/python/configuration/#python-specific-configuration

## Pre-fork servers and other pitfalls

- Gunicorn or Uvicorn with several workers break metrics: forked children inherit the `PeriodicExportingMetricReader` thread and locks. Initialize OpenTelemetry after the fork. https://opentelemetry.io/docs/zero-code/python/troubleshooting/#pre-fork-server-issues
- Options: Gunicorn with `uvicorn.workers.UvicornWorker` (`opentelemetry-instrument gunicorn --workers 4 --worker-class uvicorn.workers.UvicornWorker myapp.main:app`), a `post_fork` hook calling `initialize()` from `opentelemetry.instrumentation.auto_instrumentation`, or a single worker. Do not use `PrometheusMetricReader` with forking. https://opentelemetry.io/docs/zero-code/python/troubleshooting/#workarounds
- For FastAPI, call `initialize()` before importing `FastAPI`. Never share one `service.instance.id` across workers. https://opentelemetry.io/docs/zero-code/python/troubleshooting/#workarounds
- Flask debug reloader breaks instrumentation: `use_reloader=False`. https://opentelemetry.io/docs/zero-code/python/troubleshooting/#flask-debug-mode-with-reloader-breaks-instrumentation
- uv: `uv run opentelemetry-bootstrap -a requirements | uv add --requirement -`, and redo it after each `uv sync`. https://opentelemetry.io/docs/zero-code/python/troubleshooting/#bootstrap-using-uv
- gevent apps: `OTEL_PYTHON_AUTO_INSTRUMENTATION_EXPERIMENTAL_GEVENT_PATCH=patch_all`. https://opentelemetry.io/docs/zero-code/python/configuration/#python-specific-configuration
- Slim images need `gcc` and `gcc-c++` for some packages (Debian/Ubuntu: `python3-dev build-essential`). https://opentelemetry.io/docs/zero-code/python/troubleshooting/#python-package-installation-failure
