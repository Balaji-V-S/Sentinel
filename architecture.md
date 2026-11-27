# Sentinel — Architecture

**A self-hosted API Gateway and Service Registry for Java-based distributed systems.**

Sentinel is a centralized entry point for backend services, providing service registration, discovery, request routing, traffic control, and distributed rate limiting.

Built with Java and Spring Boot, Sentinel aims to simplify the infrastructure required to operate distributed backend applications without requiring a heavyweight orchestration platform.

> Register your services. Route your traffic. Control your infrastructure.

---

## 1. High-Level Architecture

Sentinel is a standalone server with these major components:

- **Gateway Runtime:** HTTP server, reverse proxy, request lifecycle, filter chain, connection management, graceful shutdown.
- **Routing Engine:** Path/method matching, route predicates, path rewriting, header manipulation, logical service resolution.
- **Service Registry:** Service and instance registration, metadata, heartbeat tracking, lease expiration, lifecycle, deregistration.
- **Service Discovery:** Service lookup, healthy-instance filtering, registry queries, instance selection.
- **Load Balancer:** Round Robin, Random, Weighted Round Robin; later Least Connections and Consistent Hashing.
- **Traffic Control:** Rate-limiting algorithms and policies by IP, API key, user, route, tenant.
- **Resilience Engine:** Circuit breakers, retries, timeouts, bulkheads, connection limits, failure handling.
- **Security Engine:** API-key authentication, request validation, authorization policies, IP allow/deny lists.
- **Management Plane:** Registry administration, route management, configuration inspection, runtime health and admin APIs.
- **Observability:** Structured logs, request IDs, metrics, health checks, traces, gateway latency and instance health.

These are logical modules within one Spring Boot application initially—not independently deployed microservices.

---

## 2. Service Registration and Discovery

Backend service instances register themselves with Sentinel at startup. They periodically renew a heartbeat lease. Sentinel maintains a registry of instances and makes healthy instances eligible for routing.

### Registration sequence

1. Service starts.
2. Service calls Sentinel's registration endpoint.
3. Sentinel records the instance and returns an instance ID and lease.
4. Service periodically sends heartbeats.
5. Sentinel renews the lease.
6. If the heartbeat expires, the instance is marked unhealthy and removed from routing eligibility.
7. The service may explicitly deregister during graceful shutdown.

### Instance lifecycle (v1)

`UP → DOWN → REMOVED`

- **UP:** Eligible for routing.
- **DOWN:** Heartbeat lease expired or health check failed.
- **REMOVED:** Explicitly deregistered or removed after retention/cleanup.

### Registration data example

```json
{
  "serviceId": "user-service",
  "instanceId": "user-service-01",
  "address": "192.168.1.10",
  "port": 8081,
  "scheme": "http",
  "healthCheckUrl": "/actuator/health",
  "metadata": {
    "version": "1.0.0",
    "environment": "development"
  }
}
```

A logical service can have multiple instances:

```text
user-service
├── user-service-01 → 192.168.1.10:8081 → UP
├── user-service-02 → 192.168.1.11:8081 → UP
└── user-service-03 → 192.168.1.12:8081 → DOWN
```

---

## 3. Registry vs. Subscription

Registration, heartbeat, and discovery are the initial capabilities. A registry subscription/watch is a future capability.

| Mechanism | Purpose | v1 |
|---|---|---|
| Registration | Service announces its instance | Yes |
| Heartbeat | Service renews its lease | Yes |
| Discovery query | Client requests available instances | Yes |
| Subscription / Watch | Client receives registry changes continuously | Later |

Possible subscription mechanisms later: Server-Sent Events (SSE), WebSockets, long polling, or a client SDK with caching and periodic refresh.

The gateway should use the registry internally. External services may query discovery when needed. Push subscriptions are not required for the first release.

---

## 4. Request Processing Pipeline

Every incoming request passes through a defined pipeline:

1. Request ID / context creation
2. Route resolution
3. Authentication
4. Authorization
5. Rate limiting
6. Service discovery
7. Load balancing
8. Circuit breaker
9. Request transformation
10. Proxy to backend
11. Response transformation
12. Return response

Failure responses include:

| Status | Scenario |
|---|---|
| 404 | No matching route |
| 401 | Unauthenticated |
| 403 | Forbidden |
| 429 | Rate limit exceeded |
| 503 | No healthy instances available or circuit open |
| 504 | Backend timeout |

The internal filter pipeline should be extensible. Avoid implementing a complex public plugin API initially.

Example filter contract:

```java
public interface GatewayFilter {
    Mono<GatewayResponse> filter(
        GatewayRequest request,
        GatewayFilterChain chain
    );
}
```

---

## 5. Configuration

Keep routes and server policies externalized.

```yaml
sentinel:
  server:
    port: 8080

  registry:
    heartbeat-interval: 10s
    lease-duration: 30s
    cleanup-interval: 5s

  gateway:
    connect-timeout: 3s
    response-timeout: 30s

  routing:
    strategy: round-robin

  rate-limiting:
    enabled: true
    default-algorithm: token-bucket

routes:
  - id: users
    path: /api/users/**
    serviceId: user-service

  - id: orders
    path: /api/orders/**
    serviceId: order-service

  - id: payments
    path: /api/payments/**
    serviceId: payment-service
```

Routes point to logical service IDs. The registry supplies actual addresses dynamically.

---

## 6. Internal Module Structure

Use a Maven multi-module project, deployed as one executable server.

```text
sentinel/
├── pom.xml
├── sentinel-server/
│   └── SentinelApplication.java
├── sentinel-core/
│   ├── gateway/
│   ├── routing/
│   ├── request/
│   └── filter/
├── sentinel-registry/
│   ├── registration/
│   ├── discovery/
│   ├── heartbeat/
│   ├── lifecycle/
│   └── model/
├── sentinel-loadbalancer/
│   ├── strategy/
│   └── selector/
├── sentinel-ratelimiter/
│   ├── algorithm/
│   ├── policy/
│   └── storage/
├── sentinel-resilience/
│   ├── circuitbreaker/
│   ├── retry/
│   └── timeout/
├── sentinel-security/
├── sentinel-observability/
├── sentinel-admin/
├── docs/
│   ├── architecture/
│   ├── api/
│   └── adr/
└── deployment/
    ├── docker/
    └── compose/
```

### Dependency rules

- `sentinel-server` is the composition root and wires modules together.
- `sentinel-core` consumes registry, load-balancer, rate-limit, security, and resilience abstractions.
- Registry, load balancer, rate limiter, security, and resilience modules should depend on shared contracts rather than one another unnecessarily.
- Avoid circular dependencies.

---

## 7. Deployment Evolution

### v1 — Single instance

One Sentinel instance runs gateway, registry, and policies. The registry is in-memory. Backend services register and renew leases. No database is required for the initial registry.

```text
┌──────────────────────────────────────────┐
│                 Sentinel                │
│       Gateway + Registry + Policies    │
└────────────────────┬─────────────────────┘
                     │
              ┌──────┼──────┐
              ▼      ▼      ▼
            User   Order  Payment
            API    API    API
```

### v2 — Distributed Sentinel (future)

```text
              Load Balancer
                    │
          ┌─────────┴─────────┐
          ▼                   ▼
    Sentinel Node 1     Sentinel Node 2
          │                   │
          └─────────┬─────────┘
                    │
              Shared State
              ┌─────┴─────┐
              ▼           ▼
            Redis      PostgreSQL
```

Distributed deployment introduces registry synchronization, distributed lease management, shared rate-limit state, configuration synchronization, node health, split-brain behavior, and gateway high availability.

Do not implement this before the single-node version is solid.

---

## 8. Technology Stack

| Layer | Technology |
|---|---|
| Language | Java |
| Runtime | Java 21+ |
| Framework | Spring Boot |
| HTTP Engine | Reactor Netty |
| Gateway | Custom reactive gateway engine |
| Registry | In-memory initially |
| Distributed State | Redis (later) |
| Persistence | PostgreSQL (future/configuration persistence) |
| Metrics | Micrometer |
| Monitoring | Prometheus |
| Visualization | Grafana |
| Containerization | Docker |
| Testing | JUnit 5, Testcontainers |
| Load Testing | k6 |
| CI/CD | GitHub Actions |

---

## 9. Architecture Principles

- **Self-hosted:** Operate Sentinel within your own infrastructure.
- **Configuration-first:** Configure routes and policies without modifying application code.
- **Service-oriented:** Discover backend instances by logical service ID.
- **Stateless gateway:** Keep request processing stateless where possible.
- **Dynamic discovery:** Avoid hardcoded backend instance addresses.
- **Extensible:** Keep routing, rate-limiting algorithms, and load-balancing strategies pluggable.
- **Observable:** Expose metrics, logs, health information, and operational visibility.
- **Resilient:** Treat downstream failures as expected scenarios.
- **Incremental:** Start with a single server and evolve only when requirements justify it.

---

## 10. Development Roadmap

### Phase 1 — Core Gateway

- [ ] Spring Boot bootstrap
- [ ] Reactive HTTP server
- [ ] Reverse proxy implementation
- [ ] Static route configuration
- [ ] Request forwarding
- [ ] Header propagation
- [ ] Configurable timeouts
- [ ] Graceful shutdown

### Phase 2 — Service Registry

- [ ] Registration API
- [ ] Instance identifiers
- [ ] Instance metadata
- [ ] Heartbeat API
- [ ] Lease expiration
- [ ] Instance lifecycle
- [ ] Deregistration
- [ ] Discovery API
- [ ] In-memory registry

### Phase 3 — Dynamic Routing & Load Balancing

- [ ] Route resolution by service ID
- [ ] Dynamic instance resolution
- [ ] Round Robin
- [ ] Random
- [ ] Weighted load balancing
- [ ] Instance exclusion
- [ ] Routing filters

### Phase 4 — Rate Limiting

- [ ] Rate-limiting abstraction
- [ ] Fixed Window
- [ ] Token Bucket
- [ ] Sliding Window Counter
- [ ] Redis-backed implementation
- [ ] Atomic operations
- [ ] Per-route and per-client policies
- [ ] Rate-limit response headers

### Phase 5 — Resilience

- [ ] Circuit breaker
- [ ] Retry policies
- [ ] Retry budgets
- [ ] Connection pooling
- [ ] Bulkheads
- [ ] Failure isolation
- [ ] Fail-open / fail-closed policies

### Phase 6 — Observability

- [ ] Health endpoints
- [ ] Structured logging
- [ ] Request correlation IDs
- [ ] Micrometer metrics
- [ ] Prometheus integration
- [ ] Gateway latency metrics
- [ ] Registry metrics
- [ ] Rate-limit metrics

### Phase 7 — Production Deployment

- [ ] Docker image
- [ ] Docker Compose
- [ ] Externalized configuration
- [ ] Configuration validation
- [ ] Integration tests
- [ ] Load testing
- [ ] Security review
- [ ] Deployment documentation

---

## 11. First Milestone: Sentinel v0.1

**Service Registry + Dynamic API Gateway**

- Spring Boot server and configuration
- Service registration REST API
- Heartbeats and lease expiration
- In-memory service registry
- Registry lookup API
- Path-based gateway routing
- Dynamic service-instance resolution
- Round-robin load balancing
- Reverse proxying to backend services
- Docker Compose demonstration with two instances of one service

### Demonstration

Run two instances of a backend service, register both with Sentinel, and send 100 requests through the gateway.

Demonstrate traffic distribution.

Stop one instance and verify that Sentinel stops routing to it after its lease expires.

---

## 12. Key Architectural Decision

The **service registry must be independent of the gateway**.

The gateway consumes registry information; the registry should not depend on routing. This separation improves testability and makes future evolution easier.

Sentinel begins as a single modular server. Distributed deployment, registry subscriptions, Redis-backed shared state, dashboards, and advanced policy management are future capabilities—not prerequisites for v0.1.

---

> **Sentinel — Know your services. Control your traffic.**
