# TASK-002 — Programmatic route configuration (RouteLocator)

## Status

Planned

## Parent story

`US-002-gateway-mvp.md`

## Summary

Define a second gateway route using a `RouteLocator` bean in a `@Configuration` class. The programmatic approach is more testable than YAML, supports conditional logic, and integrates cleanly with Spring's dependency injection — making it the preferred pattern for complex routes. Both this and the YAML route are kept in the MVP to demonstrate the trade-offs.

## Technical scope

This task includes:

- `GatewayConfig.java` in `dev.eithel.hermes.gateway.config`.
- A `RouteLocator` bean built via `RouteLocatorBuilder`.
- Route predicate: `Path=/api/ping/**` (distinct from the YAML echo route).
- Route filter: `StripPrefix=1`.
- Downstream URI injected via `@Value("${hermes.downstream.echo-uri}")` (reuses the same property as TASK-001 for simplicity in the MVP).
- A unit test for `GatewayConfig` verifying the route definition without starting a server.

## Out of scope

This task does not include:

- Replacing or removing the YAML route from TASK-001.
- Any filter implementation beyond `StripPrefix`.
- Resilience4j, rate limiting, or auth predicates.

## Implementation notes

- **Class**: `dev.eithel.hermes.gateway.config.GatewayConfig`
- **Route id**: `ping-route-java`
- **Predicate**: `Path=/api/ping/**`
- **Filter**: `StripPrefix=1`
- The `RouteLocatorBuilder` DSL and the YAML DSL are equivalent — Spring Cloud Gateway merges routes from both sources at startup.
- Inject the downstream URI via constructor parameter or `@Value` — avoid hardcoding URIs in configuration classes.

### Reference implementation

```java
package dev.eithel.hermes.gateway.config;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.cloud.gateway.route.RouteLocator;
import org.springframework.cloud.gateway.route.builder.RouteLocatorBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class GatewayConfig {

    @Bean
    public RouteLocator hermesRoutes(RouteLocatorBuilder builder,
                                     @Value("${hermes.downstream.echo-uri}") String echoUri) {
        return builder.routes()
                .route("ping-route-java", r -> r
                        .path("/api/ping/**")
                        .filters(f -> f.stripPrefix(1))
                        .uri(echoUri))
                .build();
    }
}
```

### Unit test reference

```java
@ExtendWith(MockitoExtension.class)
class GatewayConfigTest {

    @Test
    void routeLocator_definesExpectedRoutes() {
        RouteLocatorBuilder builder = new RouteLocatorBuilder(new StaticApplicationContext());
        GatewayConfig config = new GatewayConfig();

        RouteLocator locator = config.hermesRoutes(builder, "http://downstream:8090");

        StepVerifier.create(locator.getRoutes())
                .expectNextMatches(route -> route.getId().equals("ping-route-java"))
                .verifyComplete();
    }
}
```

## Expected files

```text
hermes-gateway/
└── src/
    ├── main/java/dev/eithel/hermes/gateway/config/
    │   └── GatewayConfig.java
    └── test/java/dev/eithel/hermes/gateway/config/
        └── GatewayConfigTest.java
```

## Acceptance criteria

- [ ] `GatewayConfig.java` exists in `dev.eithel.hermes.gateway.config`.
- [ ] The `RouteLocator` bean defines a route with id `ping-route-java` and predicate `Path=/api/ping/**`.
- [ ] The downstream URI is injected via `${hermes.downstream.echo-uri}`, not hardcoded.
- [ ] `GET /actuator/gateway/routes` at runtime lists both `echo-route-yaml` and `ping-route-java`.
- [ ] `GatewayConfigTest` passes with `.\mvnw.cmd test -pl hermes-gateway`.

## Validation

```powershell
.\mvnw.cmd test -pl hermes-gateway --also-make -Dtest=GatewayConfigTest
```

Expected: `BUILD SUCCESS`, 1 test passing.

Runtime check:

```bash
curl -s http://localhost:8080/actuator/gateway/routes | jq '[.[].id]'
```

Expected: `["echo-route-yaml", "ping-route-java"]` (order may vary).

## Commit suggestion

```text
feat: add programmatic echo route via RouteLocator bean
```

## Notes

- When both YAML routes and `RouteLocator` beans are present, Spring Cloud Gateway merges them. There is no conflict — each route needs a unique id.
- The programmatic approach enables conditional route registration (e.g., `@ConditionalOnProperty`), which is useful when routes need to be toggled per environment. This is not needed now but is a key reason to prefer programmatic configuration for production routes.
- `StepVerifier` requires `reactor-test` on the test classpath — it is included transitively via `spring-cloud-starter-gateway` in test scope. No additional dependency is needed.
