# TASK-001 — OAuth2 JWT resource server

## Status

Planned

## Parent story

`US-005-security-and-observability.md`

## Summary

Configure `hermes-gateway` as an OAuth2 resource server that validates JWT Bearer tokens on every incoming request. Unauthenticated requests receive `401 Unauthorized` before reaching any route filter or downstream service. Public paths (health, prometheus, fallback) are explicitly permitted without authentication.

## Technical scope

This task includes:

- `spring-boot-starter-oauth2-resource-server` dependency added to `hermes-gateway/pom.xml`.
- `SecurityConfig.java` in `dev.eithel.hermes.gateway.config` with `@EnableWebFluxSecurity`.
- `SecurityWebFilterChain` bean configuring JWT validation and path-level authorization.
- `application.yml` updated with `spring.security.oauth2.resourceserver.jwt.*` properties.
- `application-local.yml` with a local JWT configuration (RSA public key or test secret).
- Unit test for the security configuration using `@WebFluxTest` and `@WithMockUser`.

## Out of scope

This task does not include:

- API key authentication (TASK-002).
- Role-based authorization (all authenticated requests are permitted regardless of claims).
- OAuth2 client configuration (the gateway is a resource server only).
- Keycloak or any external IdP setup (local dev uses a pre-generated RSA key pair or HS256 secret).
- CORS configuration.

## Implementation notes

- **Annotation**: `@EnableWebFluxSecurity` on `SecurityConfig` (NOT `@EnableWebSecurity` — that targets Spring MVC).
- **Never use `HttpSecurity`**: WebFlux security uses `ServerHttpSecurity`. The two APIs are incompatible; mixing them causes silent misconfiguration.
- **Permitted paths**: `/actuator/health`, `/actuator/prometheus`, `/fallback`. All other paths require authentication.
- **JWT validation**: Spring Security auto-configures a `ReactiveJwtDecoder` from `spring.security.oauth2.resourceserver.jwt.issuer-uri`. For local dev without a real IdP, use `spring.security.oauth2.resourceserver.jwt.public-key-location` pointing to a local RSA public key file.
- **Local RSA key pair**: generate with `openssl` and commit the **public key only** to the repository (`src/main/resources/jwt/public.pem`). The private key is never committed — it is only used by the test IdP to sign tokens.
- **Error handling**: Spring Security WebFlux returns `401` for missing/invalid tokens by default. No custom entry point is needed unless a custom JSON body is required.
- **CSRF**: disable CSRF for the gateway — it is a stateless API gateway, not a browser-facing application. CSRF protection is meaningful only for cookie-based sessions.
- **Tenant extraction from JWT**: extract the tenant claim from the JWT and store it in `exchange.getAttributes()` for use by downstream filters. Claim name: `tenant_id` (convention — document in the Notes section).

### pom.xml addition

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
</dependency>
```

### SecurityConfig reference

```java
package dev.eithel.hermes.gateway.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.reactive.EnableWebFluxSecurity;
import org.springframework.security.config.web.server.ServerHttpSecurity;
import org.springframework.security.config.web.server.ServerHttpSecurity.CsrfSpec;
import org.springframework.security.web.server.SecurityWebFilterChain;

@Configuration
@EnableWebFluxSecurity
public class SecurityConfig {

    private static final String[] PUBLIC_PATHS = {
            "/actuator/health",
            "/actuator/prometheus",
            "/fallback"
    };

    @Bean
    public SecurityWebFilterChain securityWebFilterChain(ServerHttpSecurity http) {
        return http
                .csrf(CsrfSpec::disable)
                .authorizeExchange(exchanges -> exchanges
                        .pathMatchers(PUBLIC_PATHS).permitAll()
                        .anyExchange().authenticated()
                )
                .oauth2ResourceServer(oauth2 -> oauth2.jwt(jwt -> {}))
                .build();
    }
}
```

### application.yml additions

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: ${hermes.security.jwt.issuer-uri:}
          public-key-location: ${hermes.security.jwt.public-key-location:}
```

### application-local.yml additions

```yaml
hermes:
  security:
    jwt:
      public-key-location: classpath:jwt/local-public.pem
```

### Local RSA key generation (one-time setup, not committed)

```bash
# Generate private key (keep locally — sign test tokens with this)
openssl genrsa -out local-private.pem 2048

# Extract public key (commit this to src/main/resources/jwt/)
openssl rsa -in local-private.pem -pubout -out local-public.pem
```

### Unit test reference

```java
@WebFluxTest(controllers = {FallbackController.class})
@Import(SecurityConfig.class)
class SecurityConfigTest {

    @Autowired
    private WebTestClient webTestClient;

    @Test
    void healthEndpoint_isPublic() {
        webTestClient.get().uri("/actuator/health")
                .exchange()
                .expectStatus().isNotFound(); // not 401 — security permits it, route just doesn't exist in slice
    }

    @Test
    @WithMockUser
    void authenticatedRequest_isPermitted() {
        webTestClient.get().uri("/fallback")
                .exchange()
                .expectStatus().isEqualTo(HttpStatus.SERVICE_UNAVAILABLE); // fallback returns 503, not 401
    }

    @Test
    void unauthenticatedRequest_returns401() {
        webTestClient.get().uri("/api/echo/ping")
                .exchange()
                .expectStatus().isUnauthorized();
    }
}
```

## Expected files

```text
hermes-gateway/
├── pom.xml                                            ← add oauth2-resource-server
└── src/
    ├── main/
    │   ├── java/dev/eithel/hermes/gateway/config/
    │   │   └── SecurityConfig.java
    │   └── resources/
    │       ├── application.yml                        ← add spring.security.oauth2 block
    │       ├── application-local.yml                  ← add local public key location
    │       └── jwt/
    │           └── local-public.pem                   ← committed RSA public key (local only)
    └── test/java/dev/eithel/hermes/gateway/config/
        └── SecurityConfigTest.java
```

## Acceptance criteria

- [ ] `spring-boot-starter-oauth2-resource-server` is declared in `hermes-gateway/pom.xml`.
- [ ] `SecurityConfig` is annotated with `@EnableWebFluxSecurity` (not `@EnableWebSecurity`).
- [ ] `GET /actuator/health` returns `200` without any `Authorization` header.
- [ ] `GET /api/echo/ping` without credentials returns `401 Unauthorized`.
- [ ] `GET /api/echo/ping` with a valid JWT Bearer token passes the security layer (downstream handling separate).
- [ ] CSRF is disabled.
- [ ] `SecurityConfigTest` passes with `.\mvnw.cmd test -pl hermes-gateway --also-make`.

## Validation

```powershell
.\mvnw.cmd test -pl hermes-gateway --also-make -Dtest=SecurityConfigTest
```

Manual test (unauthenticated):

```bash
curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/api/echo/ping
```

Expected: `401`.

Manual test (health — no auth needed):

```bash
curl -s http://localhost:8080/actuator/health
```

Expected: `{"status":"UP"}`.

## Commit suggestion

```text
feat: configure OAuth2 JWT resource server with WebFlux security chain
```

## Notes

- **`issuer-uri` vs `public-key-location`**: `issuer-uri` causes Spring Security to fetch the IdP's JWKS endpoint at startup to discover the public keys — requires a running IdP. `public-key-location` is used locally with a static RSA public key, bypassing JWKS discovery. For production, always use `issuer-uri` with the real IdP. The `${...:-}` default ensures the property is optional: whichever is set takes precedence.
- **`jwt -> {}`** (empty lambda) vs `Customizer.withDefaults()`: both are equivalent — they tell Spring Security to use auto-configured JWT settings. The empty lambda is more explicit and avoids importing `Customizer`.
- **`@WebFluxTest` scope**: `@WebFluxTest` loads only the web layer (controllers, filters, `@ControllerAdvice`). `SecurityConfig` must be imported explicitly with `@Import(SecurityConfig.class)` — it is not auto-loaded in a test slice. A `ReactiveJwtDecoder` bean will be required by the security auto-configuration; mock it with `@MockBean` in the test.
- **Order of `SecurityWebFilterChain` vs `GlobalFilter`**: Spring Security's `WebFilter` runs at order `WebFilterChainProxy.WEB_FILTER_CHAIN_FILTER_ORDER` (`-100`). Custom `GlobalFilter` beans with `Ordered.HIGHEST_PRECEDENCE + N` run before the security chain. Ensure that `CorrelationIdFilter` and `RequestLoggingFilter` (orders `+1` and `+2`) run before Spring Security so the correlation ID is available in the security context if needed. Alternatively, rely on the trace ID set in TASK-004.
