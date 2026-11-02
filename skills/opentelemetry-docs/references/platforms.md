# Platforms: Lambda, Linux hosts, client apps, mainframes

## Contents

- AWS Lambda
- Linux hosts (system packages)
- Android
- Browser, iOS and other clients
- Mainframes

## AWS Lambda

- Community layers: per-language auto-instrumentation layers (Java, JavaScript, Python, Ruby) and a separate Collector layer, unlike ADOT, which bundles both. Instrumentation layers need the Collector layer or an external Collector; add the Collector layer first. https://opentelemetry.io/docs/platforms/faas/lambda-auto-instrument/ , https://opentelemetry.io/docs/platforms/faas/lambda-collector/
- Runtimes stated: Java 8, 11, 17; Node.js 18+; Python 3.9+; Ruby 3.2 and 3.3. https://opentelemetry.io/docs/platforms/faas/lambda-auto-instrument/#language-requirements
- Set `AWS_LAMBDA_EXEC_WRAPPER=/opt/otel-handler`. https://opentelemetry.io/docs/platforms/faas/lambda-auto-instrument/#configure-aws_lambda_exec_wrapper
- Layer ARNs are regional: replace `<region>` from the latest opentelemetry-lambda release. Publish a new function version after changes. https://opentelemetry.io/docs/platforms/faas/lambda-auto-instrument/#add-the-arn-of-instrumentation-lambda-layer
- Java cold start suffers: use provisioned concurrency and warm-up requests, and enable only the instrumentation you use: https://opentelemetry.io/docs/platforms/faas/lambda-auto-instrument/#language-requirements

```sh
OTEL_INSTRUMENTATION_COMMON_DEFAULT_ENABLED=false
OTEL_INSTRUMENTATION_AWS_LAMBDA_ENABLED=true
OTEL_INSTRUMENTATION_AWS_SDK_ENABLED=true
```

- Exporter defaults differ: Java `localhost:4317`, JavaScript hard-codes `http/protobuf` on 4318 and ignores `OTEL_EXPORTER_OTLP_PROTOCOL`, Python and Ruby 4318 over HTTP. https://opentelemetry.io/docs/platforms/faas/lambda-auto-instrument/#configure-your-sdk-exporters
- The Collector layer's default config only has the `debug` exporter: nothing reaches a backend until you supply a config. Keep the auth token in a Lambda env var referenced from the config. https://opentelemetry.io/docs/platforms/faas/lambda-collector/#update-the-default-exporters
- Point to a custom config with `OPENTELEMETRY_COLLECTOR_CONFIG_URI=/var/task/collector.yaml`, or `s3://<bucket>.s3.<region>.amazonaws.com/collector_config.yaml` (IAM read access needed). Providers: `file`, `env`, `yaml`, `http`, `https`, `s3`. https://opentelemetry.io/docs/platforms/faas/lambda-collector/#create-a-custom-configuration-file

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 'localhost:4317'
      http:
        endpoint: 'localhost:4318'
exporters:
  awsxray:
service:
  pipelines:
    traces:
      receivers: [otlp]
      exporters: [awsxray]
```

https://opentelemetry.io/docs/platforms/faas/lambda-collector/#create-a-custom-configuration-file

- The config path can be set in the Lambda console, the AWS CLI (`aws lambda update-function-configuration --function-name Function --environment Variables={OPENTELEMETRY_COLLECTOR_CONFIG_URI=/var/task/collector.yaml}`) or a CloudFormation `Environment.Variables` block. https://opentelemetry.io/docs/platforms/faas/lambda-collector/#custom-collector-configuration-using-the-cli
- Languages without a layer: instrument manually and add the Collector layer. https://opentelemetry.io/docs/platforms/faas/lambda-manual-instrument/

## Linux hosts (system packages)

- The `opentelemetry` metapackage installs the OpenTelemetry Injector (hooks the dynamic linker so Java, .NET, Node.js and Python processes load auto-instrumentation) plus language packages. Subsets: `opentelemetry-java`, `-nodejs`, `-dotnet`, `-python`. https://opentelemetry.io/docs/platforms/linux/ , https://opentelemetry.io/docs/platforms/linux/installation/
- Early, not for production: repos on GitHub Pages, packages unsigned, install snippets disable signature checks. The Collector is not included. https://opentelemetry.io/docs/platforms/linux/#status-and-limitations
- Running processes are instrumented only after restart. Default export is OTLP to localhost 4317/4318: run a local Collector. https://opentelemetry.io/docs/platforms/linux/
- Default config route: a declarative file. Copy `/etc/opentelemetry/<language>/otel-config.yaml`, then set `OTEL_CONFIG_FILE=/etc/opentelemetry/config.yaml` in `/etc/opentelemetry/injector/default_env.conf`. https://opentelemetry.io/docs/platforms/linux/configuration/
- .NET also needs `OTEL_EXPERIMENTAL_FILE_BASED_CONFIGURATION_ENABLED=true`, otherwise the file is silently ignored. https://opentelemetry.io/docs/platforms/linux/configuration/
- Variables in `default_env.conf` apply to EVERY instrumented process, so a service name there is global. Restart apps after changes. https://opentelemetry.io/docs/platforms/linux/configuration/

## Android

- OpenTelemetry Android, built on OpenTelemetry Java, needs Android SDK 21+ and a Gradle project using Kotlin (Java may be possible). Use the BOM (docs show 1.8.0). https://opentelemetry.io/docs/platforms/client-apps/android/#gradle-setup

```kotlin
implementation(platform("io.opentelemetry.android:opentelemetry-android-bom:1.8.0"))
implementation("io.opentelemetry.android:android-agent")
```

- Initialize in `Application.onCreate()` with `OpenTelemetryRumInitializer.initialize(context, configuration = { httpExport { baseUrl = "..." } })`. Export is OTLP over HTTP. All instrumentations are on by default (lifecycle, ANR, crashes, slow and frozen frames, clicks); disable individually, e.g. `slowRendering { enabled(false) }`. https://opentelemetry.io/docs/platforms/client-apps/android/#initialize-the-agent
- Batching and offline disk buffering are on by default; still sample. Redact attributes, keep PII out of span names. https://opentelemetry.io/docs/platforms/client-apps/android/#resource-constraints
- From the emulator, the host Collector is `http://10.0.2.2:4318`, not `localhost`. https://opentelemetry.io/docs/platforms/client-apps/android/#testing
- Manual spans: `openTelemetryRum.openTelemetry.getTracer("com.example.myapp")`; OkHttp via `OkHttpTelemetry.builder(openTelemetryRum.openTelemetry).build().newCallFactory(...)`. https://opentelemetry.io/docs/platforms/client-apps/android/#manual-instrumentation

## Browser, iOS and other clients

- Client apps run on devices you do not control: plan for battery and CPU, offline buffering, sessions, consent and sampling at scale. https://opentelemetry.io/docs/platforms/client-apps/#key-differences-from-server-side-instrumentation
- Inject `traceparent` and `tracestate` from the client and propagate them in backends. https://opentelemetry.io/docs/platforms/client-apps/#connecting-to-your-backend
- The iOS and Web platform pages have no content yet. https://opentelemetry.io/docs/platforms/client-apps/ios/ , https://opentelemetry.io/docs/platforms/client-apps/web/

## Mainframes

- The Collector often runs off-platform (Linux or containers) as the bridge; parse SMF, SYSLOG and job logs into OpenTelemetry signals, and correlate at integration tiers (API gateways, MQ bridges). The Mainframes SIG is still defining common terminology and identifying gaps in semantic conventions, SDKs and Collector components. https://opentelemetry.io/docs/platforms/mainframe/
