# TASK-001 — Create parent Maven project

## Status

Planned

## Parent story

`US-001-project-bootstrap.md`

## Summary

Create the root `pom.xml` that defines the multi-module Maven project. This POM acts as the build parent for all Hermes modules, centralizing dependency versions, plugin configuration, and Java version.

## Technical scope

This task includes:

- Root `pom.xml` with `packaging=pom`.
- `<parent>` pointing to `spring-boot-starter-parent` for Spring Boot dependency management.
- `<properties>` with `java.version=21` and `spring-cloud.version`.
- `<dependencyManagement>` importing `spring-cloud-dependencies` BOM.
- `<modules>` declaring `hermes-common` and `hermes-gateway`.
- `<build>` section with `spring-boot-maven-plugin` managed (not applied at root — applied per module).
- Common plugin configuration: `maven-compiler-plugin` targeting Java 21, `maven-surefire-plugin`.

## Out of scope

This task does not include:

- Any source code or module implementation.
- `hermes-admin` or `hermes-test` modules (declared in future tasks).
- Docker, Terraform, or CI/CD configuration.
- Dependency declarations beyond BOM imports and version properties.

## Implementation notes

- **groupId**: `dev.eithel`
- **artifactId**: `hermes-api-platform`
- **version**: `0.1.0-SNAPSHOT`
- **packaging**: `pom`
- Use `spring-boot-starter-parent` version `3.4.x` (latest stable at time of implementation).
- Use `spring-cloud-dependencies` version `2024.0.x` (compatible with Spring Boot 3.4.x).
- Set `<maven.compiler.source>` and `<maven.compiler.target>` to `21`.
- Enable annotation processing for Lombok via `maven-compiler-plugin` configuration.
- Do not add `spring-boot-maven-plugin` at root level — it must be added only to executable modules (`hermes-gateway`, `hermes-admin`).

## Expected files

```text
hermes-api-platform/
└── pom.xml
```

### Reference POM structure

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project>
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.4.x</version>
    </parent>

    <groupId>dev.eithel</groupId>
    <artifactId>hermes-api-platform</artifactId>
    <version>0.1.0-SNAPSHOT</version>
    <packaging>pom</packaging>

    <properties>
        <java.version>21</java.version>
        <spring-cloud.version>2024.0.x</spring-cloud.version>
    </properties>

    <modules>
        <module>hermes-common</module>
        <module>hermes-gateway</module>
    </modules>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring-cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
</project>
```

## Acceptance criteria

- [ ] `pom.xml` exists at the repository root with `packaging=pom`.
- [ ] `hermes-common` and `hermes-gateway` are declared in `<modules>`.
- [ ] `spring-boot-starter-parent` is set as parent with a Spring Boot 3.x version.
- [ ] `spring-cloud-dependencies` BOM is imported in `<dependencyManagement>`.
- [ ] `java.version=21` is set in `<properties>`.
- [ ] Running `.\mvnw.cmd clean validate` at root succeeds (modules can be absent at this stage).

## Validation

```powershell
.\mvnw.cmd clean validate
```

Expected: `BUILD SUCCESS` with all declared modules resolved.

## Commit suggestion

```text
chore: add parent Maven multi-module project
```

## Notes

- Do not use `<dependencyManagement>` to declare individual dependencies — use it only for BOM imports. Concrete dependencies belong in each module's POM.
- The Spring Boot and Spring Cloud versions must be compatible. Consult the Spring Cloud release calendar to confirm the correct pair.
