# TASK-004 — Create hermes-gateway module

## Status

Planned

## Parent story

`US-001-project-bootstrap.md`

## Summary

Create the `hermes-gateway` Maven module with a runnable Spring Boot application. This module is the core of the platform — it hosts Spring Cloud Gateway (WebFlux), all filters, route configuration, resilience, and observability. For this task, the goal is a minimal application that starts cleanly with the `local` profile.

## Technical scope

This task includes:

- `hermes-gateway/pom.xml` with Spring Cloud Gateway and Actuator dependencies.
- `HermesGatewayApplication.java` — Spring Boot entry point.
- `application.yml` — base configuration (port 8080, application name).
- `application-local.yml` — local profile overrides (suppress infrastructure-missing errors).
- Package skeleton: `config/`, `filter/`, `resilience/`, `metrics/` under `dev.eithel.hermes.gateway`.
- `spring-boot-maven-plugin` configured in this module's POM for executable JAR packaging.

## Out of scope

This task does not include:

- Any route definition or filter implementation.
- Redis, PostgreSQL, OAuth2, or any infrastructure dependency.
- Resilience4j, Micrometer, or observability configuration.
- Docker or container packaging.

## Implementation notes

- **Module name**: `hermes-gateway`
- **artifactId**: `hermes-gateway`
- **Parent**: inherits `dev.eithel:hermes-api-platform:0.1.0-SNAPSHOT`
- **Key dependencies**:
  - `spring-cloud-starter-gateway` — includes WebFlux; do **not** add `spring-boot-starter-web` (incompatible).
  - `spring-boot-starter-actuator` — exposes health endpoint.
  - `hermes-common` (local module dependency).
  - `lombok` with `optional=true`.
- **spring-boot-maven-plugin**: declare in this module's `<build>` section with `repackage` goal.
- **application.yml**:
  ```yaml
  spring:
    application:
      name: hermes-gateway
  server:
    port: 8080
  ```
- **application-local.yml** (local profile — empty for now, exists to suppress profile-not-found warnings):
  ```yaml
  # Local development profile — infrastructure overrides added incrementally
  ```
- **Main class**: `dev.eithel.hermes.gateway.HermesGatewayApplication` annotated with `@SpringBootApplication`.
- The `filter/`, `config/`, `resilience/`, and `metrics/` packages are created as empty skeletons with `package-info.java`.

## Expected files

```text
hermes-gateway/
├── pom.xml
└── src/
    ├── main/
    │   ├── java/
    │   │   └── dev/eithel/hermes/gateway/
    │   │       ├── HermesGatewayApplication.java
    │   │       ├── config/
    │   │       │   └── package-info.java
    │   │       ├── filter/
    │   │       │   └── package-info.java
    │   │       ├── resilience/
    │   │       │   └── package-info.java
    │   │       └── metrics/
    │   │           └── package-info.java
    │   └── resources/
    │       ├── application.yml
    │       └── application-local.yml
    └── test/
        └── java/
            └── dev/eithel/hermes/gateway/
```

### Reference main class

```java
package dev.eithel.hermes.gateway;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class HermesGatewayApplication {

    public static void main(String[] args) {
        SpringApplication.run(HermesGatewayApplication.class, args);
    }
}
```

## Acceptance criteria

- [ ] `hermes-gateway/pom.xml` exists and inherits from the parent POM.
- [ ] `HermesGatewayApplication.java` exists and is annotated with `@SpringBootApplication`.
- [ ] `application.yml` sets `server.port=8080` and `spring.application.name=hermes-gateway`.
- [ ] `application-local.yml` exists (content may be empty).
- [ ] `.\mvnw.cmd clean compile -pl hermes-gateway` succeeds with no errors.
- [ ] `spring-boot-starter-web` is **not** present in the dependencies (incompatible with WebFlux/Gateway).
- [ ] `spring-boot-maven-plugin` is configured in `hermes-gateway/pom.xml`.

## Validation

Compile only:

```powershell
.\mvnw.cmd clean compile -pl hermes-gateway --also-make
```

Start with local profile (actuator health not yet configured — see TASK-005):

```powershell
.\mvnw.cmd -pl hermes-gateway spring-boot:run -Dspring-boot.run.profiles=local
```

Expected: gateway starts on port 8080 with no startup errors. Spring Cloud Gateway routes list is empty (normal at this stage).

## Commit suggestion

```text
feat: add hermes-gateway module with Spring Cloud Gateway base application
```

## Notes

- `--also-make` (`-am`) ensures `hermes-common` is compiled first when running a targeted build on `hermes-gateway`.
- Spring Cloud Gateway auto-configures a reactive HTTP server (Netty via WebFlux). The startup log should show `Netty started on port 8080`.
- An empty routes list in the gateway is expected and not an error at this stage.
