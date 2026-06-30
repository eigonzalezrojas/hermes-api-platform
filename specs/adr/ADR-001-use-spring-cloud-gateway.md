# ADR-001 — Use Spring Cloud Gateway as the gateway foundation

## Status

Accepted

## Date

2024-01-01

## Context

The Hermes platform requires an HTTP API gateway capable of:

- Routing requests to multiple downstream services based on path predicates.
- Applying cross-cutting filters (authentication, rate limiting, correlation IDs, logging) without modifying downstream services.
- Handling high request throughput with low per-request overhead.
- Integrating cleanly with the existing Spring Boot 3 ecosystem (Security, Actuator, Micrometer, Resilience4j).
- Being testable with standard JVM testing tools (JUnit 5, WebTestClient, Mockito).

The following options were evaluated:

| Option | Description |
|---|---|
| **Spring Cloud Gateway (WebFlux)** | Reactive gateway built on Project Reactor and Netty; first-class Spring Boot integration |
| **Spring MVC Gateway** | Servlet-based gateway; simpler programming model but blocking I/O |
| **Kong** | Nginx-based API gateway with Lua plugin system; requires separate infrastructure |
| **Nginx + Lua** | High-performance proxy with custom scripting; no JVM, no Spring integration |
| **AWS API Gateway** | Fully managed; tight AWS coupling; no local development story |

## Decision

**Use Spring Cloud Gateway (WebFlux/Project Reactor)** as the foundation for `hermes-gateway`.

## Rationale

1. **Ecosystem fit**: Spring Cloud Gateway integrates directly with Spring Boot 3 auto-configuration. Spring Security WebFlux, Spring Boot Actuator, Micrometer, and Resilience4j all work out of the box without adapters or bridges.

2. **Reactive programming model**: the WebFlux/Project Reactor model aligns with the goal of high-concurrency, low-latency request handling. A reactive gateway can handle thousands of concurrent requests on a small thread pool without thread-per-request overhead.

3. **Extensibility via filters**: `GlobalFilter` and `GatewayFilterFactory` provide a clean, testable extension point for cross-cutting concerns. Filters compose via the `GatewayFilterChain` and return `Mono<Void>` — consistent with the reactive contract throughout the platform.

4. **Testability**: `MockServerWebExchange` and `WebTestClient` enable full unit and integration testing of filters and routes without starting a server or mocking HTTP. No external tooling required.

5. **Built-in rate limiter**: Spring Cloud Gateway ships with a Redis-backed `RequestRateLimiter` filter that implements token bucket atomically via a Lua script. This eliminates the need to implement rate limiting from scratch.

6. **Programming over configuration**: routes can be defined programmatically via `RouteLocatorBuilder` (enabling dependency injection and conditional registration) or declaratively via YAML — both approaches are first-class.

## Consequences

### Positive

- All gateway extension points use standard Spring and Reactor APIs — no custom DSL or external scripting language to learn.
- The reactive model provides natural backpressure and resource isolation without thread pool configuration.
- Spring Boot's DevTools, Actuator endpoints, and auto-configuration reduce boilerplate significantly.

### Negative

- **The entire gateway stack must be non-blocking**: any accidental `.block()` call, blocking I/O, or `Thread.sleep()` starves the Netty event loop and degrades all requests — not just the offending one. This is a correctness constraint that requires discipline across every filter.
- **Spring Data JPA is incompatible with WebFlux**: JPA is blocking by design. `hermes-gateway` cannot use JPA or any blocking persistence. Database access is architecturally restricted to `hermes-admin` (Spring MVC), which is a separate service. See [ADR-001 enforcement in CLAUDE.md].
- **Reactor learning curve**: `Mono`, `Flux`, and reactive operator chains are less intuitive than imperative code. All contributors must be fluent in Project Reactor basics before modifying filter code.

## Alternatives considered

### Spring MVC Gateway (blocking)

Rejected because blocking I/O does not scale to the target request volume without large thread pools. It also prevents use of reactive Spring Security and makes Resilience4j's reactive circuit breaker integration awkward.

### Kong

Rejected because it introduces a separate infrastructure component (PostgreSQL for Kong's own config store, separate Nginx process) with a Lua plugin system that is disconnected from the JVM ecosystem. Observability integration (Micrometer, OpenTelemetry) would require custom plugins.

### Nginx + Lua

Rejected for the same ecosystem reasons as Kong, plus: no Spring Security integration, no standard JVM testing, and Lua scripting is not aligned with the team's Java expertise.

### AWS API Gateway

Rejected because it tightly couples the platform to AWS, eliminates local development without LocalStack, and does not support the custom filter pipeline (Correlation ID, tenant resolution, circuit breaker) that is central to the platform's value.
