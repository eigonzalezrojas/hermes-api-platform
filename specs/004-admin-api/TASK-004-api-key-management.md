# TASK-004 — API key management

## Status

Planned

## Parent story

`US-004-admin-api.md`

## Summary

Implement API key generation and management in `hermes-admin`. When a key is created, a cryptographically random plaintext key is generated, its hash is stored in the database, and the plaintext is returned to the caller exactly once. Subsequent requests return only metadata — the plaintext and its hash are never exposed again.

## Technical scope

This task includes:

- Flyway migration `V3__create_api_keys_table.sql`.
- `ApiKey` JPA entity in `dev.eithel.hermes.admin.domain.apikey`.
- `ApiKeyRepository` extending `JpaRepository<ApiKey, UUID>` with lookup by key hash and by tenant.
- `ApiKeyService` with: key generation, findAll, findById, findByTenantId, and revoke (delete).
- `ApiKeyController` at `/admin/api-keys`.
- `ApiKeyCreationResponse` (includes `rawKey` field — only used on creation).
- `ApiKeyResponse` (metadata only — no `rawKey`, no hash).
- `ApiKeyHasher` component encapsulating the hash algorithm.
- `@DataJpaTest` for the repository.
- `@WebMvcTest` for the controller.

## Out of scope

This task does not include:

- API key validation in `hermes-gateway` (the gateway does not yet check incoming `X-API-Key` headers — deferred to the security story, v0.5.0).
- API key rotation endpoint (covered by delete + recreate for now).
- Key expiry enforcement (the `expiresAt` field is stored but expiry is not enforced in this milestone).
- Rate limiting per API key (deferred — rate limits are currently per-IP).

## Implementation notes

- **Entity fields**: `id` (UUID, PK), `label` (String, not null — human-readable name, e.g., `"ci-pipeline-token"`), `keyHash` (String, not null, unique — stores `{salt}:{hash}`), `tenantId` (String, not null — which tenant owns this key), `active` (boolean, default `true`), `createdAt` (Instant), `expiresAt` (Instant, nullable — for future expiry enforcement), `lastUsedAt` (Instant, nullable — updated by the gateway when it validates the key; null in this milestone).
- **Key generation algorithm**:
  1. Generate a 32-byte random value using `SecureRandom`.
  2. Base64-URL-encode it (no padding) → `rawKey` (43 characters, URL-safe).
  3. Generate a UUID as `salt`.
  4. Compute `hash = SHA-256(salt + rawKey)` and hex-encode.
  5. Store `keyHash = "{salt}:{hexHash}"`.
  6. Return `rawKey` in the creation response — never persist it.
- **Key format prefix**: prefix the `rawKey` with `hms_` (e.g., `hms_<base64url>`). This makes gateway API keys recognizable in logs and avoids confusion with other credentials. Common pattern used by Stripe, GitHub, and others.
- **REST contract**:
  - `POST /admin/api-keys` → `201 Created`, body: `ApiKeyCreationResponse` (includes `rawKey`).
  - `GET /admin/api-keys` → `200 OK`, body: `List<ApiKeyResponse>` (no `rawKey`).
  - `GET /admin/api-keys/{id}` → `200 OK`, body: `ApiKeyResponse` (no `rawKey`), or `404`.
  - `GET /admin/api-keys?tenantId={tenantId}` → `200 OK`, filtered list.
  - `DELETE /admin/api-keys/{id}` → `204 No Content` or `404`.
  - **No `PUT`**: API keys are immutable. Rotation = delete + create.
- **`ApiKeyHasher` component**: isolates the hashing logic for testability. Implements `hashKey(String rawKey)` and `matches(String rawKey, String storedHash)` methods. Injected into `ApiKeyService`.

### Flyway migration V3

```sql
-- V3__create_api_keys_table.sql
CREATE TABLE api_keys (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    label        VARCHAR(255) NOT NULL,
    key_hash     VARCHAR(500) NOT NULL UNIQUE,
    tenant_id    VARCHAR(50) NOT NULL,
    active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at   TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    expires_at   TIMESTAMP WITH TIME ZONE,
    last_used_at TIMESTAMP WITH TIME ZONE
);

CREATE INDEX idx_api_keys_tenant_id ON api_keys (tenant_id);
CREATE INDEX idx_api_keys_key_hash ON api_keys (key_hash);
```

### ApiKeyHasher reference

```java
@Component
public class ApiKeyHasher {

    public String hash(String rawKey) {
        String salt = UUID.randomUUID().toString();
        String hexHash = sha256Hex(salt + rawKey);
        return salt + ":" + hexHash;
    }

    public boolean matches(String rawKey, String storedHash) {
        String[] parts = storedHash.split(":", 2);
        if (parts.length != 2) return false;
        String expectedHex = sha256Hex(parts[0] + rawKey);
        return MessageDigest.isEqual(
                expectedHex.getBytes(StandardCharsets.UTF_8),
                parts[1].getBytes(StandardCharsets.UTF_8)
        );
    }

    private String sha256Hex(String input) {
        try {
            MessageDigest digest = MessageDigest.getInstance("SHA-256");
            byte[] hashBytes = digest.digest(input.getBytes(StandardCharsets.UTF_8));
            return HexFormat.of().formatHex(hashBytes);
        } catch (NoSuchAlgorithmException e) {
            throw new IllegalStateException("SHA-256 not available", e);
        }
    }
}
```

### ApiKeyService key generation reference

```java
public ApiKeyCreationResponse create(ApiKeyRequest request) {
    byte[] randomBytes = new byte[32];
    new SecureRandom().nextBytes(randomBytes);
    String rawKey = "hms_" + Base64.getUrlEncoder().withoutPadding().encodeToString(randomBytes);

    ApiKey apiKey = new ApiKey();
    apiKey.setLabel(request.label());
    apiKey.setTenantId(request.tenantId());
    apiKey.setKeyHash(apiKeyHasher.hash(rawKey));
    apiKey.setActive(true);
    apiKey.setExpiresAt(request.expiresAt());

    ApiKey saved = apiKeyRepository.save(apiKey);
    return ApiKeyCreationResponse.from(saved, rawKey);
}
```

### Response DTOs reference

```java
// Returned only on creation — includes the rawKey
public record ApiKeyCreationResponse(
        UUID id,
        String label,
        String tenantId,
        String rawKey,          // ← plaintext, shown once
        boolean active,
        Instant createdAt,
        Instant expiresAt
) {
    public static ApiKeyCreationResponse from(ApiKey key, String rawKey) { ... }
}

// Returned on all other reads — rawKey and keyHash excluded
public record ApiKeyResponse(
        UUID id,
        String label,
        String tenantId,
        boolean active,
        Instant createdAt,
        Instant expiresAt,
        Instant lastUsedAt
) {
    public static ApiKeyResponse from(ApiKey key) { ... }
}
```

## Expected files

```text
hermes-admin/
└── src/
    ├── main/
    │   ├── java/dev/eithel/hermes/admin/
    │   │   ├── domain/apikey/
    │   │   │   ├── ApiKey.java
    │   │   │   ├── ApiKeyRepository.java
    │   │   │   ├── ApiKeyService.java
    │   │   │   ├── ApiKeyController.java
    │   │   │   ├── ApiKeyRequest.java
    │   │   │   ├── ApiKeyResponse.java
    │   │   │   └── ApiKeyCreationResponse.java
    │   │   └── infrastructure/security/
    │   │       └── ApiKeyHasher.java
    │   └── resources/db/migration/
    │       └── V3__create_api_keys_table.sql
    └── test/java/dev/eithel/hermes/admin/
        ├── domain/apikey/
        │   ├── ApiKeyRepositoryTest.java
        │   └── ApiKeyControllerTest.java
        └── infrastructure/security/
            └── ApiKeyHasherTest.java
```

## Acceptance criteria

- [ ] Flyway applies `V3__create_api_keys_table.sql` without errors.
- [ ] `POST /admin/api-keys` returns `201 Created` with a `rawKey` field starting with `hms_`.
- [ ] `GET /admin/api-keys/{id}` does NOT include `rawKey` or `keyHash` in the response.
- [ ] `GET /admin/api-keys?tenantId=platform` returns only keys belonging to that tenant.
- [ ] `DELETE /admin/api-keys/{id}` returns `204 No Content`.
- [ ] `ApiKeyHasher.matches(rawKey, hash)` returns `true` for the original raw key and `false` for any other value.
- [ ] `ApiKeyHasher` uses `MessageDigest.isEqual` for constant-time comparison (no timing side-channel).
- [ ] `ApiKeyHasherTest`, `ApiKeyRepositoryTest`, and `ApiKeyControllerTest` all pass.
- [ ] `.\mvnw.cmd clean verify` passes.

## Validation

```powershell
.\mvnw.cmd test -pl hermes-admin --also-make -Dtest="ApiKeyHasherTest,ApiKeyRepositoryTest,ApiKeyControllerTest"
```

Functional test (requires running admin service):

```bash
# Generate a key
RESPONSE=$(curl -s -X POST http://localhost:8081/admin/api-keys \
  -H "Content-Type: application/json" \
  -d '{"label":"ci-token","tenantId":"team-platform"}')

echo $RESPONSE | jq .rawKey    # ← appears once
KEY_ID=$(echo $RESPONSE | jq -r .id)

# Read back — no rawKey
curl -s http://localhost:8081/admin/api-keys/$KEY_ID | jq .

# Revoke
curl -s -X DELETE http://localhost:8081/admin/api-keys/$KEY_ID
```

## Commit suggestion

```text
feat: add API key generation with one-time plaintext exposure and SHA-256 hashing
```

## Notes

- **Why SHA-256 and not BCrypt?** BCrypt is designed to be slow (10+ rounds of work factor) to resist brute-force attacks on password hashes. API keys generated with `SecureRandom` are 32 bytes of entropy (2^256 possible values) — brute-forcing them is computationally infeasible regardless of hash speed. SHA-256 is appropriate here and avoids the 72-character input limit that BCrypt imposes.
- **`MessageDigest.isEqual` for constant-time comparison**: string equality in Java (`equals`, `==`) short-circuits on the first differing character, leaking timing information that can be used in a timing side-channel attack to determine how many characters of the hash matched. `MessageDigest.isEqual` always compares all bytes regardless of where the mismatch is.
- **`rawKey` is never logged**: ensure that the `ApiKeyCreationResponse` is not serialized into application logs. Review the logging configuration to exclude response bodies at the INFO level. In future, add a `@JsonIgnore` or custom serializer that redacts `rawKey` from any accidental log output.
- **`HexFormat.of().formatHex()`** was introduced in Java 17. Since the project targets Java 21, this is safe and avoids the older `DatatypeConverter.printHexBinary()` from JAXB (deprecated).
- **`expiresAt` is stored but not enforced** in this milestone. When the gateway validates API keys (v0.5.0), it will check `expiresAt` on every incoming request. Storing it now ensures no schema migration is needed then.
