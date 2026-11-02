# Java agent, Spring Boot starter, Quarkus

## Contents

- Choosing agent, starter or Quarkus
- Launching the agent
- Configuration and precedence
- Suppressing and tuning instrumentation
- Annotations
- Declarative configuration
- Extensions
- Spring Boot starter specifics

## Choosing agent, starter or Quarkus

- Default: the Java agent, which has the most out-of-the-box instrumentation. Use the Spring Boot starter for native images, when agent startup overhead or another agent conflicts, or for `application.yaml` configuration. https://opentelemetry.io/docs/zero-code/java/spring-boot-starter/
- Quarkus uses its own `quarkus-opentelemetry` extension. Only tracing is on by default: `quarkus.otel.metrics.enabled=true`, `quarkus.otel.logs.enabled=true` (Quarkus 3.16.0+). https://opentelemetry.io/docs/zero-code/java/quarkus/
- The agent instruments edges; add `@WithSpan` or the API for your own code. https://opentelemetry.io/docs/zero-code/java/agent/

## Launching the agent

```sh
java -javaagent:path/to/opentelemetry-javaagent.jar -Dotel.service.name=your-service-name -jar myapp.jar
```

https://opentelemetry.io/docs/zero-code/java/agent/getting-started/

- Or `JAVA_TOOL_OPTIONS="-javaagent:..."` with `OTEL_SERVICE_NAME`. Application servers need the flag in their own JVM options file (Tomcat `setenv.sh` `CATALINA_OPTS`, GlassFish `asadmin create-jvm-options "-javaagent\:/path/..."` with escaped colon). https://opentelemetry.io/docs/zero-code/java/agent/server-config/
- Agent 2.0+ and the Spring Boot starter default the OTLP protocol to `http/protobuf`, unlike the plain SDK autoconfiguration (`grpc`). https://opentelemetry.io/docs/zero-code/java/agent/configuration/#sdk-configuration
- Debug: `-Dotel.javaagent.debug=true` (very verbose). https://opentelemetry.io/docs/zero-code/java/agent/getting-started/#troubleshooting
- In code under the agent: `GlobalOpenTelemetry.getOrNoop()`, `GlobalOpenTelemetry.getTracer("application")`. https://opentelemetry.io/docs/zero-code/java/agent/api/#opentelemetry

## Configuration and precedence

- Highest to lowest: system properties, env vars, properties file (`otel.javaagent.configuration-file`), `addPropertiesSupplier`. Env var: uppercase, `.` and `-` to `_`. https://opentelemetry.io/docs/zero-code/java/agent/configuration/#agent-configuration
- AWS, GCP and Azure resource providers are off by default (`otel.resource.providers.aws.enabled=true`). https://opentelemetry.io/docs/zero-code/java/agent/configuration/#enable-resource-providers-that-are-disabled-by-default
- `otel.javaagent.logging`: `simple` (default, stderr), `none`, `application` (app's slf4j; test before production). https://opentelemetry.io/docs/zero-code/java/agent/configuration/#java-agent-logging-output
- Predefined JMX metrics for servers are off: `-Dotel.jmx.target.system=jetty,tomcat`. https://opentelemetry.io/docs/zero-code/java/agent/server-config/#enable-predefined-jmx-metrics

## Suppressing and tuning instrumentation

- Disable one library: `otel.instrumentation.<name>.enabled=false` (`jdbc`, `kafka`, `spring-webmvc`). Disable the agent: `otel.javaagent.enabled=false`. https://opentelemetry.io/docs/zero-code/java/agent/disable/
- Allow-list mode (`otel.instrumentation.common.default-enabled=false`, then enable each) is advanced: you must enable transitive instrumentation dependencies yourself. https://opentelemetry.io/docs/zero-code/java/agent/disable/#enable-only-specific-instrumentation
- Manual-only: `-Dotel.instrumentation.common.default-enabled=false -Dotel.instrumentation.opentelemetry-api.enabled=true -Dotel.instrumentation.opentelemetry-instrumentation-annotations.enabled=true`. https://opentelemetry.io/docs/zero-code/java/agent/disable/#enable-manual-instrumentation-only
- Controller and view spans are off by default. Enable them with `otel.instrumentation.common.experimental.controller-telemetry.enabled=true`; never disable the whole web instrumentation to drop them, or you lose `http.route` and the span name. https://opentelemetry.io/docs/zero-code/java/agent/disable/#suppressing-controller-andor-view-spans
- `otel.instrumentation.experimental.span-suppression-strategy`: `semconv` (default), `span-kind`, `none` (duplicates, debugging only). https://opentelemetry.io/docs/zero-code/java/agent/disable/#instrumentation-span-suppression-behavior
- Overhead: sample, disable noisy JDBC and Redis instrumentation, raise `-Xmx`, never put `@WithSpan` on every method. Measure with warm-up and 2-10% errors. https://opentelemetry.io/docs/zero-code/java/agent/performance/
- HTTP: `otel.instrumentation.http.server.capture-request-headers=Header1`; unknown methods become `_OTHER` (`otel.instrumentation.http.known-methods`). https://opentelemetry.io/docs/zero-code/java/agent/instrumentation/http/
- `otel.instrumentation.messaging.experimental.receive-telemetry.enabled=true` makes consumers start a new trace linked to the producer only by a span link. https://opentelemetry.io/docs/zero-code/java/agent/instrumentation/#capturing-consumer-message-receive-telemetry-in-messaging-instrumentations

## Annotations

- Dependency `io.opentelemetry.instrumentation:opentelemetry-instrumentation-annotations`. `@WithSpan` name defaults to `<Class>.<method>`, kind `INTERNAL`, `inheritContext=true`. Async return types (CompletableFuture, Mono, Flux, RxJava) end the span when they complete. https://opentelemetry.io/docs/zero-code/java/agent/annotations/

```java
@WithSpan(kind = SpanKind.CLIENT, value = "my span name")
public void myMethod(@SpanAttribute("parameter1") String p) { }
```

- `@SpanAttribute` without a name works only with `javac -parameters`. https://opentelemetry.io/docs/zero-code/java/agent/annotations/#adding-attributes-to-the-span-with-spanattribute
- Without code change: `otel.instrumentation.methods.include=my.package.MyClass1[method1,method2]` (all overloads). Exclude: `otel.instrumentation.opentelemetry-instrumentation-annotations.exclude-methods`. https://opentelemetry.io/docs/zero-code/java/agent/annotations/
- Spring starter annotations rely on AOP proxies. https://opentelemetry.io/docs/zero-code/java/spring-boot-starter/annotations/

## Declarative configuration

- Agent 2.26.0+, experimental: `-Dotel.config.file=/path/otel-config.yaml` (the agent page uses `otel.config.file`). Substitution `${VAR:-default}`; other env vars are ignored. https://opentelemetry.io/docs/zero-code/java/agent/declarative-configuration/
- Durations in milliseconds only: `schedule_delay: ${OTEL_BSP_SCHEDULE_DELAY:-5000}` (`5s` errors). Use real YAML booleans and integers. https://opentelemetry.io/docs/zero-code/java/agent/declarative-configuration/#duration-format
- Still env/system-property only: `otel.javaagent.enabled`, `.debug`, `.extensions`, `.logging`, `.configuration-file`. https://opentelemetry.io/docs/zero-code/java/agent/declarative-configuration/#environment-variables-and-system-properties-only-options
- `otel.instrumentation.*` moves under `instrumentation/development.java.` with `-` to `_`; `default-enabled=false` becomes `instrumentation/development.java.agent.instrumentation_mode: none`. https://opentelemetry.io/docs/zero-code/java/agent/declarative-configuration/#mapping-of-configuration-options
- Extensions using `AutoConfigurationCustomizerProvider` must migrate to `DeclarativeConfigurationCustomizerProvider`; components such as span exporters must use `ComponentProvider`. https://opentelemetry.io/docs/zero-code/java/agent/declarative-configuration/#extension-api

## Extensions

- Load at runtime with `-Dotel.javaagent.extensions=/path/a.jar,/path/dir` (dev), or embed into the agent's `extensions/` directory (production). When repackaging, carry over the agent's `META-INF/MANIFEST.MF`. https://opentelemetry.io/docs/zero-code/java/agent/extensions/#using-extensions
- One shadow JAR: extensions cannot load dependencies from separate JARs. Agent-provided APIs are `compileOnly`, others `implementation`. Register with `@AutoService`. https://opentelemetry.io/docs/zero-code/java/agent/extensions/#project-setup-and-dependencies
- To enrich built-in instrumentation, use `InstrumenterCustomizerProvider`. https://opentelemetry.io/docs/zero-code/java/agent/extensions/#extension-points-overview

## Spring Boot starter specifics

- Spring Boot 2.6+ and 3.1+. Import `opentelemetry-instrumentation-bom`; with Maven before `spring-boot-dependencies`. Do not mix Gradle `platform(...)` with `io.spring.dependency-management`. https://opentelemetry.io/docs/zero-code/java/spring-boot-starter/getting-started/#dependency-management
- On Spring Boot 3.5+ with declarative configuration the BOM is required (Spring's pin lacks `ComponentLoader`); without it startup fails with `NoClassDefFoundError: io/opentelemetry/common/ComponentLoader`. https://opentelemetry.io/docs/zero-code/java/spring-boot-starter/declarative-configuration/
- Service name: `otel.service.name`, then `otel.resource.attributes`, then `spring.application.name`, `build-info.properties`, manifest `Implementation-Title`, default `unknown_service:java`. https://opentelemetry.io/docs/zero-code/java/spring-boot-starter/sdk-configuration/#service-name
- Disable for tests: `otel.sdk.disabled: true`; with declarative config (`otel.file_format: '1.0'`, Spring syntax `${VAR:default}`) use `otel.disabled: true`. https://opentelemetry.io/docs/zero-code/java/spring-boot-starter/declarative-configuration/
- Drop actuator spans with an `AutoConfigurationCustomizerProvider` bean (not with declarative configuration) returning `RuleBasedRoutingSampler.builder(SpanKind.SERVER, fallback).drop(UrlAttributes.URL_PATH, "^/actuator").build()`. https://opentelemetry.io/docs/zero-code/java/spring-boot-starter/programmatic-configuration/
- A Zipkin exporter on the classpath without a bean gets an exporter bean added to a simple span processor in the active tracer provider. https://opentelemetry.io/docs/zero-code/java/spring-boot-starter/other-spring-autoconfig/
