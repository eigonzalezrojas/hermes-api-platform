# TASK-003 — Prometheus metrics

## Status

Planned

## Parent story

`US-005-security-and-observability.md`

## Summary

Expose a Prometheus-format metrics endpoint at `/actuator/prometheus` and instrument the gateway with custom counters for rate limit rejections and circuit breaker state changes. Spring Boot Actuator with Micrometer already provides HTTP server metrics out of the box — this task adds the Prometheus registry and the gateway-specific instrumentation.

## Technical scope

This task includes:

- `micrometer-registry-prometheus` dependency added to `hermes-gateway/pom.xml`.
- `/actuator/prometheus` added to the Actuator exposure list in `application.yml`.
- `HermesMetrics.java` in `dev.eithel.hermes.gateway.metrics` — a Spring component registering named Micrometer counters.
- `RateLimitingMetricsFilter.java` — a `GlobalFilter` that increments `hermes.rate.limit.rejected` when a `429` response is observed.
- `CircuitBreakerMetricsListener.java` — a Resilience4j event consumer that increments `hermes.circuit.breaker.open` when the circuit transitions to OPEN.
- Unit tests for `HermesMetrics` using `SimpleMeterRegistry`.

## Out of scope

This task does not include:

- Grafana dashboard definitions (separate ops tooling).
- Custom histograms or SLO metrics (deferred — requires SLA definition first).
- Prometheus scrape configuration or `prometheus.yml` (ops responsibility; the endpoint is the deliverable).
- `hermes-admin` metrics (admin API has standard Spring Boot Actuator metrics out of the box with no custom instrumentation needed in this milestone).
- AWS CloudWatch metrics export.

## Implementation notes

- **Dependency**: `micrometer-registry-prometheus`. Spring Boot's Actuator auto-configures a `PrometheusMeterRegistry` and the `/actuator/prometheus` endpoint when this is on the classpath.
- **Expose endpoint**: add `prometheus` to `management.endpoints.web.exposure.include`.
- **Out-of-the-box metrics** (no code needed): `http.server.requests` (tagged with `method`, `uri`, `status`, `outcome`), JVM memory, GC, CPU, thread pools, Netty connection metrics.
- **Custom metric: `hermes.rate.limit.rejected`**: a `Counter` tagged with `route` (the gateway route ID). Incremented by `RateLimitingMetricsFilter` when it detects a `429` response from the `RequestRateLimiter` filter.
- **Custom metric: `hermes.circuit.breaker.open`**: a `Counter` tagged with `name` (circuit breaker name, e.g., `echo-cb`). Incremented by `CircuitBreakerMetricsListener` on `CircuitBreakerOnStateTransitionEvent` when the new state is `OPEN`.
- **`RateLimitingMetricsFilter` order**: must run *after* the route filters that generate the 429 — set to `Ordered.LOWEST_PRECEDENCE - 1` so it wraps the entire chain and observes the final response status.
- **`MeterRegistry` injection**: inject `MeterRegistry` directly into components. Micrometer auto-configures a composite registry that forwards to all registered registries (Prometheus in this case).
- **Metric naming convention**: Micrometer auto-converts dot-notation names to the target system format. `hermes.rate.limit.rejected` becomes `hermes_rate_limit_rejected_total` in Prometheus format.

### HermesMetrics reference

```java
package dev.eithel.hermes.gateway.metrics;

import io.micrometer.core.instrument.Counter;
import io.micrometer.core.instrument.MeterRegistry;
import org.springframework.stereotype.Component;

@Component
public class HermesMetrics {

    private final MeterRegistry meterRegistry;

    public HermesMetrics(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
    }

    public void incrementRateLimitRejected(String routeId) {
        Counter.builder("hermes.rate.limit.rejected")
                .description("Total requests rejected by the rate limiter")
                .tag("route", routeId)
                .register(meterRegistry)
                .increment();
    }

    public void incrementCircuitBreakerOpen(String circuitBreakerName) {
        Counter.builder("hermes.circuit.breaker.open")
                .description("Number of times a circuit breaker transitioned to OPEN state")
                .tag("name", circuitBreakerName)
                .register(meterRegistry)
                .increment();
    }
}
```

### RateLimitingMetricsFilter reference

```java
@Component
public class RateLimitingMetricsFilter implements GlobalFilter, Ordered {

    private final HermesMetrics hermesMetrics;

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        return chain.filter(exchange).doFinally(signal -> {
            if (exchange.getResponse().getStatusCode() == HttpStatus.TOO_MANY_REQUESTS) {
                String routeId = exchange.getAttribute(ServerWebExchangeUtils.GATEWAY_ROUTE_ATTR) != null
                        ? ((Route) exchange.getAttribute(ServerWebExchangeUtils.GATEWAY_ROUTE_ATTR)).getId()
                        : "unknown";
                hermesMetrics.incrementRateLimitRejected(routeId);
            }
        });
    }

    @Override
    public int getOrder() {
        return Ordered.LOWEST_PRECEDENCE - 1;
    }
}
```

### application.yml update

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, prometheus, gateway
  endpoint:
    health:
      show-details: always
  metrics:
    distribution:
      percentiles-histogram:
        http.server.requests: true   # enables P50/P95/P99 histograms
```

### Unit test reference

```java
class HermesMetricsTest {

    private final SimpleMeterRegistry registry = new SimpleMeterRegistry();
    private final HermesMetrics metrics = new HermesMetrics(registry);

    @Test
    void incrementRateLimitRejected_registersCounter() {
        metrics.incrementRateLimitRejected("echo-route-yaml");
        metrics.incrementRateLimitRejected("echo-route-yaml");

        Counter counter = registry.find("hermes.rate.limit.rejected")
                .tag("route", "echo-route-yaml")
                .counter();

        assertThat(counter).isNotNull();
        assertThat(counter.count()).isEqualTo(2.0);
    }

    @Test
    void incrementCircuitBreakerOpen_registersCounter() {
        metrics.incrementCircuitBreakerOpen("echo-cb");

        Counter counter = registry.find("hermes.circuit.breaker.open")
                .tag("name", "echo-cb")
                .counter();

        assertThat(counter).isNotNull();
        assertThat(counter.count()).isEqualTo(1.0);
    }
}
```

## Expected files

```text
hermes-gateway/
├── pom.xml                                            ← add micrometer-registry-prometheus
└── src/
    ├── main/
    │   ├── java/dev/eithel/hermes/gateway/
    │   │   ├── metrics/
    │   │   │   └── HermesMetrics.java
    │   │   └── filter/
    │   │       └── RateLimitingMetricsFilter.java
    │   └── resources/
    │       └── application.yml                        ← add prometheus to exposure + histogram config
    └── test/java/dev/eithel/hermes/gateway/metrics/
        └── HermesMetricsTest.java
```

## Acceptance criteria

- [ ] `micrometer-registry-prometheus` is declared in `hermes-gateway/pom.xml`.
- [ ] `GET /actuator/prometheus` returns `200 OK` with `Content-Type: text/plain; version=0.0.4`.
- [ ] The response includes `http_server_requests_seconds_count` (auto-instrumented).
- [ ] After a rate-limited request, `hermes_rate_limit_rejected_total` appears in the Prometheus output.
- [ ] `HermesMetricsTest` passes with `.\mvnw.cmd test -pl hermes-gateway --also-make`.

## Validation

```powershell
.\mvnw.cmd test -pl hermes-gateway --also-make -Dtest=HermesMetricsTest
```

Prometheus endpoint check (requires running gateway):

```bash
curl -s http://localhost:8080/actuator/prometheus | grep -E "^(http_server_requests|hermes_)"
```

## Commit suggestion

```text
feat: expose Prometheus metrics with custom rate limit and circuit breaker counters
```

## Notes

- **`SimpleMeterRegistry` for tests**: `SimpleMeterRegistry` is an in-memory registry that holds all registered meters. It is the correct choice for unit testing Micrometer instrumentation — no Prometheus format output is needed, just counter values.
- **`Counter.builder(...).register(registry).increment()` is idempotent on the registry**: calling `Counter.builder(...).register(registry)` with the same name and tags always returns the same `Counter` instance from the registry. It is safe to call on every request — Micrometer caches the meter.
- **`percentiles-histogram: true`** enables Prometheus histogram buckets for P50/P95/P99 computation on the Prometheus/Grafana side. Without this, only `count` and `sum` are emitted (not enough for percentile queries). This does increase cardinality — one time series per bucket per URI per method. Keep it enabled for `http.server.requests`; avoid enabling it for high-cardinality custom metrics.
- **`GATEWAY_ROUTE_ATTR`**: Spring Cloud Gateway stores the matched route object in the exchange attributes under `ServerWebExchangeUtils.GATEWAY_ROUTE_ATTR`. This attribute is populated by the route matching phase — it is available by the time `RateLimitingMetricsFilter.doFinally()` runs.
