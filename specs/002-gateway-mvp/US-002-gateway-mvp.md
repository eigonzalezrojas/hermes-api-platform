# US-002 — Gateway MVP

## Status

Planned

## Summary

As a backend engineer,
I want the gateway to route HTTP requests to downstream services, attach a correlation ID to every request, and log each transaction,
so that I can trace requests end-to-end across the system and diagnose issues without access to downstream service logs.

## Business / Engineering value

This story delivers the first observable, functional behavior of the gateway. Prior to this, the gateway starts but does nothing useful — requests pass through with no routing, no traceability, and no visibility.

By the end of this story:

- The gateway routes incoming traffic to at least one downstream service using both YAML and programmatic configuration, demonstrating both approaches for the portfolio.
- Every request carries a `X-Correlation-ID` header that flows through to the downstream service and appears in the response — the minimum requirement for distributed tracing.
- Every request produces a structured log line that includes the HTTP method, URI, and correlation ID — the minimum requirement for operational visibility.

These are the three baseline capabilities that all future gateway features (rate limiting, circuit breaking, auth) depend on.

## Scope

This story includes:

- A static route defined in `application.yml` (YAML-based route configuration).
- A programmatic route defined via a `RouteLocator` bean in a `@Configuration` class.
- A `CorrelationIdFilter` (`GlobalFilter`) that generates or propagates `X-Correlation-ID`.
- A `RequestLoggingFilter` (`GlobalFilter`) that logs method, URI, and correlation ID.
- Unit tests for both filters using `MockServerWebExchange`.
- An integration test (WebTestClient + WireMock) that validates end-to-end routing and header propagation.

## Out of scope

This story does not include:

- Redis, rate limiting, or circuit breakers.
- OAuth2, JWT, or any authentication filter.
- Multi-tenancy or tenant resolution.
- Dynamic route management (CRUD via Admin API).
- Prometheus metrics or OpenTelemetry tracing spans.
- Request/response body logging (deferred — memory and latency implications).

## Acceptance criteria

- [ ] A request to the gateway matching the configured route is forwarded to the downstream stub.
- [ ] If `X-Correlation-ID` is absent from the incoming request, the filter generates a UUID and injects it.
- [ ] If `X-Correlation-ID` is present in the incoming request, the filter uses the supplied value unchanged.
- [ ] The `X-Correlation-ID` header appears in the proxied downstream request (verified via WireMock).
- [ ] The `X-Correlation-ID` header appears in the gateway response.
- [ ] Each request produces a log line containing: HTTP method, URI, and correlation ID.
- [ ] Both YAML and programmatic routes are declared and reachable.
- [ ] `.\mvnw.cmd clean verify` passes from the repository root.

## Technical notes

- **Reactive contract**: both filters must implement `GlobalFilter` and `Ordered`. Never call `.block()`. Use `exchange.mutate()` for header manipulation — `ServerHttpRequest` and `ServerHttpResponse` are immutable.
- **Filter ordering**: `CorrelationIdFilter` must run before `RequestLoggingFilter` so the log line can include the resolved correlation ID.
- **Correlation ID header name**: `X-Correlation-ID`. Define as a constant in `hermes-common`.
- **Downstream stub**: WireMock server started by Testcontainers in integration tests. YAML/programmatic routes point to a configurable URI (`${hermes.downstream.echo-uri}`) overridden in test application properties.
- **Log format**: structured fields using SLF4J MDC is deferred. For now, a single log line with method, URI, and correlation ID is sufficient.
- **Route predicate**: `Path=/api/echo/**` with `StripPrefix=1` filter (removes `/api` segment before forwarding).

## Related tasks

- `TASK-001-static-route-configuration.md`
- `TASK-002-programmatic-route-configuration.md`
- `TASK-003-correlation-id-filter.md`
- `TASK-004-request-logging-filter.md`

## Dependencies

- `US-001-project-bootstrap.md` — must be `Done` before this story begins.

## Validation

Start gateway with local profile:

```powershell
.\mvnw.cmd -pl hermes-gateway spring-boot:run -Dspring-boot.run.profiles=local
```

Full build and tests:

```powershell
.\mvnw.cmd clean verify
```

Manual smoke test (requires a running downstream stub or `httpbin`):

```bash
curl -s -v http://localhost:8080/api/echo/ping
```

Expected: request forwarded, `X-Correlation-ID` present in response headers, log line printed to console.

## Notes

- Both YAML and programmatic route configuration are demonstrated intentionally for portfolio breadth. In production, one approach would be chosen — typically programmatic for testability.
- Request body logging is explicitly excluded: reading the body in a reactive pipeline is destructive (the stream can only be consumed once) and requires special handling. Defer to a dedicated observability story.
- The `X-Correlation-ID` constant will live in `hermes-common` to avoid string literals across modules.
