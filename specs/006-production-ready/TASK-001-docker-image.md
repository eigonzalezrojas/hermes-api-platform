# TASK-001 — Docker images

## Status

Planned

## Parent story

`US-006-production-ready.md`

## Summary

Create multi-stage Dockerfiles for `hermes-gateway` and `hermes-admin`, producing minimal, cache-efficient, production-grade container images. Enable Spring Boot layer indexing for optimal Docker layer caching. Add a `.dockerignore` to exclude build artifacts and sensitive files from the image context.

## Technical scope

This task includes:

- `hermes-gateway/Dockerfile` — multi-stage: Maven build stage → JRE runtime stage.
- `hermes-admin/Dockerfile` — same pattern.
- `.dockerignore` at the repository root.
- Spring Boot layer indexing enabled in both modules' `spring-boot-maven-plugin` config.
- `application-prod.yml` in both modules with production-safe defaults (no `show-details: always`, `show-sql: false`, etc.).
- Actuator `liveness` and `readiness` endpoints exposed (required by TASK-004 for Kubernetes probes).
- Verify the built image starts and passes a health check with `docker run`.

## Out of scope

This task does not include:

- Docker Compose changes (Compose is for local dev only — production uses Kubernetes).
- Image scanning (handled by GitHub Actions in TASK-002).
- ECR push (also TASK-002).
- Non-root user hardening beyond what the base image provides (Alpine JRE runs as non-root by default).

## Implementation notes

- **Build stage image**: `eclipse-temurin:21-jdk-alpine` — smallest JDK image with Alpine.
- **Runtime stage image**: `eclipse-temurin:21-jre-alpine` — JRE only, no compiler or build tools.
- **Multi-stage pattern**: the build stage compiles the fat JAR; the runtime stage copies only the extracted Spring Boot layers. This produces a smaller final image and maximizes Docker layer cache reuse — dependency layers rarely change and are cached independently from application class layers.
- **Spring Boot layer extraction**: Spring Boot 3.x fat JARs can be extracted with `java -Djarmode=layertools -jar app.jar extract`. The layers in order (slowest-changing first, fastest-changing last): `dependencies`, `spring-boot-loader`, `snapshot-dependencies`, `application`.
- **`spring-boot-maven-plugin` layer config** (add to both modules):
  ```xml
  <configuration>
      <layers>
          <enabled>true</enabled>
      </layers>
  </configuration>
  ```
- **Port**: `EXPOSE 8080` for gateway, `EXPOSE 8081` for admin.
- **JVM flags**: set `JAVA_OPTS` env var for GC tuning and container-aware memory limits. Default: `-XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0 -XX:+ExitOnOutOfMemoryError`.
- **`application-prod.yml`** key settings:
  - `management.endpoint.health.show-details: when-authorized`
  - `spring.jpa.show-sql: false` (admin only)
  - `logging.level.root: WARN`, `logging.level.dev.eithel: INFO`
  - `management.endpoints.web.exposure.include: health, prometheus` — never expose all endpoints in production.
- **Liveness and readiness groups**: Spring Boot Actuator requires explicit configuration to separate liveness from readiness probes.

### Liveness/readiness probe configuration (application.yml)

```yaml
management:
  endpoint:
    health:
      probes:
        enabled: true
      group:
        liveness:
          include: livenessState
        readiness:
          include: readinessState
```

This enables `GET /actuator/health/liveness` and `GET /actuator/health/readiness` as distinct probe endpoints.

### hermes-gateway/Dockerfile reference

```dockerfile
# ── Stage 1: Build ──────────────────────────────────────────────────────────
FROM eclipse-temurin:21-jdk-alpine AS build
WORKDIR /workspace

COPY mvnw mvnw.cmd ./
COPY .mvn .mvn
COPY pom.xml ./
COPY hermes-common/pom.xml hermes-common/
COPY hermes-gateway/pom.xml hermes-gateway/

# Download dependencies (cached unless pom.xml changes)
RUN ./mvnw dependency:go-offline -pl hermes-gateway --also-make -q

COPY hermes-common/src hermes-common/src
COPY hermes-gateway/src hermes-gateway/src

RUN ./mvnw package -pl hermes-gateway --also-make -DskipTests -q

# Extract layers for cache-efficient runtime image
RUN java -Djarmode=layertools \
    -jar hermes-gateway/target/hermes-gateway-*.jar extract \
    --destination hermes-gateway/target/extracted

# ── Stage 2: Runtime ────────────────────────────────────────────────────────
FROM eclipse-temurin:21-jre-alpine AS runtime
WORKDIR /app

ENV JAVA_OPTS="-XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0 -XX:+ExitOnOutOfMemoryError"
ENV SPRING_PROFILES_ACTIVE=prod

COPY --from=build /workspace/hermes-gateway/target/extracted/dependencies/ ./
COPY --from=build /workspace/hermes-gateway/target/extracted/spring-boot-loader/ ./
COPY --from=build /workspace/hermes-gateway/target/extracted/snapshot-dependencies/ ./
COPY --from=build /workspace/hermes-gateway/target/extracted/application/ ./

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD wget -qO- http://localhost:8080/actuator/health || exit 1

ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS org.springframework.boot.loader.launch.JarLauncher"]
```

### .dockerignore reference

```
# Build outputs
target/
**/target/

# IDE files
.idea/
*.iml
.vscode/

# Git
.git/
.gitignore

# Local config and secrets
**/*.env
.env
**/*-local.yml
**/application-local.yml

# Infra (not needed in app images)
infra/

# Docs
*.md
specs/
```

## Expected files

```text
hermes-api-platform/
├── .dockerignore
├── hermes-gateway/
│   ├── Dockerfile
│   └── src/main/resources/
│       └── application-prod.yml
└── hermes-admin/
    ├── Dockerfile
    └── src/main/resources/
        └── application-prod.yml
```

## Acceptance criteria

- [ ] `docker build -f hermes-gateway/Dockerfile -t hermes-gateway:local .` succeeds from the repo root.
- [ ] `docker build -f hermes-admin/Dockerfile -t hermes-admin:local .` succeeds from the repo root.
- [ ] Gateway image size is under 300 MB (`docker image inspect hermes-gateway:local --format '{{.Size}}'`).
- [ ] `docker run --rm -p 8080:8080 -e SPRING_PROFILES_ACTIVE=prod hermes-gateway:local` starts and `GET /actuator/health` returns `200`.
- [ ] `GET /actuator/health/liveness` and `GET /actuator/health/readiness` both return `200` when healthy.
- [ ] Rebuilding after a non-code change (e.g., editing a comment) reuses the `dependencies` layer (Docker cache hit visible in build output).
- [ ] `application-prod.yml` sets `show-details: when-authorized` and does not expose sensitive actuator endpoints.

## Validation

```bash
# Build
docker build -f hermes-gateway/Dockerfile -t hermes-gateway:local .

# Run
docker run -d --name hermes-gw-test -p 8080:8080 hermes-gateway:local

# Check health
curl -s http://localhost:8080/actuator/health
curl -s http://localhost:8080/actuator/health/liveness
curl -s http://localhost:8080/actuator/health/readiness

# Image size
docker image inspect hermes-gateway:local --format '{{.Size}}' | \
  awk '{printf "%.0f MB\n", $1/1024/1024}'

# Cleanup
docker stop hermes-gw-test && docker rm hermes-gw-test
```

## Commit suggestion

```text
feat: add multi-stage Dockerfiles with Spring Boot layer extraction
```

## Notes

- **`dependency:go-offline` layer**: copying only POM files before source code allows Docker to cache the dependency download layer. If only source files change, Maven does not re-download dependencies on the next build.
- **`JarLauncher` in ENTRYPOINT**: Spring Boot 3.3+ uses `org.springframework.boot.loader.launch.JarLauncher` (note the `.launch.` sub-package). Earlier versions use `org.springframework.boot.loader.JarLauncher`. Verify the correct class name matches the Spring Boot version in use.
- **`SPRING_PROFILES_ACTIVE=prod`**: the `prod` profile is set as the default in the image. It can be overridden at runtime via `-e SPRING_PROFILES_ACTIVE=dev` or a Kubernetes `ConfigMap`. Never bake environment-specific secrets into the image.
- **`-XX:+ExitOnOutOfMemoryError`**: crashes the JVM immediately on OOM instead of running in a degraded state. Kubernetes will restart the pod, which is the correct recovery behavior in a container environment.
- **`wget` in `HEALTHCHECK`**: `curl` is not present in Alpine JRE images by default; `wget` is. Alternatively, use `curl` and install it with `RUN apk add --no-cache curl` in the runtime stage (adds ~1 MB).
