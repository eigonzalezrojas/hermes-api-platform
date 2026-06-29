# US-003 — Resilience and rate limiting

## Status

Planned

## Summary

As a platform engineer,
I want the gateway to enforce per-client request rate limits backed by Redis and to open a circuit breaker when a downstream service is failing,
so that the platform remains stable under traffic spikes, protects downstream services from overload, and degrades gracefully with a meaningful fallback response instead of cascading errors.

## Business / Engineering value

Without rate limiting, a single abusive or misbehaving client can saturate all downstream capacity. Without a circuit breaker, a failing downstream service causes slow, connection-exhausting timeouts that degrade the entire gateway under load.

This story delivers the two most critical protective mechanisms for an API gateway:

- **Rate limiting**: Redis-backed token bucket shared across all gateway instances, enforced before a request reaches any downstream service.
- **Circuit breaker**: Resilience4j reactive circuit breaker that trips on sustained downstream failures, returns a fast fallback response, and automatically probes for recovery.

Both mechanisms are verified with real infrastructure: Redis runs in a Testcontainers container during integration tests, and WireMock simulates downstream failure scenarios. This also introduces the `hermes-test` module as the home for all integration and contract tests.

## Scope

This story includes:

- `docker-compose.yml` at the repository root with a Redis service (port `6379`).
- Spring Cloud Gateway `RequestRateLimiter` filter backed by Redis, with a configurable `KeyResolver`.
- A `FallbackController` in `hermes-gateway` that returns a structured 503 response.
- Resilience4j reactive circuit breaker filter wired to the fallback controller.
- The `hermes-test` Maven module with Testcontainers and WireMock dependencies.
- Integration tests verifying: rate limit enforcement (HTTP 429), circuit breaker fallback (HTTP 503), and normal proxying (HTTP 200).

## Out of scope

This story does not include:

- PostgreSQL or any relational database (added in v0.4.0 with `hermes-admin`).
- OAuth2, JWT, or authentication.
- Retry filter configuration (deferred — retry must be tuned per downstream SLA).
- Bulkhead / thread pool isolation.
- Prometheus metrics for circuit breaker state (v0.5.0).
- Multi-tenancy rate limiting (separate limit per tenant/API key).

## Acceptance criteria

- [ ] `docker compose up -d` starts a Redis container on port `6379` with no errors.
- [ ] A client sending requests above the configured rate limit receives HTTP `429 Too Many Requests`.
- [ ] The `X-RateLimit-Remaining` and `X-RateLimit-Replenish-Rate` response headers are present on rate-limited responses.
- [ ] When the downstream service returns 5xx errors above the failure threshold, the circuit breaker opens and the gateway returns HTTP `503` with a JSON fallback body.
- [ ] When the downstream service recovers, the circuit breaker transitions from open → half-open → closed and normal proxying resumes.
- [ ] All integration tests in `hermes-test` pass with `.\mvnw.cmd clean verify`.
- [ ] `.\mvnw.cmd clean verify` passes from the repository root (no Docker daemon required for unit tests; Testcontainers requires Docker for integration tests).

## Technical notes

- **Rate limiter**: Spring Cloud Gateway's built-in `RequestRateLimiter` filter uses a Redis Lua script to implement token bucket atomically. All gateway instances share the same Redis, so limits are globally enforced.
- **KeyResolver**: for the MVP, resolve by client IP (`RemoteAddrKeyResolver` or a custom bean returning `exchange.getRequest().getRemoteAddress().getHostString()`). Future stories will resolve by API key or tenant ID.
- **Redis reactive**: requires `spring-boot-starter-data-redis-reactive` in `hermes-gateway`. Lettuce is the default reactive client — no additional driver dependency needed.
- **Circuit breaker**: `spring-cloud-starter-circuitbreaker-reactor-resilience4j`. The circuit breaker is applied as a gateway filter (`CircuitBreaker`) with a `fallbackUri: forward:/fallback`.
- **Fallback controller**: a `@RestController` in `hermes-gateway` at path `/fallback`. It must return a `Mono<ResponseEntity<?>>` with HTTP 503 and a JSON body: `{"status": 503, "error": "Service Unavailable", "message": "Downstream service is temporarily unavailable."}`.
- **hermes-test module**: plain Maven module, `<packaging>jar</packaging>`, depends on `hermes-gateway`. Uses `@SpringBootTest` with `RANDOM_PORT` and `WebTestClient`. Testcontainers manages the Redis lifecycle; WireMock stubs the downstream.
- **Testcontainers**: use `@Testcontainers` + `@Container` annotations with `GenericContainer("redis:7-alpine")`. Override `spring.data.redis.host` and `spring.data.redis.port` via `@DynamicPropertySource`.

## Related tasks

- `TASK-001-docker-compose-redis.md`
- `TASK-002-token-bucket-rate-limiting.md`
- `TASK-003-circuit-breaker-fallback.md`
- `TASK-004-integration-tests.md`

## Dependencies

- `US-001-project-bootstrap.md` — must be `Done`.
- `US-002-gateway-mvp.md` — must be `Done` (routes and filters from v0.2.0 are the target of rate limiting and circuit breaking).

## Validation

Local infrastructure:

```bash
docker compose up -d
docker compose ps   # redis should be Up
```

Full build and integration tests (requires Docker):

```powershell
.\mvnw.cmd clean verify
```

Manual rate limit smoke test (adjust `replenishRate` to a low value locally):

```bash
for i in {1..15}; do curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8080/api/echo/ping; done
```

Expected: first N requests return `200`, then `429`.

## Notes

- Redis is introduced here only for rate limiting. In v0.4.0, PostgreSQL is added for `hermes-admin`. The `docker-compose.yml` will be extended then — `TASK-001` should design it for extensibility.
- The `hermes-test` module depends on `hermes-gateway` but is not itself a Spring Boot application — it hosts tests only. It must not be packaged as a fat JAR.
- Testcontainers requires a Docker daemon at test time. CI must run on a Docker-capable agent (GitHub Actions `ubuntu-latest` satisfies this).
