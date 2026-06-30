# TASK-001 — Create hermes-admin module

## Status

Planned

## Parent story

`US-004-admin-api.md`

## Summary

Create the `hermes-admin` Maven module — a standalone Spring Boot 3.x application with Spring MVC, Spring Data JPA, Flyway, and a PostgreSQL datasource. This task covers the module scaffold, PostgreSQL service in Docker Compose, and the Flyway migration baseline. No domain entities or REST endpoints are implemented here — those follow in TASK-002 through TASK-004.

## Technical scope

This task includes:

- `hermes-admin/pom.xml` with Spring MVC, JPA, Flyway, PostgreSQL, Validation, and Actuator dependencies.
- `HermesAdminApplication.java` — Spring Boot entry point.
- `application.yml` — base configuration (port `8081`, application name, Flyway enabled).
- `application-local.yml` — local profile datasource pointing to Docker Compose PostgreSQL.
- PostgreSQL service added to the existing `docker-compose.yml`.
- Flyway migration directory: `src/main/resources/db/migration/` (empty at this stage — migrations added in TASK-002 through TASK-004).
- `spring-boot-maven-plugin` configured in `hermes-admin/pom.xml`.
- `hermes-admin` added to the parent POM `<modules>`.

## Out of scope

This task does not include:

- Any domain entity, repository, service, or controller.
- Flyway migration SQL files (added per TASK-002, TASK-003, TASK-004).
- Authentication or security configuration.
- Integration tests.

## Implementation notes

- **artifactId**: `hermes-admin`
- **Parent**: inherits `dev.eithel:hermes-api-platform:0.1.0-SNAPSHOT`
- **Port**: `8081` — distinct from the gateway (`8080`).
- **Base package**: `dev.eithel.hermes.admin`
- **Key dependencies**:
  - `spring-boot-starter-web` — Spring MVC (NOT `spring-cloud-starter-gateway`, NOT WebFlux).
  - `spring-boot-starter-data-jpa` — Spring Data JPA + Hibernate.
  - `spring-boot-starter-validation` — Bean Validation (Hibernate Validator).
  - `spring-boot-starter-actuator` — health endpoint.
  - `org.flywaydb:flyway-core` — database migrations.
  - `org.postgresql:postgresql` — JDBC driver (runtime scope).
  - `hermes-common` — shared constants and DTOs.
  - `lombok` with `optional=true`.
- **PostgreSQL Docker Compose**: add a `postgres` service to the existing `docker-compose.yml`. Database name: `hermesdb`, user: `hermes`, password: `hermes` (local dev only — never use these credentials in any other environment).
- **Flyway**: auto-configured by Spring Boot when `flyway-core` is on the classpath and a datasource is configured. Set `spring.flyway.enabled=true` explicitly for clarity.
- **Datasource** (local profile): `jdbc:postgresql://localhost:5432/hermesdb`, username `hermes`, password `hermes`.

### docker-compose.yml additions (postgres service)

```yaml
  postgres:
    image: postgres:16-alpine
    container_name: hermes-postgres
    environment:
      POSTGRES_DB: hermesdb
      POSTGRES_USER: hermes
      POSTGRES_PASSWORD: hermes
    ports:
      - "5432:5432"
    volumes:
      - postgres-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U hermes -d hermesdb"]
      interval: 5s
      timeout: 3s
      retries: 5

volumes:
  redis-data:
  postgres-data:    # ← add alongside existing redis-data
```

### application.yml reference

```yaml
spring:
  application:
    name: hermes-admin
  flyway:
    enabled: true
  jpa:
    open-in-view: false
    hibernate:
      ddl-auto: validate

server:
  port: 8081

management:
  endpoints:
    web:
      exposure:
        include: health
  endpoint:
    health:
      show-details: always
```

### application-local.yml reference

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/hermesdb
    username: hermes
    password: hermes
    driver-class-name: org.postgresql.Driver
  jpa:
    show-sql: true
```

### HermesAdminApplication reference

```java
package dev.eithel.hermes.admin;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class HermesAdminApplication {

    public static void main(String[] args) {
        SpringApplication.run(HermesAdminApplication.class, args);
    }
}
```

## Expected files

```text
hermes-api-platform/
├── pom.xml                                          ← add hermes-admin module
└── docker-compose.yml                               ← add postgres service + postgres-data volume

hermes-admin/
├── pom.xml
└── src/main/
    ├── java/dev/eithel/hermes/admin/
    │   └── HermesAdminApplication.java
    └── resources/
        ├── application.yml
        ├── application-local.yml
        └── db/migration/                            ← empty, migrations added in TASK-002–004
```

## Acceptance criteria

- [ ] `hermes-admin` is declared in the parent POM `<modules>`.
- [ ] `hermes-admin/pom.xml` exists with Spring MVC, JPA, Flyway, and PostgreSQL dependencies.
- [ ] `spring-cloud-starter-gateway` is **not** present in `hermes-admin/pom.xml`.
- [ ] `docker compose up -d` starts both Redis and PostgreSQL without errors.
- [ ] `.\mvnw.cmd -pl hermes-admin spring-boot:run -Dspring-boot.run.profiles=local` starts on port `8081` (Flyway runs with 0 migrations — success).
- [ ] `GET http://localhost:8081/actuator/health` returns `{"status":"UP"}`.
- [ ] `spring.jpa.hibernate.ddl-auto=validate` is set (Flyway owns the schema — Hibernate must not auto-create tables).
- [ ] `open-in-view: false` is set to prevent lazy-loading issues in REST controllers.

## Validation

```bash
docker compose up -d
```

```powershell
.\mvnw.cmd -pl hermes-admin spring-boot:run -Dspring-boot.run.profiles=local
```

```bash
curl -s http://localhost:8081/actuator/health
```

Expected: `{"status":"UP"}`.

Full build:

```powershell
.\mvnw.cmd clean verify
```

## Commit suggestion

```text
chore: add hermes-admin module with Spring MVC, JPA, Flyway, and PostgreSQL
```

## Notes

- `spring.jpa.hibernate.ddl-auto=validate` is critical: it tells Hibernate to check that the schema matches the entity model but never modify the database. Flyway is the single source of truth for schema changes. Setting this to `update` or `create-drop` would bypass Flyway and create an unmanaged schema.
- `open-in-view: false` disables the OSIV (Open Session In View) pattern, which keeps the JPA session open for the entire HTTP request lifecycle. OSIV can cause unexpected lazy-loading queries during serialization and makes latency profiling misleading. Always disable it explicitly.
- The Docker Compose credentials (`hermes`/`hermes`) are intentionally weak and only valid for local development. In staging and production, credentials are injected via Kubernetes Secrets or AWS Secrets Manager — never hardcoded.
