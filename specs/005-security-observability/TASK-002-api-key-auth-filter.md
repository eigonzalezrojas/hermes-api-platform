# TASK-002 — API key authentication filter

## Status

Planned

## Parent story

`US-005-security-and-observability.md`

## Summary

Implement an `ApiKeyAuthFilter` that validates `X-API-Key` headers on every request, and extend `hermes-admin` to publish API key hashes to Redis on creation and deletion. The gateway reads from Redis to validate keys without calling `hermes-admin` on the hot path, keeping latency low and the security boundary intact.

## Technical scope

This task includes:

- `ApiKeyAuthFilter.java` in `dev.eithel.hermes.gateway.filter` — a `WebFilter` (not `GlobalFilter`) that integrates with Spring Security's reactive context.
- Redis key schema for API key validation: `apikey:{hash}` → JSON metadata.
- `hermes-admin` extended: `ApiKeyService.create()` and `ApiKeyService.delete()` publish/remove the Redis entry via `ReactiveRedisTemplate`.
- `RedisApiKeyPublisher.java` in `hermes-admin` — encapsulates Redis write/delete operations.
- `spring-boot-starter-data-redis-reactive` added to `hermes-admin/pom.xml` (gateway already has it from US-003).
- `SecurityConfig` updated to allow the `ApiKeyAuthFilter` as an additional authentication path alongside JWT.
- Unit tests for `ApiKeyAuthFilter` and `RedisApiKeyPublisher`.

## Out of scope

This task does not include:

- API key expiry enforcement (the `expiresAt` field is stored in Redis but checked only when the gateway validates keys in a future story; for the MVP, only `active: true` is checked).
- Rate limiting per API key (still per-IP — deferred to a multi-tenancy story).
- API key rotation endpoint in `hermes-admin`.
- Caching the Redis result in-process (one Redis call per request — sub-millisecond in production, acceptable for MVP).

## Implementation notes

### Redis key schema

```
Key:   apikey:{sha256hex}
Value: {"tenantId":"team-platform","label":"ci-token","active":true}
TTL:   none (evicted on revocation by deleting the key)
```

The stored field is the SHA-256 hex of the raw key **without the salt prefix** used in the DB. For Redis lookups, the gateway computes `SHA-256(rawKey)` directly (no salt needed — the hash is the lookup key, not a stored verifier). This is a deliberate design split:
- **DB**: `{salt}:{salted-hash}` for defense-in-depth if the DB is breached.
- **Redis**: `{unsalted-hash}` as a lookup key — Redis is a cache, not the authoritative store; the hash itself is not a credential.

**hermes-admin** must write both the salted hash to the DB (TASK-004 of US-004) and the unsalted hash key to Redis (this task).

### ApiKeyAuthFilter integration with Spring Security

The filter must set the `ReactiveSecurityContextHolder` so that Spring Security considers the request authenticated after API key validation. This allows the `SecurityWebFilterChain` to treat API key requests identically to JWT requests.

```java
@Component
@Order(Ordered.HIGHEST_PRECEDENCE + 10)
public class ApiKeyAuthFilter implements WebFilter {

    private static final String API_KEY_HEADER = "X-API-Key";

    private final ReactiveRedisTemplate<String, String> redisTemplate;
    private final ObjectMapper objectMapper;

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, WebFilterChain chain) {
        String rawKey = exchange.getRequest().getHeaders().getFirst(API_KEY_HEADER);
        if (rawKey == null || rawKey.isBlank()) {
            return chain.filter(exchange);   // no API key — let JWT path handle it
        }

        String keyHash = sha256Hex(rawKey);
        String redisKey = "apikey:" + keyHash;

        return redisTemplate.opsForValue().get(redisKey)
                .flatMap(json -> parseMetadata(json))
                .filter(meta -> meta.active())
                .flatMap(meta -> {
                    exchange.getAttributes().put(HermesHeaders.TENANT_ID, meta.tenantId());
                    Authentication auth = new UsernamePasswordAuthenticationToken(
                            meta.tenantId(), null, List.of());
                    return chain.filter(exchange)
                            .contextWrite(ReactiveSecurityContextHolder.withAuthentication(auth));
                })
                .switchIfEmpty(unauthorized(exchange));
    }

    private Mono<Void> unauthorized(ServerWebExchange exchange) {
        exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
        return exchange.getResponse().setComplete();
    }
}
```

### RedisApiKeyPublisher (hermes-admin)

```java
@Component
@RequiredArgsConstructor
public class RedisApiKeyPublisher {

    private final ReactiveRedisTemplate<String, String> redisTemplate;
    private final ObjectMapper objectMapper;

    public Mono<Void> publish(String rawKey, String tenantId, String label) {
        String keyHash = sha256Hex(rawKey);
        String redisKey = "apikey:" + keyHash;
        String value = toJson(new ApiKeyMeta(tenantId, label, true));
        return redisTemplate.opsForValue().set(redisKey, value).then();
    }

    public Mono<Void> revoke(String rawKey) {
        return redisTemplate.delete("apikey:" + sha256Hex(rawKey)).then();
    }
}
```

### SecurityConfig update

Add `.httpBasic(ServerHttpSecurity.HttpBasicSpec::disable)` explicitly and ensure the API key filter order (`+10`) is after correlation ID (`+1`) and logging (`+2`) but before Spring Security's own chain (`-100` relative scale — see Notes).

### application-local.yml additions (hermes-admin)

```yaml
spring:
  data:
    redis:
      host: localhost
      port: 6379
```

## Expected files

```text
hermes-gateway/
└── src/
    ├── main/java/dev/eithel/hermes/gateway/filter/
    │   └── ApiKeyAuthFilter.java
    └── test/java/dev/eithel/hermes/gateway/filter/
        └── ApiKeyAuthFilterTest.java

hermes-admin/
├── pom.xml                                            ← add spring-boot-starter-data-redis-reactive
└── src/
    ├── main/java/dev/eithel/hermes/admin/infrastructure/redis/
    │   └── RedisApiKeyPublisher.java
    └── test/java/dev/eithel/hermes/admin/infrastructure/redis/
        └── RedisApiKeyPublisherTest.java

hermes-common/
└── src/main/java/dev/eithel/hermes/common/
    └── HermesHeaders.java                             ← add TENANT_ID = "X-Tenant-ID" constant
```

## Acceptance criteria

- [ ] `POST /admin/api-keys` (hermes-admin) writes a Redis entry `apikey:{hash}` with tenant metadata.
- [ ] `DELETE /admin/api-keys/{id}` removes the Redis entry.
- [ ] `GET /api/echo/ping` with a valid `X-API-Key` passes authentication (downstream call succeeds).
- [ ] `GET /api/echo/ping` with an invalid or revoked `X-API-Key` returns `401 Unauthorized`.
- [ ] `GET /api/echo/ping` with neither `Authorization` nor `X-API-Key` returns `401 Unauthorized`.
- [ ] The `X-Tenant-ID` attribute is set on the exchange for authenticated API key requests.
- [ ] `ApiKeyAuthFilterTest` passes with `.\mvnw.cmd test -pl hermes-gateway --also-make`.
- [ ] `RedisApiKeyPublisherTest` passes with `.\mvnw.cmd test -pl hermes-admin --also-make`.

## Validation

```powershell
.\mvnw.cmd test -pl hermes-gateway,hermes-admin --also-make `
  -Dtest="ApiKeyAuthFilterTest,RedisApiKeyPublisherTest"
```

End-to-end smoke test (requires Docker + both services running):

```bash
RAW_KEY=$(curl -s -X POST http://localhost:8081/admin/api-keys \
  -H "Content-Type: application/json" \
  -d '{"label":"smoke","tenantId":"team-platform"}' | jq -r .rawKey)

# Valid key → forwarded
curl -s -o /dev/null -w "%{http_code}" \
  -H "X-API-Key: $RAW_KEY" http://localhost:8080/api/echo/ping

# No key → 401
curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/api/echo/ping
```

## Commit suggestion

```text
feat: add API key authentication filter with Redis-backed validation
```

## Notes

- **Why Redis and not calling hermes-admin directly?** A synchronous HTTP call to `hermes-admin` on every gateway request adds latency, creates a hard dependency (if admin is down, no request can be authenticated), and violates the principle that the gateway should be self-contained at runtime. Redis is already in the stack and provides sub-millisecond reads.
- **Unsalted hash in Redis vs salted hash in DB**: the unsalted SHA-256 in Redis is the *lookup key*, not a *stored secret*. If Redis is compromised and someone obtains `sha256(rawKey)`, they cannot reverse it to the original key (SHA-256 is one-way) and cannot use it to authenticate (the gateway accepts the `rawKey`, not the hash). The security model is: the raw key is the credential; the hash is the lookup index. The salted hash in the DB adds defense if the DB is breached separately.
- **`WebFilter` vs `GlobalFilter`**: `ApiKeyAuthFilter` implements `WebFilter` (Spring WebFlux) rather than `GlobalFilter` (Spring Cloud Gateway). This lets it integrate with `ReactiveSecurityContextHolder` and run within the Spring Security filter chain. `GlobalFilter` runs *outside* the security chain and cannot set the security context in a way that Spring Security respects.
- **`HermesHeaders.TENANT_ID`**: adding `X-Tenant-ID` to the common constants class ensures both the gateway (sets it) and future consumers (read it) use the same string without duplication.
