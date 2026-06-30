# TASK-002 — Route CRUD

## Status

Planned

## Parent story

`US-004-admin-api.md`

## Summary

Implement the `Route` domain in `hermes-admin`: JPA entity, Flyway migration, Spring Data JPA repository, service with CRUD logic, and a REST controller exposing `POST/GET/PUT/DELETE /admin/routes`. Routes represent the routing rules that will eventually be fed into the gateway's route configuration.

## Technical scope

This task includes:

- Flyway migration `V1__create_routes_table.sql`.
- `Route` JPA entity in `dev.eithel.hermes.admin.domain.route`.
- `RouteRepository` extending `JpaRepository<Route, UUID>`.
- `RouteService` with create, findAll, findById, update, and delete operations.
- `RouteController` with REST endpoints at `/admin/routes`.
- `RouteRequest` (input DTO with Bean Validation) and `RouteResponse` (output DTO).
- `JpaAttributeConverter` for serializing `List<String>` predicates and filters to/from a JSON `TEXT` column.
- `@DataJpaTest` for the repository.
- `@WebMvcTest` for the controller (mocking the service layer).

## Out of scope

This task does not include:

- Dynamic reload of routes into `hermes-gateway` at runtime (deferred to v1.0.0).
- Route validation against gateway predicate syntax.
- Pagination on `GET /admin/routes`.
- Tenant scoping — routes can optionally reference a `tenantId` (String, nullable) but no tenant FK constraint is added here.

## Implementation notes

- **Entity fields**: `id` (UUID, PK, auto-generated), `routeId` (String, unique, not null — this is the logical name used by the gateway), `uri` (String, not null), `predicates` (TEXT, JSON array), `filters` (TEXT, JSON array), `tenantId` (String, nullable), `enabled` (boolean, default `true`), `createdAt` (Instant, `@CreationTimestamp`), `updatedAt` (Instant, `@UpdateTimestamp`).
- **Predicates/filters storage**: stored as JSON text. An `AttributeConverter<List<String>, String>` serializes with `ObjectMapper.writeValueAsString` and deserializes with `ObjectMapper.readValue`. Annotate the entity field with `@Convert(converter = StringListConverter.class)`.
- **UUID primary key**: use `@GeneratedValue(strategy = GenerationType.UUID)` (Hibernate 6 / Java 21 compatible). Column type in the migration: `uuid`.
- **REST contract**:
  - `POST /admin/routes` → `201 Created`, body: `RouteResponse`.
  - `GET /admin/routes` → `200 OK`, body: `List<RouteResponse>`.
  - `GET /admin/routes/{id}` → `200 OK` or `404 Not Found`.
  - `PUT /admin/routes/{id}` → `200 OK` or `404 Not Found`.
  - `DELETE /admin/routes/{id}` → `204 No Content` or `404 Not Found`.
- **Bean Validation on `RouteRequest`**: `@NotBlank` on `routeId` and `uri`. `@Pattern` on `uri` to require a valid URI scheme prefix (`http://` or `https://`).
- **`@ControllerAdvice`**: create a `GlobalExceptionHandler` (or reuse if one exists) to handle `EntityNotFoundException` → 404 and `MethodArgumentNotValidException` → 400.

### Flyway migration V1

```sql
-- V1__create_routes_table.sql
CREATE TABLE routes (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    route_id    VARCHAR(255) NOT NULL UNIQUE,
    uri         VARCHAR(500) NOT NULL,
    predicates  TEXT,
    filters     TEXT,
    tenant_id   VARCHAR(255),
    enabled     BOOLEAN NOT NULL DEFAULT TRUE,
    created_at  TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_routes_tenant_id ON routes (tenant_id);
```

### Entity reference

```java
@Entity
@Table(name = "routes")
@Getter @Setter
@NoArgsConstructor
public class Route {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id;

    @Column(name = "route_id", nullable = false, unique = true)
    private String routeId;

    @Column(nullable = false)
    private String uri;

    @Convert(converter = StringListConverter.class)
    @Column(columnDefinition = "TEXT")
    private List<String> predicates = new ArrayList<>();

    @Convert(converter = StringListConverter.class)
    @Column(columnDefinition = "TEXT")
    private List<String> filters = new ArrayList<>();

    @Column(name = "tenant_id")
    private String tenantId;

    @Column(nullable = false)
    private boolean enabled = true;

    @CreationTimestamp
    @Column(name = "created_at", updatable = false)
    private Instant createdAt;

    @UpdateTimestamp
    @Column(name = "updated_at")
    private Instant updatedAt;
}
```

### RouteRequest DTO reference

```java
public record RouteRequest(
        @NotBlank String routeId,
        @NotBlank @Pattern(regexp = "https?://.+") String uri,
        List<String> predicates,
        List<String> filters,
        String tenantId,
        boolean enabled
) {}
```

### Controller reference

```java
@RestController
@RequestMapping("/admin/routes")
@RequiredArgsConstructor
public class RouteController {

    private final RouteService routeService;

    @PostMapping
    public ResponseEntity<RouteResponse> create(@Valid @RequestBody RouteRequest request) {
        return ResponseEntity.status(HttpStatus.CREATED).body(routeService.create(request));
    }

    @GetMapping
    public List<RouteResponse> findAll() {
        return routeService.findAll();
    }

    @GetMapping("/{id}")
    public RouteResponse findById(@PathVariable UUID id) {
        return routeService.findById(id);
    }

    @PutMapping("/{id}")
    public RouteResponse update(@PathVariable UUID id, @Valid @RequestBody RouteRequest request) {
        return routeService.update(id, request);
    }

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void delete(@PathVariable UUID id) {
        routeService.delete(id);
    }
}
```

## Expected files

```text
hermes-admin/
└── src/
    ├── main/
    │   ├── java/dev/eithel/hermes/admin/
    │   │   ├── domain/route/
    │   │   │   ├── Route.java
    │   │   │   ├── RouteRepository.java
    │   │   │   ├── RouteService.java
    │   │   │   ├── RouteController.java
    │   │   │   ├── RouteRequest.java
    │   │   │   └── RouteResponse.java
    │   │   ├── infrastructure/persistence/
    │   │   │   └── StringListConverter.java
    │   │   └── web/
    │   │       └── GlobalExceptionHandler.java
    │   └── resources/db/migration/
    │       └── V1__create_routes_table.sql
    └── test/java/dev/eithel/hermes/admin/domain/route/
        ├── RouteRepositoryTest.java
        └── RouteControllerTest.java
```

## Acceptance criteria

- [ ] Flyway applies `V1__create_routes_table.sql` on startup without errors.
- [ ] `POST /admin/routes` with a valid body returns `201 Created` with an `id` field.
- [ ] `GET /admin/routes/{id}` with a non-existent ID returns `404 Not Found`.
- [ ] `POST /admin/routes` with a missing `routeId` or `uri` returns `400 Bad Request`.
- [ ] `DELETE /admin/routes/{id}` returns `204 No Content`.
- [ ] Predicates and filters round-trip correctly: stored as JSON text, deserialized as `List<String>`.
- [ ] `RouteRepositoryTest` passes with `@DataJpaTest`.
- [ ] `RouteControllerTest` passes with `@WebMvcTest`.

## Validation

```powershell
.\mvnw.cmd test -pl hermes-admin --also-make -Dtest="RouteRepositoryTest,RouteControllerTest"
```

Functional test (requires Docker + running admin service):

```bash
curl -s -X POST http://localhost:8081/admin/routes \
  -H "Content-Type: application/json" \
  -d '{"routeId":"svc-a","uri":"http://service-a:8090","predicates":["Path=/api/a/**"],"filters":["StripPrefix=1"],"enabled":true}' | jq .
```

## Commit suggestion

```text
feat: add Route CRUD endpoints with JPA persistence and Flyway migration
```

## Notes

- **`@Getter @Setter` vs `@Data` on JPA entities**: avoid `@Data` on JPA entities — it generates `equals`/`hashCode` using all fields, which breaks Hibernate's entity identity contract (entities are equal if they have the same PK, regardless of field values). Use `@Getter @Setter` with a manual or `@EqualsAndHashCode(onlyExplicitlyIncluded = true)` annotation.
- **`StringListConverter`** should use a `static final ObjectMapper` instance — `ObjectMapper` is thread-safe and expensive to construct. Do not instantiate it per conversion.
- **`gen_random_uuid()`** requires the `pgcrypto` extension on PostgreSQL < 13. PostgreSQL 13+ includes it natively. Use `gen_random_uuid()` in migrations targeting PostgreSQL 13+ (which is the case with the `postgres:16-alpine` image).
- **`@WebMvcTest` scope**: `@WebMvcTest(RouteController.class)` loads only the web layer. The `RouteService` must be mocked with `@MockBean`. The `StringListConverter` is not in the web layer and does not need to be configured for these tests.
