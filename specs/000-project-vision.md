# Project Vision — Hermes API Platform

## Status

Active

## Problem

Backend teams that expose APIs independently end up with a fragmented landscape: each service manages its own authentication, rate limiting, and observability. New consumers must integrate directly with each service, learning different auth schemes, absorbing cascading failures when a single service degrades, and having no unified view of traffic or errors.

There is no single answer to "who is calling our APIs, at what rate, and what is failing right now?"

## Solution

Hermes is a single, reactive API gateway that acts as the only entry point for all downstream services. It centralizes:

- **Authentication**: OAuth2 JWT Bearer tokens and API key validation — resolved once at the gateway, not in every service.
- **Rate limiting**: per-client token bucket enforced across all gateway instances via Redis — fair access without any service implementing its own limit.
- **Resilience**: circuit breakers that absorb downstream failure at the edge — consumers receive a fast, meaningful error instead of a timeout cascade.
- **Observability**: every request produces a trace, a metric, and a log line — the operations team gets full visibility without touching downstream services.
- **Multi-tenancy**: tenant identity resolved at the gateway from headers or credentials, propagated to downstream services — they never need to re-authenticate the caller.

Hermes is built as a **portfolio project** — it demonstrates enterprise-grade API gateway design using the Spring ecosystem, from a clean Maven multi-module structure through to AWS production deployment.

## Target users

| User | Goal |
|---|---|
| API consumer (external or internal team) | Call downstream services through a single stable endpoint with consistent auth and predictable rate limits |
| Platform engineer | Configure routes, tenants, and API keys via the Admin API without restarting the gateway |
| Operations engineer | Observe request rates, error rates, latency, and circuit breaker state in Prometheus/Grafana without reading raw logs |
| Security engineer | Enforce authentication and rate limiting globally without changes to downstream services |

## Key capabilities

| Capability | Mechanism | Milestone |
|---|---|---|
| HTTP routing | Spring Cloud Gateway `RouteLocator` + YAML routes | v0.2.0 |
| Correlation ID propagation | `CorrelationIdFilter` — `X-Correlation-ID` on every request | v0.2.0 |
| Request logging | `RequestLoggingFilter` — method, URI, status, correlation ID | v0.2.0 |
| Rate limiting | Redis-backed token bucket — `RequestRateLimiter` filter | v0.3.0 |
| Circuit breaker | Resilience4j reactive circuit breaker with fallback | v0.3.0 |
| Admin API | REST CRUD for routes, tenants, API keys — `hermes-admin` service | v0.4.0 |
| OAuth2 authentication | JWT Bearer token validation — Spring Security WebFlux | v0.5.0 |
| API key authentication | `X-API-Key` validation via Redis hash lookup | v0.5.0 |
| Prometheus metrics | Micrometer counters, histograms — `/actuator/prometheus` | v0.5.0 |
| Distributed tracing | OpenTelemetry — W3C `traceparent` propagation, trace ↔ correlation ID | v0.5.0 |
| Container packaging | Multi-stage Docker images — Spring Boot layer extraction | v1.0.0 |
| CI/CD pipeline | GitHub Actions — build, test, OWASP scan, ECR push, EKS deploy | v1.0.0 |
| AWS infrastructure | Terraform — EKS, RDS, ElastiCache, ALB, WAF, IRSA | v1.0.0 |

## Non-negotiable design principles

These principles are enforced in every spec and every code review. Violating them is a blocker, not a suggestion.

### 1. Reactive throughout
`hermes-gateway` is built on Spring Cloud Gateway WebFlux. The event loop must never be blocked. No `.block()`, no `Thread.sleep()`, no blocking I/O. All operations return `Mono<T>` or `Flux<T>`.

### 2. Security boundary: gateway does not touch the database
`hermes-gateway` holds no database credentials and makes no direct DB connections. Only `hermes-admin` owns the PostgreSQL datasource. This boundary is enforced structurally — the gateway module has no JPA or JDBC dependency.

### 3. Observable by default
Every request produces: a correlation ID (in the response header), a log line (method + URI + status + correlation ID), a Prometheus metric (via Micrometer auto-instrumentation), and a distributed trace span (via OpenTelemetry). Observability is not opt-in.

### 4. Spec-Driven Development (SDD)
Every feature starts as a spec document in `specs/`. No implementation begins without an accepted spec that defines scope, acceptance criteria, and technical notes. Specs are versioned alongside the code.

### 5. No long-lived credentials
All AWS access uses OIDC federation (GitHub Actions) or IRSA (EKS pods). No `AWS_ACCESS_KEY_ID` or `AWS_SECRET_ACCESS_KEY` is stored anywhere in the repository or CI secrets.

## Milestone roadmap

| Version | Milestone | Key deliverable |
|---|---|---|
| v0.1.0 | Project bootstrap | Parent POM, Maven Wrapper, `hermes-common`, `hermes-gateway`, health endpoint |
| v0.2.0 | Gateway MVP | Static + programmatic routes, Correlation ID filter, Request logging filter |
| v0.3.0 | Resilience & rate limiting | Redis token bucket, Resilience4j circuit breaker, `hermes-test` Testcontainers suite |
| v0.4.0 | Admin API | `hermes-admin` Spring MVC service, Route/Tenant/API-key CRUD, PostgreSQL + Flyway |
| v0.5.0 | Security & observability | OAuth2 JWT resource server, API key auth via Redis, Prometheus, OpenTelemetry |
| v1.0.0 | Production ready | Docker images, GitHub Actions CI/CD, Terraform AWS baseline, EKS deployment |

## Out of scope (for v1.0.0)

- GraphQL or gRPC gateway (HTTP/REST only in v1.0.0).
- Multi-region active-active deployment.
- Service mesh integration (Istio, Linkerd).
- Dynamic plugin system for third-party filter extensions.
- Per-route SLA definitions and tail-based sampling policies.
