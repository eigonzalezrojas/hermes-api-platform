# US-001 — Project Bootstrap

## Status

Planned

## Summary

As a backend engineer,
I want a working Maven multi-module project with a running Spring Cloud Gateway instance that exposes a health endpoint,
so that I have a stable, buildable foundation from which to incrementally add gateway features, filters, and infrastructure.

## Business / Engineering value

This story establishes the structural and operational baseline of the platform. Without it, no feature work can begin. A correct multi-module layout enforces module boundaries from day one, preventing the coupling issues that arise when structure is added retroactively.

Delivering a running gateway — even without routes or filters — proves that the build toolchain, dependency management, and Spring Boot auto-configuration are correctly wired. It also validates the local development workflow before any complexity is added.

## Scope

This story includes:

- A parent Maven project (`hermes-api-platform`) with multi-module POM.
- Maven Wrapper (`mvnw`, `mvnw.cmd`, `.mvn/wrapper/`) committed to the repository.
- A `hermes-common` module with package skeleton for shared DTOs and exceptions.
- A `hermes-gateway` module with a `HermesGatewayApplication` Spring Boot main class.
- Spring Boot Actuator health endpoint reachable at `GET /actuator/health`.
- The gateway starts successfully on port `8080` with the `local` Spring profile.

## Out of scope

This story does not include:

- Any gateway route configuration or filter implementation.
- Redis, PostgreSQL, or any infrastructure dependency.
- Authentication, rate limiting, or resilience configuration.
- Docker Compose or containerization.
- CI/CD pipeline setup.
- The `hermes-admin` or `hermes-test` modules.

## Acceptance criteria

- [ ] `.\mvnw.cmd clean verify` (Windows) or `./mvnw clean verify` (Linux/macOS) completes successfully from the repository root.
- [ ] `.\mvnw.cmd -pl hermes-gateway spring-boot:run -Dspring-boot.run.profiles=local` starts the gateway without errors.
- [ ] `GET http://localhost:8080/actuator/health` returns HTTP 200 with body `{"status":"UP"}`.
- [ ] The project compiles with Java 21 and produces no warnings related to module boundaries.
- [ ] Maven Wrapper is committed and functional without a global Maven installation.
- [ ] Both `hermes-common` and `hermes-gateway` are declared as modules in the parent POM.

## Technical notes

- **Parent POM**: `groupId=dev.eithel`, `artifactId=hermes-api-platform`, `version=0.1.0-SNAPSHOT`, `packaging=pom`.
- **Spring Boot parent**: use `spring-boot-starter-parent` as the parent of the parent POM (BOM import chain).
- **Spring Cloud**: add `spring-cloud-dependencies` BOM via `<dependencyManagement>` for future Spring Cloud Gateway dependency resolution.
- **Java version**: `<java.version>21</java.version>` in parent POM properties.
- **hermes-common**: plain Java module, no Spring Boot auto-configuration. Only `lombok` as a dependency for now.
- **hermes-gateway**: depends on `hermes-common`. Key dependencies: `spring-cloud-starter-gateway`, `spring-boot-starter-actuator`.
- **Base package**: `dev.eithel.hermes`.
- **Spring profile**: a `local` profile is expected. For this story, it can be empty — just enough to start without errors.
- **Port**: `server.port=8080` in `application.yml`.

## Related tasks

- `TASK-001-parent-maven-project.md`
- `TASK-002-maven-wrapper.md`
- `TASK-003-hermes-common-module.md`
- `TASK-004-hermes-gateway-module.md`
- `TASK-005-actuator-health-endpoint.md`

## Dependencies

- None. This is the first story in the project.

## Validation

Build from root:

```powershell
.\mvnw.cmd clean verify
```

Start gateway:

```powershell
.\mvnw.cmd -pl hermes-gateway spring-boot:run -Dspring-boot.run.profiles=local
```

Health check:

```bash
curl http://localhost:8080/actuator/health
```

Expected response:

```json
{"status":"UP"}
```

## Notes

- The `local` Spring profile will be used throughout the project for local development. For now it only needs to suppress missing-infrastructure errors (Redis, PostgreSQL) that will be added in later stories.
- The `hermes-test` module is deferred to US-003 (Resilience and rate limiting) when Testcontainers will be introduced.
