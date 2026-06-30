# ADR-002 — Use Redis as the rate limiter backend

## Status

Accepted

## Date

2024-01-01

## Context

The Hermes gateway requires a rate limiter that:

- Enforces per-client request limits consistently across **all running gateway instances** (a single client must not be able to exceed the limit by distributing requests across pods).
- Operates on the **hot path** of every request — latency budget for the rate limit check is under 5 ms.
- Uses a **token bucket** algorithm to allow short bursts above the sustained rate without permanently blocking a client.
- Fails safely: if the rate limiter backend is unavailable, the gateway should degrade gracefully (fail open — allow traffic) rather than blocking all requests.
- Does not require a dedicated new infrastructure component in the stack.

The following backends were evaluated:

| Option | Consistency | Latency | New infra? | Notes |
|---|---|---|---|---|
| **In-process (ConcurrentHashMap)** | Per-instance only | Sub-ms | No | Each pod has its own counter — a client with N pods gets N× the limit |
| **PostgreSQL** | Globally consistent | 5–20 ms | Yes (already planned for admin) | Too slow for the hot path; row-level locks add contention |
| **Redis** | Globally consistent | < 1 ms | Yes (but dual-use) | Atomic Lua script; already required for API key validation in v0.5.0 |
| **Redis Cluster** | Globally consistent | < 1 ms | Yes (higher complexity) | Needed only at very high scale; single-shard Redis sufficient for v1.0.0 |

## Decision

**Use Redis** as the shared backend for the gateway rate limiter, via Spring Cloud Gateway's built-in `RequestRateLimiter` filter and its Lua-script token bucket implementation.

## Rationale

1. **Global consistency**: Redis is a single shared store. All gateway pods read and write the same token bucket counters. A client cannot bypass the rate limit by distributing requests across instances.

2. **Atomic token bucket via Lua**: Spring Cloud Gateway implements the token bucket algorithm as a Redis Lua script (`request_rate_limiter.lua`). Lua scripts execute atomically on the Redis server — no race conditions, no distributed lock needed.

3. **Sub-millisecond latency**: Redis round-trip on a local network (including ElastiCache in the same VPC/AZ) is under 1 ms. This is within the latency budget for every gateway request.

4. **Already in the stack**: Redis is introduced in v0.3.0 for rate limiting. In v0.5.0, the same Redis instance is reused for API key hash validation. Adding Redis once justifies two use cases — it is not a single-purpose dependency.

5. **Native Spring integration**: `spring-cloud-starter-gateway` ships with the `RequestRateLimiter` filter that requires only a `RedisRateLimiter` bean and a `KeyResolver` bean. No custom rate limiter implementation is needed.

## Consequences

### Positive

- Rate limits are globally enforced with no coordination overhead between gateway pods.
- The token bucket algorithm provides a better client experience than fixed-window rate limiting — short bursts are absorbed without triggering the limit.
- `replenishRate` and `burstCapacity` are externalizable via Spring properties, enabling per-environment tuning without code changes.

### Negative

- **Redis availability is a dependency on the hot path**: if Redis is unavailable, rate limiting fails. Spring Cloud Gateway's `RequestRateLimiter` defaults to **denying** requests when Redis is down (`deny-empty-key=true`). This must be evaluated per deployment:
  - For public-facing APIs: fail open (allow traffic, log the unavailability) is safer — a Redis outage should not take down the entire gateway.
  - For abuse-sensitive APIs: fail closed may be acceptable.
  - **Recommendation**: configure `deny-empty-key=false` and monitor Redis availability as a separate alert.
- **Redis is a new operational dependency**: the team must operate Redis in production (ElastiCache in HA mode — see infrastructure specs in milestone 006). Redis failure modes, memory limits, and eviction policies must be understood.
- **Single-shard limit**: the Lua script implementation does not support Redis Cluster keyslots — all rate limit keys must live in the same Redis slot. This is not a concern until very high key cardinality is reached (millions of unique `KeyResolver` values). Redis Cluster support, if needed, is a v2.0.0 concern.

## Alternatives considered

### In-process rate limiting (ConcurrentHashMap or Guava RateLimiter)

Rejected because per-instance counters do not enforce global limits in a multi-pod deployment. A client can send N times the allowed rate by distributing requests across N gateway pods. This is not acceptable for public-facing rate limiting.

### PostgreSQL

Rejected because row-level locking for atomic counter updates adds 5–20 ms of latency on the request hot path. This is unacceptable for a rate limiting check that must complete before routing begins. PostgreSQL is appropriate for `hermes-admin` (low-frequency CRUD) but not for per-request gateway operations.

### Dedicated rate limiting service (e.g., Envoy, custom microservice)

Rejected because it adds a network hop and a new service to operate and monitor. The value of a dedicated service is justified only at very large scale (global rate limiting across multiple datacenters) — not for v1.0.0.
