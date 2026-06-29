# TASK-003 — Correlation ID filter

## Status

Planned

## Parent story

`US-002-gateway-mvp.md`

## Summary

Implement a `CorrelationIdFilter` — a `GlobalFilter` that ensures every request passing through the gateway carries an `X-Correlation-ID` header. If the incoming request already has the header, the value is preserved. If absent, a UUID is generated and injected. The correlation ID is added to both the proxied downstream request and the gateway response, enabling end-to-end request traceability.

## Technical scope

This task includes:

- `CorrelationIdFilter.java` in `dev.eithel.hermes.gateway.filter`.
- `HermesHeaders.java` constant class in `dev.eithel.hermes.common` (defines `X_CORRELATION_ID = "X-Correlation-ID"`).
- The filter implements `GlobalFilter` and `Ordered`.
- The correlation ID is stored in the reactive `ServerWebExchange` attributes for use by downstream filters (e.g., `RequestLoggingFilter`).
- Unit test using `MockServerWebExchange`.
- Integration test verifying the header appears in the WireMock-captured downstream request and in the gateway response.

## Out of scope

This task does not include:

- SLF4J MDC propagation (deferred — MDC is thread-local and requires a dedicated MDC-adapter for reactive pipelines).
- Span or trace ID injection for OpenTelemetry (v0.5.0).
- Writing the correlation ID to the response body.

## Implementation notes

- **Class**: `dev.eithel.hermes.gateway.filter.CorrelationIdFilter`
- **Implements**: `GlobalFilter`, `Ordered`
- **Order**: `Ordered.HIGHEST_PRECEDENCE + 1` — runs first among application filters; leaves room for a future security filter at `HIGHEST_PRECEDENCE`.
- **Header name constant**: `dev.eithel.hermes.common.HermesHeaders.X_CORRELATION_ID`
- **Header value**: `UUID.randomUUID().toString()` when absent.
- **Exchange attribute key**: `HermesHeaders.X_CORRELATION_ID` — stored via `exchange.getAttributes().put(...)` for access by `RequestLoggingFilter`.
- **Immutability**: `ServerHttpRequest` is immutable. Use `exchange.mutate().request(r -> r.header(...)).build()` to create a new exchange with the added header.
- **Response header**: set via `exchange.getResponse().getHeaders().add(...)` before calling `chain.filter(mutatedExchange)`.

### Reference implementation

```java
package dev.eithel.hermes.gateway.filter;

import dev.eithel.hermes.common.HermesHeaders;
import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.cloud.gateway.filter.GlobalFilter;
import org.springframework.core.Ordered;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

import java.util.UUID;

@Component
public class CorrelationIdFilter implements GlobalFilter, Ordered {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String correlationId = exchange.getRequest().getHeaders()
                .getFirst(HermesHeaders.X_CORRELATION_ID);

        if (correlationId == null || correlationId.isBlank()) {
            correlationId = UUID.randomUUID().toString();
        }

        String resolvedId = correlationId;

        exchange.getAttributes().put(HermesHeaders.X_CORRELATION_ID, resolvedId);
        exchange.getResponse().getHeaders().add(HermesHeaders.X_CORRELATION_ID, resolvedId);

        ServerWebExchange mutatedExchange = exchange.mutate()
                .request(r -> r.header(HermesHeaders.X_CORRELATION_ID, resolvedId))
                .build();

        return chain.filter(mutatedExchange);
    }

    @Override
    public int getOrder() {
        return Ordered.HIGHEST_PRECEDENCE + 1;
    }
}
```

### HermesHeaders constant class

```java
package dev.eithel.hermes.common;

public final class HermesHeaders {

    public static final String X_CORRELATION_ID = "X-Correlation-ID";

    private HermesHeaders() {}
}
```

### Unit test reference

```java
class CorrelationIdFilterTest {

    private final CorrelationIdFilter filter = new CorrelationIdFilter();

    @Test
    void generatesCorrelationId_whenAbsent() {
        MockServerHttpRequest request = MockServerHttpRequest.get("/api/echo/ping").build();
        ServerWebExchange exchange = MockServerWebExchange.from(request);
        GatewayFilterChain chain = ex -> Mono.empty();

        StepVerifier.create(filter.filter(exchange, chain))
                .verifyComplete();

        String correlationId = exchange.getResponse().getHeaders()
                .getFirst(HermesHeaders.X_CORRELATION_ID);
        assertThat(correlationId).isNotNull().isNotBlank();
    }

    @Test
    void preservesCorrelationId_whenPresent() {
        String existing = "test-correlation-id-123";
        MockServerHttpRequest request = MockServerHttpRequest.get("/api/echo/ping")
                .header(HermesHeaders.X_CORRELATION_ID, existing)
                .build();
        ServerWebExchange exchange = MockServerWebExchange.from(request);
        GatewayFilterChain chain = ex -> Mono.empty();

        StepVerifier.create(filter.filter(exchange, chain))
                .verifyComplete();

        String correlationId = exchange.getResponse().getHeaders()
                .getFirst(HermesHeaders.X_CORRELATION_ID);
        assertThat(correlationId).isEqualTo(existing);
    }
}
```

## Expected files

```text
hermes-common/
└── src/main/java/dev/eithel/hermes/common/
    └── HermesHeaders.java

hermes-gateway/
└── src/
    ├── main/java/dev/eithel/hermes/gateway/filter/
    │   └── CorrelationIdFilter.java
    └── test/java/dev/eithel/hermes/gateway/filter/
        └── CorrelationIdFilterTest.java
```

## Acceptance criteria

- [ ] `HermesHeaders.X_CORRELATION_ID` constant exists in `hermes-common`.
- [ ] `CorrelationIdFilter` implements `GlobalFilter` and `Ordered`.
- [ ] Filter order is `Ordered.HIGHEST_PRECEDENCE + 1`.
- [ ] Requests without `X-Correlation-ID` receive a generated UUID.
- [ ] Requests with `X-Correlation-ID` preserve the supplied value.
- [ ] The correlation ID is present in `exchange.getAttributes()` after the filter runs.
- [ ] The correlation ID appears in the gateway response headers.
- [ ] Both unit test cases pass.
- [ ] `.\mvnw.cmd test -pl hermes-gateway --also-make` succeeds.

## Validation

Unit tests only:

```powershell
.\mvnw.cmd test -pl hermes-gateway --also-make -Dtest=CorrelationIdFilterTest
```

Expected: `BUILD SUCCESS`, 2 tests passing.

## Commit suggestion

```text
feat: add CorrelationIdFilter to inject X-Correlation-ID on every request
```

## Notes

- **Why `HIGHEST_PRECEDENCE + 1` and not `HIGHEST_PRECEDENCE`?** Spring Cloud Gateway's own internal filters (NettyRoutingFilter, ForwardRoutingFilter) use `Integer.MAX_VALUE` order, so they run last. `HIGHEST_PRECEDENCE` is `Integer.MIN_VALUE`. Adding `+ 1` leaves `HIGHEST_PRECEDENCE` open for a future security/auth filter that must run before correlation ID.
- **Thread-safety**: `UUID.randomUUID()` is thread-safe. No synchronization needed.
- **Response header timing**: `exchange.getResponse().getHeaders().add(...)` must be called before `chain.filter(...)` — once the response is committed, headers can no longer be mutated.
- **Do not use `MDC`** directly in reactive pipelines. SLF4J MDC is thread-local; in a reactive thread pool, a single request may run on multiple threads, making MDC unreliable without a dedicated adapter (e.g., `reactor-extra` `MDCContext`). Defer to the observability story.
