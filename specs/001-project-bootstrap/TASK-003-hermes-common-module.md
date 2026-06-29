# TASK-003 — Create hermes-common module

## Status

Planned

## Parent story

`US-001-project-bootstrap.md`

## Summary

Create the `hermes-common` Maven module that will hold shared DTOs, exceptions, constants, and utilities used across all other Hermes modules. For this task, only the module scaffold and package skeleton are required — no classes yet.

## Technical scope

This task includes:

- `hermes-common/pom.xml` inheriting from the parent POM.
- Standard Maven directory structure (`src/main/java`, `src/test/java`).
- Package skeleton: `dev.eithel.hermes.common.dto` and `dev.eithel.hermes.common.exception`.
- `lombok` as the only compile dependency for now.
- A `package-info.java` placeholder in each package to ensure the directories are tracked by Git.

## Out of scope

This task does not include:

- Any concrete DTO, exception, or utility class (those are added per feature story).
- Spring Boot auto-configuration or `@SpringBootApplication`.
- Any dependency other than `lombok`.

## Implementation notes

- **Module name**: `hermes-common`
- **artifactId**: `hermes-common`
- **packaging**: `jar` (default)
- **Parent**: inherits `dev.eithel:hermes-api-platform:0.1.0-SNAPSHOT`
- This module must **not** depend on `spring-cloud-starter-gateway` or `spring-boot-starter-web` — it is infrastructure-agnostic.
- `lombok` is declared as a dependency with `scope=provided` (or `optional=true`) since it is only needed at compile time.
- Keep placeholder files minimal — a `package-info.java` with only the `package` statement is sufficient.

## Expected files

```text
hermes-common/
├── pom.xml
└── src/
    ├── main/
    │   └── java/
    │       └── dev/eithel/hermes/common/
    │           ├── dto/
    │           │   └── package-info.java
    │           └── exception/
    │               └── package-info.java
    └── test/
        └── java/
            └── dev/eithel/hermes/common/
```

### Reference pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project>
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>dev.eithel</groupId>
        <artifactId>hermes-api-platform</artifactId>
        <version>0.1.0-SNAPSHOT</version>
    </parent>

    <artifactId>hermes-common</artifactId>
    <name>Hermes :: Common</name>
    <description>Shared DTOs, exceptions, constants, and utilities for the Hermes platform.</description>

    <dependencies>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
    </dependencies>
</project>
```

## Acceptance criteria

- [ ] `hermes-common/pom.xml` exists and inherits from the parent POM.
- [ ] Package directories `dto/` and `exception/` exist under `dev.eithel.hermes.common`.
- [ ] `.\mvnw.cmd clean compile -pl hermes-common` succeeds with no errors.
- [ ] `hermes-common` does not depend on any Spring Boot starter or Spring Cloud dependency.

## Validation

```powershell
.\mvnw.cmd clean compile -pl hermes-common
```

Expected: `BUILD SUCCESS`.

## Commit suggestion

```text
chore: add hermes-common module with package skeleton
```

## Notes

- The `dto/` and `exception/` sub-packages are intentionally empty. Classes are added incrementally as each feature story requires them.
- Future additions to this module: `HermesException` (base unchecked exception), `ApiErrorResponse` DTO, correlation ID constants.
