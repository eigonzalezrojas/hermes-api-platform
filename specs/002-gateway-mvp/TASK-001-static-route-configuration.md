# TASK-001 — Static route configuration (YAML)

## Status

Planned

## Parent story

`US-002-gateway-mvp.md`

## Summary

Define a gateway route in `application.yml` using Spring Cloud Gateway's YAML DSL. This is the declarative approach to route configuration — no Java code required, configuration is human-readable, and it establishes the baseline routing behavior that the programmatic approach in TASK-002 will complement.

## Technical scope

This task includes:

- A `spring.cloud.gateway.routes` entry in `application.yml`.
- Route predicate: `Path=/api/echo/**`.
- Route filter: `StripPrefix=1` (removes the `/api` prefix segment before forwarding).
- Downstream URI sourced from a property (`${hermes.downstream.echo-uri}`) to allow test overrides.
- Custom property `hermes.downstream.echo-uri` declared in `application.yml` with a default local value.
- `application-local.yml` override of `hermes.downstream.echo-uri` (e.g., `http://localhost:8090`).

## Out of scope

This task does not include:

- Any Java configuration class or `RouteLocator` bean (that is TASK-002).
- Resilience4j or rate limiting filters on this route.
- Authentication predicates.
- A running downstream service (integration test uses WireMock — wired in TASK-003/004).

## Implementation notes

- **Route id**: `echo-route-yaml`
- **Predicate**: `Path=/api/echo/**`
- **Filter**: `StripPrefix=1` — a request to `/api/echo/ping` is forwarded as `/echo/ping`.
- **URI property**: `hermes.downstream.echo-uri` — externalizing the downstream URI is critical: tests override it to point at WireMock; staging/prod override it via Kubernetes ConfigMap or environment variable.
- **Default value in `application.yml`**: `http://localhost:8090` (placeholder; no service needs to run locally for the build to pass).

### application.yml route block

```yaml
hermes:
  downstream:
    echo-uri: http://localhost:8090

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
```

## Expected files

Modified file only:

```text
hermes-gateway/
└── src/main/resources/
    └── application.yml   ← add hermes.downstream block and spring.cloud.gateway.routes
```

## Acceptance criteria

- [ ] `application.yml` contains a `spring.cloud.gateway.routes` entry with id `echo-route-yaml`.
- [ ] The route predicate is `Path=/api/echo/**`.
- [ ] The route applies `StripPrefix=1`.
- [ ] The downstream URI is read from `${hermes.downstream.echo-uri}`.
- [ ] `GET /actuator/gateway/routes` at runtime lists `echo-route-yaml`.
- [ ] `.\mvnw.cmd clean compile -pl hermes-gateway --also-make` succeeds.

## Validation

Start the gateway and inspect the runtime route list:

```powershell
.\mvnw.cmd -pl hermes-gateway spring-boot:run -Dspring-boot.run.profiles=local
```

```bash
curl -s http://localhost:8080/actuator/gateway/routes | jq '.[].id'
```

Expected output includes `"echo-route-yaml"`.

To enable the `gateway` actuator endpoint, add to `application.yml`:

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, gateway
```

## Commit suggestion

```text
feat: add static echo route via YAML configuration
```

## Notes

- The `StripPrefix=1` filter strips the first path segment. A request for `/api/echo/ping` becomes `/echo/ping` at the downstream. Adjust the strip count if the route prefix changes.
- Externalizing the downstream URI via a property from the start avoids hardcoded test/prod splits later. The pattern `${hermes.downstream.<service>-uri}` should be used consistently for all downstream services.
- The `gateway` actuator endpoint must be explicitly included in `management.endpoints.web.exposure.include` — it is not exposed by default.
