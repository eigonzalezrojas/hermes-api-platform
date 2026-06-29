# TASK-005 — Configure Actuator health endpoint

## Status

Planned

## Parent story

`US-001-project-bootstrap.md`

## Summary

Configure Spring Boot Actuator in `hermes-gateway` so that `GET /actuator/health` returns `{"status":"UP"}`. This endpoint serves as the liveness probe for Kubernetes and the primary validation signal for the bootstrap milestone.

## Technical scope

This task includes:

- Actuator configuration in `application.yml`: expose the `health` endpoint over HTTP.
- Verify `spring-boot-starter-actuator` is already declared in `hermes-gateway/pom.xml` (added in TASK-004).
- Confirm the health endpoint is reachable and returns `{"status":"UP"}` when the gateway starts cleanly.
- Optional: configure `management.endpoint.health.show-details=when-authorized` for production safety.

## Out of scope

This task does not include:

- Readiness probe configuration (`/actuator/health/readiness`) — added when the gateway has real dependencies (Redis, DB) to check.
- Prometheus endpoint (`/actuator/prometheus`) — added in v0.5.0.
- Custom `HealthIndicator` implementations.
- Security restrictions on the actuator endpoints (deferred to the security story).

## Implementation notes

- **Configuration file**: `hermes-gateway/src/main/resources/application.yml`
- **Required properties**:
  ```yaml
  management:
    endpoints:
      web:
        exposure:
          include: health
    endpoint:
      health:
        show-details: always   # local only — restrict in production
  ```
- For the `local` profile, `show-details: always` is acceptable to aid debugging. In production (`application-prod.yml`), it should be `when-authorized`.
- No additional classes or configuration beans are required for this task — Actuator auto-configuration covers it.
- Actuator base path defaults to `/actuator`. Do **not** change it.

## Expected files

Modified file only:

```text
hermes-gateway/
└── src/main/resources/
    └── application.yml   ← add management configuration block
```

### application.yml after this task

```yaml
spring:
  application:
    name: hermes-gateway

server:
  port: 8080

management:
  endpoints:
    web:
      exposure:
        include: health
  endpoint:
    health:
      show-details: always
```

## Acceptance criteria

- [ ] `application.yml` includes the `management.endpoints.web.exposure.include=health` property.
- [ ] `GET http://localhost:8080/actuator/health` returns HTTP 200.
- [ ] Response body contains `{"status":"UP"}`.
- [ ] No other Actuator endpoints are exposed (only `health` in the include list).
- [ ] `.\mvnw.cmd clean verify` passes from the repository root.

## Validation

Start the gateway:

```powershell
.\mvnw.cmd -pl hermes-gateway spring-boot:run -Dspring-boot.run.profiles=local
```

In a separate terminal, check the health endpoint:

```bash
curl -s http://localhost:8080/actuator/health
```

Expected response:

```json
{"status":"UP"}
```

Full build validation (from repository root):

```powershell
.\mvnw.cmd clean verify
```

Expected: `BUILD SUCCESS`.

## Commit suggestion

```text
feat: expose actuator health endpoint on hermes-gateway
```

## Notes

- This is the final task of US-001. After this commit, the acceptance criteria of `US-001-project-bootstrap.md` should be fully satisfied and the story status updated to `Done`.
- The `show-details: always` setting is intentional for local development. When the `prod` profile is introduced, override it to `when-authorized` in `application-prod.yml`.
- Future health contributors (Redis, PostgreSQL, downstream services) will automatically appear in the health response once those dependencies are added — no additional configuration required.
