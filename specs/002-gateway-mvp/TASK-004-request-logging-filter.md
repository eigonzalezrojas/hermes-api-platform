# TASK-004 — Request logging filter

## Status

Planned

## Parent story

`US-002-gateway-mvp.md`

## Summary

Implement a `RequestLoggingFilter` — a `GlobalFilter` that logs a structured line for every inbound request. The log line includes the HTTP method, request URI, and the correlation ID resolved by `CorrelationIdFilter`. On completion, the response status code is logged. This provides the minimum operational visibility for the gateway without touching the request or response body.

## Technical scope

This task includes:

- `RequestLoggingFilter.java` in `dev.eithel.hermes.gateway.filter`.
- The filter implements `GlobalFilter` and `Ordered`.
- Reads the correlation ID from `exchange.getAttributes()` (set by `CorrelationIdFilter`).
- Logs on entry: `[method] [uri] correlationId=[id]`.
- Logs on completion (using `.doFinally()`): `[method] [uri] status=[status] correlationId=[id]`.
- Unit test using `MockServerWebExchange`.

## Out of scope

This task does not include:

- Request or response body logging.
- SLF4J MDC propagation.
- Logging headers other than the correlation ID.
- Metrics recording (deferred to v0.5.0).
- Log format customization via configuration properties.

## Implementation notes

- **Class**: `dev.eithel.hermes.gateway.filter.RequestLoggingFilter`
- **Implements**: `GlobalFilter`, `Ordered`
- **Order**: `Ordered.HIGHEST_PRECEDENCE + 2` — runs immediately after `CorrelationIdFilter` (`+ 1`), which guarantees the correlation ID is already in `exchange.getAttributes()`.
- **Logger**: SLF4J `LoggerFactory.getLogger(RequestLoggingFilter.class)`. Use `log.info(...)` for normal requests.
- **Correlation ID access**: `exchange.getAttribute(HermesHeaders.X_CORRELATION_ID)` — the attribute is a `String` placed by `CorrelationIdFilter`.
- **Response status logging**: use `.doFinally(signal -> ...)` on the result of `chain.filter(exchange)`. The `ServerWebExchange`'s response status is readable after the chain completes without blocking.
- **Reactive contract**: never block. `.doFinally()` is a side-effect operator that runs on the scheduler of the upstream signal — it is synchronous and brief. Only use it for logging, never for I/O.

### Reference implementation

```java
package dev.eithel.hermes.gateway.filter;

import dev.eithel.hermes.common.HermesHeaders;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.cloud.gateway.filter.GlobalFilter;
import org.springframework.core.Ordered;
import org.springframework.http.HttpMethod;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

import java.net.URI;

@Component
public class RequestLoggingFilter implements GlobalFilter, Ordered {

    private static final Logger log = LoggerFactory.getLogger(RequestLoggingFilter.class);

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        HttpMethod method = exchange.getRequest().getMethod();
        URI uri = exchange.getRequest().getURI();
        String correlationId = exchange.getAttribute(HermesHeaders.X_CORRELATION_ID);

        log.info(">>> {} {} correlationId={}", method, uri, correlationId);

        return chain.filter(exchange)
                .doFinally(signal -> {
                    Integer statusCode = exchange.getResponse().getStatusCode() != null
                            ? exchange.getResponse().getStatusCode().value()
                            : -1;
                    log.info("<<< {} {} status={} correlationId={}", method, uri, statusCode, correlationId);
                });
    }

    @Override
    public int getOrder() {
        return Ordered.HIGHEST_PRECEDENCE + 2;
    }
}
```

### Unit test reference

```java
class RequestLoggingFilterTest {

    private final RequestLoggingFilter filter = new RequestLoggingFilter();

    @Test
    void logsRequestAndCompletes_withCorrelationId() {
        MockServerHttpRequest request = MockServerHttpRequest
                .get("http://localhost:8080/api/echo/ping")
                .build();
        ServerWebExchange exchange = MockServerWebExchange.from(request);
        exchange.getAttributes().put(HermesHeaders.X_CORRELATION_ID, "test-id-456");
        GatewayFilterChain chain = ex -> Mono.empty();

        StepVerifier.create(filter.filter(exchange, chain))
                .verifyComplete();
    }

    @Test
    void logsRequestAndCompletes_withoutCorrelationId() {
        MockServerHttpRequest request = MockServerHttpRequest
                .get("http://localhost:8080/api/echo/ping")
                .build();
        ServerWebExchange exchange = MockServerWebExchange.from(request);
        GatewayFilterChain chain = ex -> Mono.empty();

        StepVerifier.create(filter.filter(exchange, chain))
                .verifyComplete();
    }
}
```

## Expected files

```text
hermes-gateway/
└── src/
    ├── main/java/dev/eithel/hermes/gateway/filter/
    │   └── RequestLoggingFilter.java
    └── test/java/dev/eithel/hermes/gateway/filter/
        └── RequestLoggingFilterTest.java
```

## Acceptance criteria

- [ ] `RequestLoggingFilter` implements `GlobalFilter` and `Ordered`.
- [ ] Filter order is `Ordered.HIGHEST_PRECEDENCE + 2`.
- [ ] An entry log line is emitted for every request containing method, URI, and correlation ID.
- [ ] A completion log line is emitted after the chain resolves containing method, URI, status code, and correlation ID.
- [ ] The filter reads the correlation ID from `exchange.getAttributes()`, not from request headers.
- [ ] No `.block()` calls anywhere in the filter.
- [ ] Both unit test cases pass.
- [ ] `.\mvnw.cmd test -pl hermes-gateway --also-make` succeeds.

## Validation

Unit tests only:

```powershell
.\mvnw.cmd test -pl hermes-gateway --also-make -Dtest=RequestLoggingFilterTest
```

Expected: `BUILD SUCCESS`, 2 tests passing.

Full milestone validation:

```powershell
.\mvnw.cmd clean verify
```

Expected: `BUILD SUCCESS`.

## Commit suggestion

```text
feat: add RequestLoggingFilter to log method, URI, and correlation ID per request
```

## Notes

- **Why read from `exchange.getAttributes()` and not from request headers?** The `CorrelationIdFilter` mutates the `ServerWebExchange` and creates a new request with the correlation ID in its headers — but `RequestLoggingFilter` receives the *original* exchange reference at the start of the chain, not the mutated one. Reading from `exchange.getAttributes()` avoids this ordering dependency because attributes are shared across the mutable exchange.
- **`doFinally` vs `then`**: `doFinally` fires on completion, error, and cancellation — covering all terminal signals. `then(Mono.fromRunnable(...))` only fires on success. `doFinally` is the correct choice for logging response status.
- **Null-safe status code**: `exchange.getResponse().getStatusCode()` may be null if the chain short-circuits (e.g., a security filter rejects the request before the route filter sets the status). The `-1` fallback is intentional and makes the null case visible in logs.
- **Future enhancement**: when MDC is enabled (observability story), replace the inline `correlationId=` interpolation with an MDC key that the log appender writes automatically.
