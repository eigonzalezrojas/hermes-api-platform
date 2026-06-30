# TASK-003 — Circuit breaker with fallback

## Status

Planned

## Parent story

`US-003-resilience-and-rate-limiting.md`

## Summary

Add a Resilience4j reactive circuit breaker to the gateway routes using Spring Cloud Gateway's `CircuitBreaker` filter. When the circuit is open (downstream is failing), the gateway forwards the request to a local `FallbackController` that returns a structured `503 Service Unavailable` response, giving clients a meaningful error instead of a timeout.

## Technical scope

This task includes:

- `spring-cloud-starter-circuitbreaker-reactor-resilience4j` dependency added to `hermes-gateway/pom.xml`.
- `CircuitBreaker` filter applied to the `echo-route-yaml` route in `application.yml`.
- `FallbackController.java` in `dev.eithel.hermes.gateway` — a `@RestController` at `/fallback`.
- Resilience4j circuit breaker instance configuration in `application.yml`.
- Unit test for `FallbackController` using `WebTestClient` (standalone slice, no full context).

## Out of scope

This task does not include:

- Retry filter (deferred — retry amplifies load on a failing service; needs careful per-route tuning).
- Bulkhead / thread pool isolation.
- Prometheus circuit breaker state metrics (v0.5.0).
- Per-circuit-breaker configuration for multiple downstream services (only `echo-cb` for the echo route in MVP).
- Custom exception mapping in the fallback (a static JSON body is sufficient).

## Implementation notes

- **Dependency**: `spring-cloud-starter-circuitbreaker-reactor-resilience4j`. This brings Resilience4j and its Spring Cloud reactive adapter. No additional Resilience4j core dependency is needed.
- **Gateway filter**: `CircuitBreaker` filter applied after `StripPrefix` and `RequestRateLimiter` on the route. Arguments:
  - `name`: circuit breaker instance name (matches the Resilience4j config key).
  - `fallbackUri`: `forward:/fallback` — routes to a local endpoint when the circuit is open.
- **Circuit breaker instance name**: `echo-cb`.
- **Resilience4j configuration** in `application.yml`:
  - `slidingWindowSize`: `5` (last 5 calls determine failure rate).
  - `failureRateThreshold`: `50` (50% failures → circuit opens).
  - `waitDurationInOpenState`: `10s` (waits 10 seconds before trying half-open).
  - `permittedNumberOfCallsInHalfOpenState`: `2`.
  - `automaticTransitionFromOpenToHalfOpenEnabled`: `true`.
- **FallbackController**: standard `@RestController`. Must return a `Mono<ResponseEntity<Map<String, Object>>>` with:
  - HTTP status: `503 Service Unavailable`.
  - Body: `{"status": 503, "error": "Service Unavailable", "message": "Downstream service is temporarily unavailable. Please try again later."}`.
- The fallback endpoint is internal — it should not be exposed as a public API route. The gateway predicate for routes must not match `/fallback`.

### application.yml circuit breaker additions

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: echo-route-yaml
          uri: ${hermes.downstream.echo-uri}
          predicates:
            - Path=/api/echo/**
          filters:
            - StripPrefix=1
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: ${hermes.ratelimit.replenish-rate:10}
                redis-rate-limiter.burstCapacity: ${hermes.ratelimit.burst-capacity:20}
                key-resolver: "#{@remoteAddrKeyResolver}"
            - name: CircuitBreaker
              args:
                name: echo-cb
                fallbackUri: forward:/fallback

resilience4j:
  circuitbreaker:
    instances:
      echo-cb:
        sliding-window-size: 5
        failure-rate-threshold: 50
        wait-duration-in-open-state: 10s
        permitted-number-of-calls-in-half-open-state: 2
        automatic-transition-from-open-to-half-open-enabled: true
```

### FallbackController reference

```java
package dev.eithel.hermes.gateway;

import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;
import reactor.core.publisher.Mono;

import java.util.Map;

@RestController
public class FallbackController {

    @GetMapping("/fallback")
    public Mono<ResponseEntity<Map<String, Object>>> fallback() {
        Map<String, Object> body = Map.of(
                "status", 503,
                "error", "Service Unavailable",
                "message", "Downstream service is temporarily unavailable. Please try again later."
        );
        return Mono.just(ResponseEntity.status(HttpStatus.SERVICE_UNAVAILABLE).body(body));
    }
}
```

### Unit test reference

```java
class FallbackControllerTest {

    private final WebTestClient webTestClient = WebTestClient
            .bindToController(new FallbackController())
            .build();

    @Test
    void fallback_returns503WithStructuredBody() {
        webTestClient.get().uri("/fallback")
                .exchange()
                .expectStatus().isEqualTo(HttpStatus.SERVICE_UNAVAILABLE)
                .expectBody()
                .jsonPath("$.status").isEqualTo(503)
                .jsonPath("$.error").isEqualTo("Service Unavailable")
                .jsonPath("$.message").isNotEmpty();
    }
}
```

## Expected files

```text
hermes-gateway/
├── pom.xml                                              ← add circuitbreaker-reactor-resilience4j
└── src/
    ├── main/
    │   ├── java/dev/eithel/hermes/gateway/
    │   │   └── FallbackController.java
    │   └── resources/
    │       └── application.yml                          ← add CircuitBreaker filter + resilience4j config
    └── test/java/dev/eithel/hermes/gateway/
        └── FallbackControllerTest.java
```

## Acceptance criteria

- [ ] `spring-cloud-starter-circuitbreaker-reactor-resilience4j` is declared in `hermes-gateway/pom.xml`.
- [ ] `FallbackController` exists at `GET /fallback` and returns HTTP 503 with a JSON body.
- [ ] `CircuitBreaker` filter is applied to `echo-route-yaml` with `fallbackUri: forward:/fallback`.
- [ ] Resilience4j `echo-cb` instance is configured with `slidingWindowSize=5` and `failureRateThreshold=50`.
- [ ] `FallbackControllerTest` passes with `.\mvnw.cmd test -pl hermes-gateway --also-make`.
- [ ] Consecutive 5xx responses from the downstream eventually open the circuit (verified in integration tests — TASK-004).

## Validation

Unit test:

```powershell
.\mvnw.cmd test -pl hermes-gateway --also-make -Dtest=FallbackControllerTest
```

Expected: `BUILD SUCCESS`, 1 test passing.

## Commit suggestion

```text
feat: add Resilience4j circuit breaker with structured 503 fallback
```

## Notes

- **Why `forward:/fallback` and not a full URL?** `forward:` instructs the gateway to handle the request internally via Spring's `DispatcherHandler` (WebFlux equivalent of `DispatcherServlet`). This avoids an outbound network call for the fallback and keeps the response path fully within the gateway process.
- **Filter ordering on the route matters**: `StripPrefix` → `RequestRateLimiter` → `CircuitBreaker`. The circuit breaker wraps the actual routing — if the rate limiter rejects the request (429), the circuit breaker never sees it. This is correct: rate limiting is a gate, circuit breaking is a downstream protection.
- **`FallbackController` is NOT a gateway route**: `GET /fallback` is a local WebFlux endpoint. It must not be matched by any `Path` predicate in `spring.cloud.gateway.routes`. If a predicate accidentally matches `/fallback`, the circuit breaker creates an infinite loop.
- **`Map.of()` preserves insertion order in Java 9+ only for up to 10 entries** and the JSON serialization order is not guaranteed. If ordered output is required in a future story, switch to `LinkedHashMap` or a dedicated DTO.
