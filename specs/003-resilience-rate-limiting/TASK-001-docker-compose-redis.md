# TASK-001 — Docker Compose with Redis

## Status

Planned

## Parent story

`US-003-resilience-and-rate-limiting.md`

## Summary

Add a `docker-compose.yml` at the repository root that defines a Redis service for local development. Redis is the shared state store for the gateway's token-bucket rate limiter. The compose file is designed for extensibility — PostgreSQL will be added in v0.4.0 without restructuring this file.

## Technical scope

This task includes:

- `docker-compose.yml` at the repository root with a `redis` service.
- Redis image: `redis:7-alpine` (minimal footprint, stable LTS).
- Port mapping: `6379:6379`.
- A named volume for Redis persistence across restarts (optional but avoids losing state during development).
- `application-local.yml` updated with Redis connection properties pointing to `localhost:6379`.
- `.gitignore` verified — `docker-compose.override.yml` should be excluded (allows developers to override ports locally without committing changes).

## Out of scope

This task does not include:

- PostgreSQL service (added in TASK-001 of the `004-admin-api` milestone).
- Redis authentication or TLS configuration (not needed for local development).
- Docker Compose production profiles or multi-environment setup.
- The `spring-boot-starter-data-redis-reactive` dependency (added in TASK-002).

## Implementation notes

- **Compose file version**: use `services:` top-level key (Compose V2 syntax — no `version:` field needed).
- **Redis image**: `redis:7-alpine`. Do not use `latest` — pin the major version for reproducibility.
- **Health check**: include a `healthcheck` using `redis-cli ping` so dependent services (future) can wait for Redis to be ready.
- **Named volume**: `redis-data` for data persistence between `docker compose down` and `docker compose up` cycles.
- **application-local.yml**: add `spring.data.redis.host=localhost` and `spring.data.redis.port=6379`. These values match the Docker Compose port mapping.

### docker-compose.yml reference

```yaml
services:
  redis:
    image: redis:7-alpine
    container_name: hermes-redis
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5

volumes:
  redis-data:
```

### application-local.yml additions

```yaml
spring:
  data:
    redis:
      host: localhost
      port: 6379
```

## Expected files

```text
hermes-api-platform/
└── docker-compose.yml                              ← new

hermes-gateway/
└── src/main/resources/
    └── application-local.yml                       ← add Redis connection block
```

## Acceptance criteria

- [ ] `docker-compose.yml` exists at the repository root.
- [ ] `docker compose up -d` starts Redis on port `6379` without errors.
- [ ] `docker compose ps` shows the `hermes-redis` container as healthy.
- [ ] `docker compose down` stops and removes the container (volume is preserved).
- [ ] `application-local.yml` contains `spring.data.redis.host` and `spring.data.redis.port`.
- [ ] `docker-compose.override.yml` is listed in `.gitignore`.

## Validation

```bash
docker compose up -d
docker compose ps
docker exec hermes-redis redis-cli ping
```

Expected: `PONG`.

```bash
docker compose down
```

Expected: container stopped, `redis-data` volume retained.

## Commit suggestion

```text
chore: add Docker Compose with Redis for local development
```

## Notes

- The `docker-compose.yml` is committed to the repository (it is project infrastructure, not a secret). Developers who prefer not to use Docker can run Redis natively — `application-local.yml` will connect to any Redis on `localhost:6379`.
- The `hermes-redis` container name is explicit to make it easy to target with `docker exec` and `docker logs` commands.
- PostgreSQL will be added as a second service in the same `docker-compose.yml` in the `004-admin-api` milestone. Keep the file open for extension — no changes to the Redis service will be needed then.
- Add `docker-compose.override.yml` to `.gitignore` now even if no override file exists yet. It is a common pattern for per-developer port overrides and avoids accidental commits of local machine-specific settings.
