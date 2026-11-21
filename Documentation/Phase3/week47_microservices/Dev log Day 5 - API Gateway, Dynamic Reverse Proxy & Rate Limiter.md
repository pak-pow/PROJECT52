---
date: 2026-11-19
project: PulseMesh Microservices Architecture & Distributed Systems Platform
topic: Day 5 - API Gateway, Dynamic Reverse Proxy & Rate Limiter
Tags:
  - "[[Microservices]]"
  - "[[API-Gateway]]"
  - "[[Reverse-Proxy]]"
  - "[[Rate-Limiter]]"
  - "[[Token-Bucket]]"
  - "[[Dev Log]]"
---
# DEV LOG: WEEK 47 - DAY 5

**Core Objective:** Build the centralized ingress and traffic control layer of the PulseMesh platform by developing the API Gateway (`gateway/app.py`), integrating the dynamic path-prefix reverse proxy router (`gateway/proxy.py`) with hop-by-hop header sanitization, the sliding-window Token Bucket Rate Limiter (`gateway/rate_limiter.py`) with burst capacity and whitelisting, the aggregated cluster health probe (`GET /health`), and real-time gateway telemetry (`GET /api/v1/gateway/stats`) backed by 98 passing tests with 95.32% branch coverage and 0 linter/security warnings.

---

## 1. The Big Picture and Simple Explanation

In a microservices ecosystem, client applications should never communicate directly with dozens of internal microservice ports and IP addresses. Direct communication exposes internal network topology, complicates cross-cutting concerns (authentication, rate limiting, logging), and creates severe cross-origin resource sharing (CORS) friction:
1. **Unified Edge Ingress:** The API Gateway acts as the single reverse proxy facade (Port 8000). Clients send all requests to `/api/v1/*`, and the gateway dynamically maps route prefixes to internal services (`/api/v1/users` -> `user-service:8002`, `/api/v1/products` -> `catalog-service:8003`, etc.).
2. **Denial-of-Service & Burst Protection:** Without edge rate limiting, a single runaway client or bot can flood internal microservices, exhausting database connections and CPU. The **Token Bucket algorithm** allows client traffic bursts up to a configured bucket capacity while strictly enforcing a sustainable average replenishment rate.
3. **Hop-by-Hop Header Sanitization:** Proxies must strip connection-specific HTTP headers (`Transfer-Encoding`, `Connection`, `Keep-Alive`, `Upgrade`) before forwarding requests upstream to avoid chunked encoding desynchronization and protocol errors.
4. **Aggregated Cluster Health Probing:** Operations teams and cloud load balancers need a single vitality endpoint. The gateway queries all downstream service instances and aggregates global cluster health (`HEALTHY`, `DEGRADED`, or `DOWN`).

```
+-----------------------------------------------------------------------------------------+
|                       PulseMesh API Gateway Edge Ingress Layer                          |
|                                                                                         |
|       External HTTP Client / Mobile App                                                 |
|                     |                                                                   |
|                     v                                                                   |
|         +---------------------------------------------------------------+               |
|         |                PulseMesh API Gateway (Port 8000)              |               |
|         |                                                               |               |
|         |   1. Token Bucket Rate Limiter                                |               |
|         |      * Client IP / API Key Tracking                           |               |
|         |      * Tokens exhausted? -> HTTP 429 Too Many Requests        |               |
|         |                                                               |               |
|         |   2. Distributed Trace Injection                              |               |
|         |      * Injects or preserves X-Correlation-ID                  |               |
|         |                                                               |               |
|         |   3. Hop-by-Hop Header Sanitization                           |               |
|         |      * Strips Connection, Host, Transfer-Encoding             |               |
|         |                                                               |               |
|         |   4. Dynamic Reverse Proxy Dispatcher                         |               |
|         |      * Longest prefix matching                                |               |
|         |      * Dispatches via Resilient PulseClient                   |               |
|         +-------------------------------+-------------------------------+               |
|                                         |                                               |
|                    +--------------------+--------------------+                          |
|                    |                    |                    |                          |
|                    v                    v                    v                          |
|             /api/v1/users        /api/v1/products      /api/v1/orders                   |
|                    |                    |                    |                          |
|                    v                    v                    v                          |
|             +-------------+      +-------------+      +-------------+                   |
|             | User Service|      | Catalog Svc |      | Order Svc   |                   |
|             | (Port 8002) |      | (Port 8003) |      | (Port 8004) |                   |
|             +-------------+      +-------------+      +-------------+                   |
+-----------------------------------------------------------------------------------------+
```

---

## 2. Key Engineering Implementations

### Sliding Window Token Bucket Rate Limiter (`gateway/rate_limiter.py`)
- **Token Bucket Core:**
  - `TokenBucket` maintains burst capacity (`capacity`) and sustained refill rates (`refill_rate`).
  - Thread-safe token consumption using `threading.Lock`. Calculates exact `retry_after` seconds when tokens are exhausted.
- **Multi-Client Gateway Rate Limiter (`GatewayRateLimiter`):**
  - Maintains individual client buckets mapped to remote IP or client identification key.
  - Whitelist bypass for trusted infrastructure nodes (`127.0.0.1`, `localhost`).
  - Emits standards-compliant rate limit headers:
    - `X-RateLimit-Limit`: Maximum burst capacity.
    - `X-RateLimit-Remaining`: Available token balance.
    - `Retry-After`: Required cooldown duration in seconds when throttled (HTTP 429).
  - Telemetry tracking: active client count, total requests, rate-limited requests, and rejection percentage.

### Dynamic Reverse Proxy & Route Dispatcher (`gateway/proxy.py`)
- **Prefix Route Dispatch Table:**
  - Dynamically routes incoming paths to microservices based on prefix length precedence:
    - `/api/v1/users` -> `user-service`
    - `/api/v1/products` -> `catalog-service`
    - `/api/v1/inventory` -> `catalog-service`
    - `/api/v1/catalog` -> `catalog-service`
    - `/api/v1/orders` -> `order-service`
    - `/api/v1/notifications` -> `notification-service`
    - `/api/v1/registry` -> `registry-service`
  - Supports runtime dynamic route registration via `register_route(prefix, service_name)`.
- **Hop-by-Hop Header Sanitization:**
  - Automatically filters connection-specific headers (`Connection`, `Keep-Alive`, `Proxy-Authenticate`, `Transfer-Encoding`, `Upgrade`, `Content-Length`, `Host`) to maintain HTTP proxy compliance.
- **Fault-Tolerant Forwarding via `PulseClient`:**
  - Integrates service discovery resolution, load balancing across active instances, and circuit breaker protection.
  - Translates `CircuitBreakerOpenException` into HTTP 503 Service Unavailable with structured JSON detailing breaker cooldown.
  - Records per-service response latencies, request counts, and error rates.

### API Gateway Flask Application & Health Aggregator (`gateway/app.py`)
- **Aggregated Health Probe (`GET /health`):**
  - Evaluates cluster vitality across all target microservices (`user-service`, `catalog-service`, `order-service`, `notification-service`, `registry-service`).
  - Inspects instance count and health statuses (`HEALTHY`, `DEGRADED`, `DOWN`).
  - Calculates global cluster status:
    - `HEALTHY` (HTTP 200): All registered services operational.
    - `DEGRADED` (HTTP 200): Some services or replica nodes degraded, but partial traffic viable.
    - `DOWN` (HTTP 503): Critical service failure or zero healthy nodes.
- **Dynamic Reverse Proxy Catch-All (`/api/v1/<path:subpath>`):**
  - Intercepts all inbound HTTP methods (`GET`, `POST`, `PUT`, `DELETE`, `PATCH`).
  - Extracts or generates `X-Correlation-ID` distributed tracing headers.
  - Evaluates rate limiting: returns HTTP 429 Too Many Requests if bucket is depleted.
  - Forwards payload and query parameters, returning upstream status codes and payload data.
- **Gateway Telemetry API (`GET /api/v1/gateway/stats`):**
  - Real-time telemetry exposing rate limiter utilization, route counts, average latencies, and circuit breaker states.

---

## 3. Verification & Quality Gates Report

All 5 automated enterprise quality gates passed with zero warnings or violations:

```text
======================================================================
  PULSEMESH MICROSERVICES: 5-STAGE AUTOMATED QUALITY GATES
======================================================================
  * FLAKE8 LINTER        : [OK] PASSED (1.30s)
  * BLACK FORMATTER      : [OK] PASSED (0.46s)
  * ISORT IMPORTS        : [OK] PASSED (0.39s)
  * BANDIT SECURITY      : [OK] PASSED (0.78s)
  * PYTEST COVERAGE      : [OK] PASSED (3.92s)
======================================================================
>> ALL QUALITY GATES PASSED! Safe to commit and deploy.
```

### Test Suite Metrics
- Total Tests: **98 passed** in 2.80s across 12 test suites.
- Branch Coverage: **95.32%** (2096 statements, 66 missed; 428 branches, 48 partial). Exceeds the 90.0% requirement.
- Zero-Emoji Compliance: 100% verified across all source code, tests, documentation, and commit messages.

---

## 4. Key Engineering Insights & Decisions

1. **Centralized Edge Ingress:**
   The API Gateway acts as the single public interface, hiding the internal microservice network topology and IP allocations. Internal services communicate privately over their assigned port maps without requiring public routing.
2. **Upstream Circuit Breaker Decoupling:**
   Integrating `PulseClient` inside `ReverseProxyRouter` ensures that cascading backend outages are arrested at the gateway edge. When downstream services fail repeatedly, the breaker trips to `OPEN`, immediately returning HTTP 503 without saturating backend network sockets.
3. **Hop-by-Hop Header Sanitization:**
   Transparently stripping hop-by-hop headers (`Connection`, `Transfer-Encoding`, `Keep-Alive`) prevents protocol violations and chunked encoding desynchronization when proxying requests through Flask and Requests.

---

## 5. Next Steps

With the API Gateway reverse proxy and traffic control engine operational, **Day 6** will focus on:
- Interactive Single-Page Topology & Chaos Engineering Console (`frontend/`).
- Live SVG vector mesh graph with status-reactive borders and animated pulse connectors.
- Real-time scale controls, traffic routing simulators, and chaos fault injection tools.
- Distributed saga trace waterfall viewer and Message Broker DLQ inspector.
