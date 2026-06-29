# TASK-004 — Integration tests (hermes-test module)

## Status

Planned

## Parent story

`US-003-resilience-and-rate-limiting.md`

## Summary

Create the `hermes-test` Maven module and implement integration tests that verify the rate limiter and circuit breaker work end-to-end with real infrastructure. Redis runs in a Testcontainers container; WireMock stubs the downstream service. Tests are written with `WebTestClient` against a full `@SpringBootTest` context.

## Technical scope

This task includes:

- `hermes-test` Maven module added to the parent POM.
- `hermes-test/pom.xml` with Testcontainers, WireMock, and Spring Boot test dependencies.
- `application-test.yml` (test resources) overriding downstream URI and Redis connection.
- `RateLimitingIntegrationTest`: verifies that requests above burst capacity receive HTTP 429.
- `CircuitBreakerIntegrationTest`: verifies that sustained downstream 5xx failures open the circuit and return HTTP 503 via the fallback.
- `RoutingIntegrationTest`: baseline test verifying normal proxying returns HTTP 200 with `X-Correlation-ID` in the response.

## Out of scope

This task does not include:

- Contract tests (Pact) — deferred to a dedicated contract testing story.
- Load or performance tests.
- Tests for the `hermes-admin` module (not yet implemented).
- Any test that requires a real external service (all downstream calls are stubbed by WireMock).

## Implementation notes

- **Module name**: `hermes-test`
- **artifactId**: `hermes-test`
- **packaging**: `jar` — test module, not a runnable application. `spring-boot-maven-plugin` must NOT be added.
- **Parent**: inherits `dev.eithel:hermes-api-platform:0.1.0-SNAPSHOT`.
- **Add to parent POM `<modules>`**: `<module>hermes-test</module>`.
- **Key dependencies** (all in `test` scope or default scope since this is a test-only module):
  - `hermes-gateway` (project dependency — brings the application class and all configuration).
  - `spring-boot-starter-test` (JUnit 5, AssertJ, Mockito).
  - `reactor-test` (StepVerifier).
  - `org.testcontainers:testcontainers`.
  - `org.testcontainers:junit-jupiter`.
  - `org.wiremock.integrations:wiremock-spring-boot` or `com.github.tomakehurst:wiremock-jre8-standalone`.
- **Testcontainers BOM**: add to parent POM's `<dependencyManagement>`:
  ```xml
  <dependency>
      <groupId>org.testcontainers</groupId>
      <artifactId>testcontainers-bom</artifactId>
      <version>${testcontainers.version}</version>
      <type>pom</type>
      <scope>import</scope>
  </dependency>
  ```
- **Redis container**: `GenericContainer<>("redis:7-alpine").withExposedPorts(6379)`.
- **`@DynamicPropertySource`**: override `spring.data.redis.host` and `spring.data.redis.port` with the Testcontainers-mapped values.
- **WireMock**: started as a static field with `@RegisterExtension WireMockExtension`. Override `hermes.downstream.echo-uri` with WireMock's base URL via `@DynamicPropertySource`.
- **`@SpringBootTest`**: `webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT` + inject `WebTestClient`.
- **Circuit breaker test strategy**: send `slidingWindowSize` (5) requests that WireMock returns as 500, then assert the next request returns 503 (fallback). Add a brief `Awaitility` wait if needed for the state machine to process.

### hermes-test/pom.xml reference

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project>
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>dev.eithel</groupId>
        <artifactId>hermes-api-platform</artifactId>
        <version>0.1.0-SNAPSHOT</version>
    </parent>

    <artifactId>hermes-test</artifactId>
    <name>Hermes :: Integration Tests</name>

    <dependencies>
        <dependency>
            <groupId>dev.eithel</groupId>
            <artifactId>hermes-gateway</artifactId>
            <version>${project.version}</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>io.projectreactor</groupId>
            <artifactId>reactor-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.testcontainers</groupId>
            <artifactId>testcontainers</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.testcontainers</groupId>
            <artifactId>junit-jupiter</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.wiremock</groupId>
            <artifactId>wiremock-standalone</artifactId>
            <version>${wiremock.version}</version>
            <scope>test</scope>
        </dependency>
    </dependencies>
</project>
```

### Base integration test class reference

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
@ActiveProfiles("test")
abstract class BaseIntegrationTest {

    @Container
    static final GenericContainer<?> REDIS = new GenericContainer<>("redis:7-alpine")
            .withExposedPorts(6379);

    @RegisterExtension
    static WireMockExtension WIREMOCK = WireMockExtension.newInstance()
            .options(wireMockConfig().dynamicPort())
            .build();

    @Autowired
    protected WebTestClient webTestClient;

    @DynamicPropertySource
    static void overrideProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.data.redis.host", REDIS::getHost);
        registry.add("spring.data.redis.port", () -> REDIS.getMappedPort(6379));
        registry.add("hermes.downstream.echo-uri", WIREMOCK::baseUrl);
    }
}
```

### RoutingIntegrationTest reference

```java
class RoutingIntegrationTest extends BaseIntegrationTest {

    @Test
    void request_isProxiedToDownstream_with200() {
        WIREMOCK.stubFor(get(urlPathEqualTo("/echo/ping"))
                .willReturn(ok().withBody("pong")));

        webTestClient.get().uri("/api/echo/ping")
                .exchange()
                .expectStatus().isOk()
                .expectHeader().exists("X-Correlation-ID");
    }
}
```

### RateLimitingIntegrationTest reference

```java
class RateLimitingIntegrationTest extends BaseIntegrationTest {

    @Test
    void requestsAboveBurstCapacity_receive429() {
        WIREMOCK.stubFor(get(urlPathEqualTo("/echo/ping")).willReturn(ok()));

        // burstCapacity=5 in test profile — send 6 requests rapidly
        List<ResponseSpec> responses = IntStream.range(0, 6)
                .mapToObj(i -> webTestClient.get().uri("/api/echo/ping").exchange())
                .toList();

        long tooManyRequests = responses.stream()
                .filter(r -> r.returnResult(String.class).getStatus() == HttpStatus.TOO_MANY_REQUESTS)
                .count();

        assertThat(tooManyRequests).isGreaterThanOrEqualTo(1);
    }
}
```

### CircuitBreakerIntegrationTest reference

```java
class CircuitBreakerIntegrationTest extends BaseIntegrationTest {

    @Test
    void circuitBreaker_openAfterThreshold_returnsFallback() {
        // Stub downstream to always return 500
        WIREMOCK.stubFor(get(urlPathEqualTo("/echo/ping"))
                .willReturn(serverError()));

        // Send slidingWindowSize (5) requests to trigger the circuit
        for (int i = 0; i < 5; i++) {
            webTestClient.get().uri("/api/echo/ping").exchange();
        }

        // Circuit should now be open — next request must hit fallback
        webTestClient.get().uri("/api/echo/ping")
                .exchange()
                .expectStatus().isEqualTo(HttpStatus.SERVICE_UNAVAILABLE)
                .expectBody()
                .jsonPath("$.status").isEqualTo(503)
                .jsonPath("$.error").isEqualTo("Service Unavailable");
    }
}
```

## Expected files

```text
hermes-api-platform/
└── pom.xml                                          ← add hermes-test module + Testcontainers BOM

hermes-test/
├── pom.xml
└── src/test/
    ├── java/dev/eithel/hermes/test/
    │   ├── BaseIntegrationTest.java
    │   ├── RoutingIntegrationTest.java
    │   ├── RateLimitingIntegrationTest.java
    │   └── CircuitBreakerIntegrationTest.java
    └── resources/
        └── application-test.yml
```

### application-test.yml

```yaml
hermes:
  ratelimit:
    replenish-rate: 2
    burst-capacity: 5

spring:
  cloud:
    gateway:
      routes:
        - id: echo-route-yaml
          uri: ${hermes.downstream.echo-uri}
          predicates:
            - Path=/api/echo/**
          filters:
            - StripPrefix=1
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 2
                redis-rate-limiter.burstCapacity: 5
                key-resolver: "#{@remoteAddrKeyResolver}"
            - name: CircuitBreaker
              args:
                name: echo-cb
                fallbackUri: forward:/fallback
```

## Acceptance criteria

- [ ] `hermes-test` is declared in the parent POM `<modules>`.
- [ ] `hermes-test/pom.xml` declares Testcontainers and WireMock dependencies.
- [ ] `BaseIntegrationTest` starts a Redis container and a WireMock server via `@DynamicPropertySource`.
- [ ] `RoutingIntegrationTest` passes: proxied request returns 200 with `X-Correlation-ID`.
- [ ] `RateLimitingIntegrationTest` passes: at least one request above burst capacity returns 429.
- [ ] `CircuitBreakerIntegrationTest` passes: after 5 upstream 500s, the gateway returns 503 with fallback body.
- [ ] `.\mvnw.cmd clean verify` passes (requires Docker daemon).

## Validation

```powershell
.\mvnw.cmd clean verify
```

Expected: `BUILD SUCCESS`, all integration tests green.

To run only integration tests:

```powershell
.\mvnw.cmd verify -pl hermes-test --also-make
```

## Commit suggestion

```text
test: add hermes-test module with rate limiting and circuit breaker integration tests
```

## Notes

- **Docker requirement at test time**: Testcontainers requires a running Docker daemon. CI (GitHub Actions `ubuntu-latest`) has Docker available. Local Windows development requires Docker Desktop running. If Docker is unavailable, tests are skipped gracefully when annotated with `@DisabledWithoutDocker` (Testcontainers 1.19+).
- **Circuit breaker timing**: Resilience4j's state machine is asynchronous. In a test, the circuit may not open synchronously after the 5th failure. Add `Awaitility.await().atMost(2, SECONDS).until(...)` around the assertion if flakiness is observed.
- **Container reuse**: Testcontainers' `reuse = true` (`.withReuse(true)`) can be enabled to share containers between test runs during development. It requires a `~/.testcontainers.properties` file with `testcontainers.reuse.enable=true`. Do not enable by default in the spec — it is a developer-local optimization.
- **hermes-test is a test-only module**: it has no `src/main/java` sources. Maven still compiles it as a `jar`, but the JAR contains only test classes. `spring-boot-maven-plugin` must NOT be applied — it would fail trying to find a `@SpringBootApplication` main class in this module.
