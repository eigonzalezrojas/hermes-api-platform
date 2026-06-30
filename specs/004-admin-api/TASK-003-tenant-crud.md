# TASK-003 — Tenant CRUD

## Status

Planned

## Parent story

`US-004-admin-api.md`

## Summary

Implement the `Tenant` domain in `hermes-admin`: Flyway migration, JPA entity, repository, service, and REST controller exposing CRUD at `/admin/tenants`. A tenant is the organizational unit that owns routes and API keys, enabling traffic isolation between teams or customers.

## Technical scope

This task includes:

- Flyway migration `V2__create_tenants_table.sql`.
- `Tenant` JPA entity in `dev.eithel.hermes.admin.domain.tenant`.
- `TenantRepository` extending `JpaRepository<Tenant, UUID>` with a `findByTenantId` query method.
- `TenantService` with create, findAll, findById, update, deactivate (soft-like via `active` flag), and delete operations.
- `TenantController` with REST endpoints at `/admin/tenants`.
- `TenantRequest` (input DTO with Bean Validation) and `TenantResponse` (output DTO).
- `@DataJpaTest` for the repository.
- `@WebMvcTest` for the controller.

## Out of scope

This task does not include:

- Tenant enforcement in `hermes-gateway` (deferred to v0.5.0 — the tenant header `X-Tenant-ID` is read but not yet validated against this table).
- Referential integrity between `tenants` and `routes` tables at the database level (the `tenant_id` column on `routes` is a plain `VARCHAR`, not a FK — this keeps schema evolution simpler for the MVP).
- Tenant-scoped rate limiting.
- Tenant deletion cascade (deleting a tenant does not cascade to its routes or API keys in this milestone).

## Implementation notes

- **Entity fields**: `id` (UUID, PK), `tenantId` (String, unique, not null — the logical identifier, e.g., `"platform"`, `"team-mobile"`), `name` (String, not null — human-readable label), `active` (boolean, default `true`), `createdAt` (Instant, `@CreationTimestamp`), `updatedAt` (Instant, `@UpdateTimestamp`).
- **`tenantId` uniqueness**: enforced at both the database level (UNIQUE constraint) and the service layer (check before insert, throw a specific exception that maps to 409 Conflict).
- **REST contract**:
  - `POST /admin/tenants` → `201 Created`, body: `TenantResponse`.
  - `GET /admin/tenants` → `200 OK`, body: `List<TenantResponse>`.
  - `GET /admin/tenants/{id}` → `200 OK` or `404 Not Found`.
  - `PUT /admin/tenants/{id}` → `200 OK` or `404 Not Found`.
  - `DELETE /admin/tenants/{id}` → `204 No Content` or `404 Not Found`.
- **`tenantId` format validation**: `@Pattern(regexp = "[a-z0-9][a-z0-9-]{1,48}[a-z0-9]")` — lowercase alphanumeric with hyphens, 3–50 characters. This ensures the tenant ID is safe for use as a header value and URL path segment.
- **`GlobalExceptionHandler`** (created in TASK-002): extend to handle `DuplicateTenantException` → `409 Conflict` with body `{"error": "Conflict", "message": "Tenant ID already exists."}`.

### Flyway migration V2

```sql
-- V2__create_tenants_table.sql
CREATE TABLE tenants (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id   VARCHAR(50) NOT NULL UNIQUE,
    name        VARCHAR(255) NOT NULL,
    active      BOOLEAN NOT NULL DEFAULT TRUE,
    created_at  TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);
```

### Entity reference

```java
@Entity
@Table(name = "tenants")
@Getter @Setter
@NoArgsConstructor
public class Tenant {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id;

    @Column(name = "tenant_id", nullable = false, unique = true, length = 50)
    private String tenantId;

    @Column(nullable = false)
    private String name;

    @Column(nullable = false)
    private boolean active = true;

    @CreationTimestamp
    @Column(name = "created_at", updatable = false)
    private Instant createdAt;

    @UpdateTimestamp
    @Column(name = "updated_at")
    private Instant updatedAt;
}
```

### TenantRequest DTO reference

```java
public record TenantRequest(
        @NotBlank @Pattern(regexp = "[a-z0-9][a-z0-9-]{1,48}[a-z0-9]",
                message = "tenantId must be 3-50 lowercase alphanumeric characters or hyphens")
        String tenantId,

        @NotBlank @Size(max = 255)
        String name,

        boolean active
) {}
```

### TenantService reference (key behavior)

```java
public TenantResponse create(TenantRequest request) {
    if (tenantRepository.existsByTenantId(request.tenantId())) {
        throw new DuplicateTenantException(request.tenantId());
    }
    Tenant tenant = new Tenant();
    tenant.setTenantId(request.tenantId());
    tenant.setName(request.name());
    tenant.setActive(request.active());
    return TenantResponse.from(tenantRepository.save(tenant));
}
```

## Expected files

```text
hermes-admin/
└── src/
    ├── main/
    │   ├── java/dev/eithel/hermes/admin/
    │   │   ├── domain/tenant/
    │   │   │   ├── Tenant.java
    │   │   │   ├── TenantRepository.java
    │   │   │   ├── TenantService.java
    │   │   │   ├── TenantController.java
    │   │   │   ├── TenantRequest.java
    │   │   │   └── TenantResponse.java
    │   │   └── web/
    │   │       └── GlobalExceptionHandler.java     ← extend with 409 Conflict handler
    │   └── resources/db/migration/
    │       └── V2__create_tenants_table.sql
    └── test/java/dev/eithel/hermes/admin/domain/tenant/
        ├── TenantRepositoryTest.java
        └── TenantControllerTest.java
```

## Acceptance criteria

- [ ] Flyway applies `V2__create_tenants_table.sql` after `V1` without errors.
- [ ] `POST /admin/tenants` with a valid body returns `201 Created` with `id` and `tenantId`.
- [ ] `POST /admin/tenants` with a duplicate `tenantId` returns `409 Conflict`.
- [ ] `POST /admin/tenants` with an invalid `tenantId` format (e.g., `"My Tenant!"`) returns `400 Bad Request`.
- [ ] `GET /admin/tenants/{id}` with a non-existent ID returns `404 Not Found`.
- [ ] `DELETE /admin/tenants/{id}` returns `204 No Content`.
- [ ] `TenantRepositoryTest` passes with `@DataJpaTest`.
- [ ] `TenantControllerTest` passes with `@WebMvcTest`.

## Validation

```powershell
.\mvnw.cmd test -pl hermes-admin --also-make -Dtest="TenantRepositoryTest,TenantControllerTest"
```

Functional test (requires running admin service):

```bash
# Create a tenant
curl -s -X POST http://localhost:8081/admin/tenants \
  -H "Content-Type: application/json" \
  -d '{"tenantId":"team-platform","name":"Platform Team","active":true}' | jq .

# Duplicate → 409
curl -s -X POST http://localhost:8081/admin/tenants \
  -H "Content-Type: application/json" \
  -d '{"tenantId":"team-platform","name":"Duplicate"}' | jq .
```

## Commit suggestion

```text
feat: add Tenant CRUD endpoints with unique tenantId validation
```

## Notes

- **Why not a FK from `routes.tenant_id` to `tenants.tenant_id`?** A FK would require tenant records to exist before routes, complicating seeding and migration rollbacks. Since tenant enforcement happens in the gateway (not in the admin DB), the loose coupling is intentional for the MVP. A FK can be added later if data integrity becomes a concern.
- **`existsByTenantId` vs catching `DataIntegrityViolationException`**: catching the database exception works but maps a generic DB error to a business rule, coupling the service to JPA internals. An explicit pre-check with `existsByTenantId` is more readable and produces a cleaner error message, at the cost of one extra query. For low-frequency admin operations, this trade-off is correct.
- **`tenantId` is immutable after creation**: the `PUT /admin/tenants/{id}` endpoint allows updating `name` and `active` but not `tenantId`. Changing the tenant identifier would invalidate all references in `routes.tenant_id` and `api_keys.tenant_id`. Enforce this in `TenantService.update()` by ignoring the `tenantId` field from the request or by throwing `400 Bad Request` if it differs from the stored value.
