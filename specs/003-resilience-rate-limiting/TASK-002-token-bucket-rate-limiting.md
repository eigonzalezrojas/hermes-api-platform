# TASK-002 — Token bucket rate limiting (Redis)

## Status

Planned

## Parent story

`US-003-resilience-and-rate-limiting.md`

## Summary

Configure Spring Cloud Gateway's built-in `RequestRateLimiter` filter on the gateway routes, backed by Redis using the token bucket algorithm. The rate limiter is global (shared across all gateway instances via Redis), enforced per client IP in the MVP, and configurable via `application.yml` properties without redeployment.

## Technical scope

This task includes:

- `spring-boot-starter-data-redis-reactive` dependency added to `hermes-gateway/pom.xml`.
- `RateLimiterConfig.java` in `dev.eithel.hermes.gateway.config` defining a `KeyResolver` bean.
- `RequestRateLimiter` filter applied to existing routes in `application.yml`.
- Rate limiter properties: `replenishRate`, `burstCapacity`, and `requestedTokens` in `application.yml`.
- `application-local.yml` overrides with a low replenish rate for local testing.
- Unit test for `RateLimiterConfig` verifying the `KeyResolver` returns a non-empty key `Mono`.

## Out of scope

This task does not include:

- Per-tenant or per-API-key rate limiting (resolved key will be enhanced in a future story).
- Custom rate limiter response body (the default 429 response from Spring Cloud Gateway is sufficient).
- Rate limit headers customization beyond the defaults (`X-RateLimit-Remaining`, `X-RateLimit-Replenish-Rate`).
- Redis cluster configuration.

## Implementation notes

- **Dependency**: `spring-boot-starter-data-redis-reactive` — brings Lettuce reactive client. Spring Cloud Gateway uses Lettuce internally for the rate limiter Lua script.
- **KeyResolver**: resolves the rate limit bucket key per request. For the MVP, use the client's remote IP address. Define as a `@Bean` named `remoteAddrKeyResolver` (or set `spring.cloud.gateway.default-filters` to use it by name).
- **Bean name matters**: Spring Cloud Gateway's `RequestRateLimiter` filter references the `KeyResolver` by bean name via the `key-resolver` filter argument (`#{@remoteAddrKeyResolver}`).
- **Rate limiter properties** (defined in `application.yml`, overridable per-route):
  - `replenishRate`: tokens added per second (sustained rate). E.g., `10`.
  - `burstCapacity`: max tokens in the bucket (handles burst). E.g., `20`.
  - `requestedTokens`: tokens consumed per request. Default `1`.
- **Local profile override**: set `replenishRate: 2` and `burstCapacity: 5` in `application-local.yml` to make rate limiting observable during local testing without sending many requests.
- **Redis connection**: Spring auto-configures Lettuce using `spring.data.redis.host` and `spring.data.redis.port` from `application-local.yml`.

### RateLimiterConfig reference

```java
package dev.eithel.hermes.gateway.config;

import org.springframework.cloud.gateway.filter.ratelimit.KeyResolver;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import reactor.core.publisher.Mono;

@Configuration
public class RateLimiterConfig {

    @Bean
    public KeyResolver remoteAddrKeyResolver() {
        return exchange -> Mono.justOrEmpty(
                exchange.getRequest().getRemoteAddress()
        ).map(addr -> addr.getAddress().getHostAddress());
    }
}
```

### application.yml route update (echo-route-yaml)

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
                redis-rate-limiter.requestedTokens: 1
                key-resolver: "#{@remoteAddrKeyResolver}"
```

### application-local.yml additions

```yaml
hermes:
  ratelimit:
    replenish-rate: 2
    burst-capacity: 5
```

### Unit test reference

```java
class RateLimiterConfigTest {

    @Test
    void remoteAddrKeyResolver_returnsNonEmptyKey() {
        RateLimiterConfig config = new RateLimiterConfig();
        KeyResolver resolver = config.remoteAddrKeyResolver();

        MockServerHttpRequest request = MockServerHttpRequest
                .get("/api/echo/ping")
                .remoteAddress(new InetSocketAddress("192.168.1.1", 12345))
                .build();
        ServerWebExchange exchange = MockServerWebExchange.from(request);

        StepVerifier.create(resolver.resolve(exchange))
                .expectNext("192.168.1.1")
                .verifyComplete();
    }

    @Test
    void remoteAddrKeyResolver_emitsEmpty_whenNoRemoteAddress() {
        RateLimiterConfig config = new RateLimiterConfig();
        KeyResolver resolver = config.remoteAddrKeyResolver();

        MockServerHttpRequest request = MockServerHttpRequest.get("/api/echo/ping").build();
        ServerWebExchange exchange = MockServerWebExchange.from(request);

        StepVerifier.create(resolver.resolve(exchange))
                .verifyComplete();
    }
}
```

## Expected files

```text
hermes-gateway/
├── pom.xml                                              ← add redis-reactive dependency
└── src/
    ├── main/
    │   ├── java/dev/eithel/hermes/gateway/config/
    │   │   └── RateLimiterConfig.java
    │   └── resources/
    │       ├── application.yml                          ← add RequestRateLimiter filter to routes
    │       └── application-local.yml                   ← add hermes.ratelimit overrides
    └── test/java/dev/eithel/hermes/gateway/config/
        └── RateLimiterConfigTest.java
```

## Acceptance criteria

- [ ] `spring-boot-starter-data-redis-reactive` is declared in `hermes-gateway/pom.xml`.
- [ ] `remoteAddrKeyResolver` bean is defined in `RateLimiterConfig`.
- [ ] `RequestRateLimiter` filter is applied to at least the `echo-route-yaml` route.
- [ ] Rate limiter parameters are externalized via `${hermes.ratelimit.*}` properties.
- [ ] With local Redis running, sending requests above `burstCapacity` returns HTTP `429`.
- [ ] `RateLimiterConfigTest` passes with `.\mvnw.cmd test -pl hermes-gateway --also-make`.

## Validation

Unit tests:

```powershell
.\mvnw.cmd test -pl hermes-gateway --also-make -Dtest=RateLimiterConfigTest
```

Expected: `BUILD SUCCESS`, 2 tests passing.

Rate limit smoke test (requires Redis running):

```bash
docker compose up -d
# Start gateway, then:
for i in {1..8}; do curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8080/api/echo/ping; done
```

Expected: first 5 return `200` (or whatever the downstream returns), next 3 return `429`.

## Commit suggestion

```text
feat: add Redis-backed token bucket rate limiting per client IP
```

## Notes

- **`Mono.justOrEmpty` vs `Mono.just`**: `getRemoteAddress()` can return `null` in test environments (e.g., `MockServerHttpRequest` without `.remoteAddress(...)`). `justOrEmpty` prevents a `NullPointerException` and emits an empty `Mono` instead — Spring Cloud Gateway will deny the request with 429 if the key is empty and `deny-empty-key=true` (the default). This is the safe default.
- **SpEL expression `#{@remoteAddrKeyResolver}`**: the `#{}` syntax in YAML tells Spring Cloud Gateway to resolve the bean by name from the application context. The bean name must exactly match (case-sensitive).
- **Redis not running locally**: without Redis, the rate limiter filter will fail at startup (autoconfiguration will fail to connect). Use `application-local.yml` to disable rate limiting entirely during offline development if needed by removing the `RequestRateLimiter` filter from the local profile routes, or set `spring.cloud.gateway.redis-rate-limiter.include-headers=false` to suppress header injection without failing.
