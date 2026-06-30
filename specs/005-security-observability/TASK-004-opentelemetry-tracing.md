# TASK-004 — OpenTelemetry distributed tracing

## Status

Planned

## Parent story

`US-005-security-and-observability.md`

## Summary

Add distributed tracing to `hermes-gateway` using Micrometer Tracing with the OpenTelemetry bridge. Every inbound request gets a trace context (W3C `traceparent` header), the trace ID is correlated with the existing `X-Correlation-ID` header, and spans are exported to a local OTLP collector (or logged to console) during development. In production, spans flow to AWS X-Ray via ADOT.

## Technical scope

This task includes:

- `micrometer-tracing-bridge-otel` and `opentelemetry-exporter-otlp` dependencies in `hermes-gateway/pom.xml`.
- `application.yml` tracing configuration: sampling rate, OTLP endpoint, propagation format.
- `application-local.yml`: configure the `logging` span exporter (prints spans to console — no collector required locally).
- `CorrelationIdFilter` updated (from US-002 `TASK-003`) to use the active trace ID as the `X-Correlation-ID` value when a trace context is present, falling back to UUID when absent.
- `TracingConfig.java` in `dev.eithel.hermes.gateway.config` — custom `SpanCustomizer` bean if needed for adding gateway-level attributes to every span.
- Unit test verifying the trace ID is used as the correlation ID when a span is active.

## Out of scope

This task does not include:

- AWS X-Ray integration (uses the X-Ray trace ID format and requires the ADOT agent — deferred to v1.0.0).
- Zipkin exporter (OTLP is the standard; Zipkin is a legacy option).
- Tracing for `hermes-admin` (admin has low traffic; standard Spring Boot Actuator provides enough for the MVP).
- Baggage propagation (tenant ID in baggage — deferred to the multi-tenancy story).
- Adding a Jaeger or Tempo container to `docker-compose.yml` (the logging exporter is sufficient for local development).

## Implementation notes

- **Dependencies**:
  - `io.micrometer:micrometer-tracing-bridge-otel` — bridges Micrometer's `Tracer` to OpenTelemetry's SDK.
  - `io.opentelemetry:opentelemetry-exporter-otlp` — exports spans via OTLP/gRPC.
  - `io.opentelemetry:opentelemetry-exporter-logging` — logs spans to console (local dev only).
  - Versions are managed by Spring Boot's BOM — no explicit version needed.
- **Propagation format**: W3C `traceparent` (B3 is legacy — use W3C for new projects). Configure via `management.tracing.propagation.type=w3c`.
- **Sampling rate**: `management.tracing.sampling.probability=1.0` locally (100% sampling for dev). Set to `0.1` (10%) or lower in production to reduce span volume.
- **OTLP endpoint** (production): `management.otlp.tracing.endpoint=http://otel-collector:4317`. For local, this is unset and the logging exporter handles output instead.
- **Local exporter**: configure `spring.main.banner-mode=off` and add `logging` as the exporter type. This writes span summaries to the application log — visible in the console without a collector.
- **`CorrelationIdFilter` update**: inject `io.micrometer.tracing.Tracer` into `CorrelationIdFilter`. In `filter()`, call `tracer.currentSpan()` — if non-null and the span context has a valid trace ID, use `span.context().traceId()` as the `X-Correlation-ID` value. Fall back to `UUID.randomUUID()` if no active span (e.g., when tracing is disabled or sampling rate excludes this request).
- **Automatic HTTP propagation**: Micrometer Tracing auto-configures `WebFluxObservationFilter` which creates a root span for every HTTP request and propagates trace context in outbound calls made via instrumented `WebClient` instances. No manual span creation is needed for standard request handling.
- **`WebClient` instrumentation**: Spring Cloud Gateway's routing uses an internal `WebClient` to call downstream services. Micrometer Tracing auto-instruments it when `spring.cloud.gateway.httpclient.wiretap=false` (the default). The `traceparent` header is automatically added to outbound requests.

### pom.xml additions

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-exporter-otlp</artifactId>
</dependency>
<!-- Local development only — omit or scope=test in production builds -->
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-exporter-logging</artifactId>
</dependency>
```

### application.yml tracing block

```yaml
management:
  tracing:
    sampling:
      probability: ${hermes.tracing.sampling-probability:1.0}
    propagation:
      type: w3c
  otlp:
    tracing:
      endpoint: ${hermes.tracing.otlp-endpoint:}
```

### application-local.yml tracing overrides

```yaml
hermes:
  tracing:
    sampling-probability: 1.0
    otlp-endpoint: ""              # empty → falls back to logging exporter

logging:
  level:
    io.opentelemetry.exporter.logging: DEBUG   # prints spans to console
```

### Updated CorrelationIdFilter (with tracing)

```java
@Component
public class CorrelationIdFilter implements GlobalFilter, Ordered {

    private final Tracer tracer;

    public CorrelationIdFilter(Tracer tracer) {
        this.tracer = tracer;
    }

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String correlationId = resolveCorrelationId(exchange);

        exchange.getAttributes().put(HermesHeaders.X_CORRELATION_ID, correlationId);
        exchange.getResponse().getHeaders().set(HermesHeaders.X_CORRELATION_ID, correlationId);

        ServerWebExchange mutatedExchange = exchange.mutate()
                .request(r -> r.header(HermesHeaders.X_CORRELATION_ID, correlationId))
                .build();

        return chain.filter(mutatedExchange);
    }

    private String resolveCorrelationId(ServerWebExchange exchange) {
        String incoming = exchange.getRequest().getHeaders().getFirst(HermesHeaders.X_CORRELATION_ID);
        if (incoming != null && !incoming.isBlank()) {
            return incoming;
        }
        Span currentSpan = tracer.currentSpan();
        if (currentSpan != null && !currentSpan.context().traceId().equals("0000000000000000")) {
            return currentSpan.context().traceId();
        }
        return UUID.randomUUID().toString();
    }

    @Override
    public int getOrder() {
        return Ordered.HIGHEST_PRECEDENCE + 1;
    }
}
```

### Unit test reference

```java
class CorrelationIdFilterTracingTest {

    @Test
    void usesTraceId_whenActiveSpanPresent() {
        Tracer mockTracer = mock(Tracer.class);
        Span mockSpan = mock(Span.class);
        TraceContext mockContext = mock(TraceContext.class);

        when(mockTracer.currentSpan()).thenReturn(mockSpan);
        when(mockSpan.context()).thenReturn(mockContext);
        when(mockContext.traceId()).thenReturn("abc123traceId0000");

        CorrelationIdFilter filter = new CorrelationIdFilter(mockTracer);
        MockServerHttpRequest request = MockServerHttpRequest.get("/api/echo/ping").build();
        ServerWebExchange exchange = MockServerWebExchange.from(request);
        GatewayFilterChain chain = ex -> Mono.empty();

        StepVerifier.create(filter.filter(exchange, chain)).verifyComplete();

        String correlationId = exchange.getResponse().getHeaders()
                .getFirst(HermesHeaders.X_CORRELATION_ID);
        assertThat(correlationId).isEqualTo("abc123traceId0000");
    }

    @Test
    void fallsBackToUuid_whenNoActiveSpan() {
        Tracer mockTracer = mock(Tracer.class);
        when(mockTracer.currentSpan()).thenReturn(null);

        CorrelationIdFilter filter = new CorrelationIdFilter(mockTracer);
        MockServerHttpRequest request = MockServerHttpRequest.get("/api/echo/ping").build();
        ServerWebExchange exchange = MockServerWebExchange.from(request);
        GatewayFilterChain chain = ex -> Mono.empty();

        StepVerifier.create(filter.filter(exchange, chain)).verifyComplete();

        String correlationId = exchange.getResponse().getHeaders()
                .getFirst(HermesHeaders.X_CORRELATION_ID);
        assertThat(correlationId).isNotNull().isNotBlank();
    }
}
```

## Expected files

```text
hermes-gateway/
├── pom.xml                                            ← add tracing dependencies
└── src/
    ├── main/
    │   ├── java/dev/eithel/hermes/gateway/filter/
    │   │   └── CorrelationIdFilter.java               ← updated with Tracer injection
    │   └── resources/
    │       ├── application.yml                        ← add management.tracing block
    │       └── application-local.yml                  ← add tracing overrides
    └── test/java/dev/eithel/hermes/gateway/filter/
        └── CorrelationIdFilterTracingTest.java
```

## Acceptance criteria

- [ ] `micrometer-tracing-bridge-otel` and `opentelemetry-exporter-otlp` are declared in `hermes-gateway/pom.xml`.
- [ ] Each gateway request produces a log entry with a trace ID when the logging exporter is active.
- [ ] The `X-Correlation-ID` response header equals the trace ID for sampled requests.
- [ ] The `traceparent` header is forwarded to the downstream service (visible in WireMock logs or integration test assertions).
- [ ] When no active span is present, `X-Correlation-ID` falls back to a UUID (existing behavior preserved).
- [ ] `CorrelationIdFilterTracingTest` passes with `.\mvnw.cmd test -pl hermes-gateway --also-make`.
- [ ] `.\mvnw.cmd clean verify` passes.

## Validation

```powershell
.\mvnw.cmd test -pl hermes-gateway --also-make -Dtest=CorrelationIdFilterTracingTest
```

Full build:

```powershell
.\mvnw.cmd clean verify
```

Local trace output (requires running gateway with local profile):

```bash
# Send a request and observe the console for span output
curl -s http://localhost:8080/actuator/health
# Look for lines like: [trace_id=... span_id=...]
```

## Commit suggestion

```text
feat: add OpenTelemetry tracing with W3C traceparent propagation and trace-correlated X-Correlation-ID
```

## Notes

- **Micrometer Tracing vs OpenTelemetry SDK directly**: Micrometer Tracing is a facade over tracing implementations (OTel, Brave/Zipkin). Using it keeps the application code decoupled from the underlying exporter — switching from OTLP to X-Ray (via ADOT) in v1.0.0 requires only a dependency swap, not code changes.
- **`traceparent` header format** (W3C): `00-{traceId}-{spanId}-{flags}`. The `traceId` is 16 bytes hex (32 characters). Micrometer Tracing formats the trace ID in W3C format when propagating outbound.
- **Reactor context and trace propagation**: in WebFlux, the trace context is propagated via the Reactor `Context` (not `ThreadLocal`). Micrometer Tracing's WebFlux integration handles this automatically — no manual context propagation is needed in application code.
- **Sampling rate in production**: `probability=1.0` means every request produces a trace. At scale (e.g., 1000 req/s), this generates 86 million spans per day. Set to `0.05`–`0.1` in production and rely on tail-based sampling at the collector level for capturing error and slow traces at full resolution.
- **The `traceparent` trace ID is 32 hex chars; `X-Correlation-ID` as used in the codebase has no format constraint** — using the trace ID directly satisfies both. The existing filter behavior (use supplied `X-Correlation-ID` if present) ensures interoperability with clients that generate their own correlation IDs.
