---
date: 2026-11-15
project: PulseMesh Microservices Architecture & Distributed Systems Platform
topic: Day 1 - Microservices Foundation & Service Registry
Tags:
  - "[[Microservices]]"
  - "[[Service-Discovery]]"
  - "[[Heartbeat-Eviction]]"
  - "[[Structured-Logging]]"
  - "[[CloudEvents]]"
  - "[[Dev Log]]"
---
# DEV LOG: WEEK 47 - DAY 1

**Core Objective:** Scaffold the production repository architecture for PulseMesh, establish the shared core domain models, configuration profiles, and structured JSON correlation logging engine, and construct the thread-safe Service Registry & Discovery Service (`services/registry/`) featuring dynamic node registration, heartbeat renewal, and automated lease-based stale instance eviction backed by 17 automated tests passing with 98.03% branch coverage and 0 linter/security warnings.

---

## 1. The Big Picture and Simple Explanation

In monolithic software architectures, modules communicate via in-process memory pointers. In a distributed microservices platform, services run in separate processes across isolated network boundaries. This introduces fundamental distributed systems challenges:
1. **Dynamic Network Location & Ephemeral Topology:** Microservice instances frequently spin up, crash, scale up, or relocate across ports and IP addresses. Hardcoding static IP addresses into downstream services leads to brittle deployments. A centralized Service Registry acts as an authoritative service catalog, allowing nodes to register their network coordinates on boot and deregister on shutdown.
2. **Heartbeats & Leases vs. Polling:** Waiting for an HTTP client to fail on a dead server degrades end-user latency. By enforcing periodic heartbeats (e.g. every 5s) and automatic lease expiration (15s timeout), a background watchdog prunes stale or crashed nodes before traffic is routed to them.
3. **Structured Distributed Tracing:** In multi-hop architectures (Gateway -> Order -> Catalog -> Notification), diagnosing latencies and bugs requires end-to-end trace correlation. Injecting and propagating `X-Correlation-ID` across thread boundaries and JSON log lines is essential for distributed observability.
4. **Shared Protocol Contracts (CloudEvents v1.0):** Event-driven microservices require standardized event envelopes across boundaries (`id`, `source`, `type`, `time`, `data`, `correlation_id`).

```
+-----------------------------------------------------------------------------------------+
|                    PulseMesh Service Discovery & Heartbeat Topology                     |
|                                                                                         |
|       +-----------------------------------------------------------------------+        |
|       |                 Service Registry & Store (Port 8001)                  |        |
|       |  * Thread-Safe InstanceStore (threading.RLock)                        |        |
|       |  * HeartbeatWatchdog (Background Daemon Lease Eviction)               |        |
|       |  * Aggregated Catalog Queries: GET /api/v1/registry/services          |        |
|       +-------------------+-------------------------------+-------------------+        |
|                           ^                               ^                            |
|       POST /register      | POST /heartbeat               | GET /services/<name>       |
|       DELETE /deregister  | (Every 5 seconds)             | (Fetch healthy instances)  |
|                           |                               |                            |
|       +-------------------+                               +-------------------+        |
|       |                                                                       |        |
|       v                                                                       v        |
|  +---------------------------+                               +-----------------------+ |
|  | Microservice Node Boot    |                               | Resilient HTTP Client | |
|  | * Auto-registers instance |                               | (PulseClient/Gateway) | |
|  | * Emits heartbeats loop   |                               | * Discovers upstreams | |
|  | * Traps SIGTERM to unreg  |                               | * Filters healthy     | |
|  +---------------------------+                               +-----------------------+ |
+-----------------------------------------------------------------------------------------+
```

---

## 2. Key Engineering Implementations

### Shared Core Library & Contracts (`shared/models.py`)
- **`ServiceInstance`:** Dataclass representing an active node in the mesh:
  - Fields: `service_name`, `instance_id`, `host`, `port`, `tags`, `weight`, `health_status`, `registered_at`, `last_heartbeat`, `metadata`.
  - Properties: `base_url` (`http://{host}:{port}`).
  - Methods: `update_heartbeat(status=None)`, `is_stale(lease_timeout_seconds)`, `to_dict()`, `from_dict()`.
- **`ServiceRegistration`:** Strongly validated payload schema ensuring port boundaries (1-65535), non-empty service and host names, and positive weights. Supports deterministic auto-generation of unique instance IDs (`{service_name}-{port}-{token_hex(4)}`) when omitted.
- **`CloudEvent`:** Conforms strictly to the CloudEvents v1.0 specification (`id`, `source`, `type`, `specversion="1.0"`, `time`, `data`, `correlation_id`, `datacontenttype="application/json"`).
- **`CorrelationContext`:** Thread-local storage (`threading.local`) managing `X-Correlation-ID` continuity across thread boundaries, supporting contextual scope managers (`with CorrelationContext.scope(corr_id): ...`).

### Configuration Profiles & Logging (`shared/config.py`, `shared/logging.py`)
- **`BaseMeshConfig`:** Centralizes default ports across the topology:
  - API Gateway: 8000
  - Service Registry: 8001
  - User Service: 8002 (replica: 8012)
  - Catalog Service: 8003 (replica: 8013)
  - Order Service: 8004 (replica: 8014)
  - Notification Service: 8005
- **Timing & Resilience Parameters:**
  - Heartbeat interval: 5.0s (testing: 0.5s)
  - Lease timeout: 15.0s (testing: 1.0s)
  - Eviction sweep interval: 5.0s (testing: 0.5s)
  - Circuit breaker failure threshold: 5 (testing: 3)
  - Circuit breaker recovery timeout: 10.0s (testing: 1.0s)
- **`StructuredJsonFormatter`:** Outputs single-line JSON log events capturing timestamp, log level, service name, instance ID, active correlation ID, message, exception tracebacks, and structured extra fields.

### Service Registry & Discovery Service (`services/registry/`)
- **`InstanceStore`:**
  - Employs an `RLock` to synchronize multi-threaded access.
  - Maintains indexes allowing O(1) lookups by `instance_id` and filtered queries by `service_name` and `health_status`.
  - Evaluates timestamp differences against lease timeouts during `evict_stale()`.
- **REST API Endpoints:**
  - `POST /api/v1/registry/register`: Accepts registration payload, performs schema validation, registers node, and returns HTTP 201 Created.
  - `POST /api/v1/registry/heartbeat`: Renews instance lease timestamp. Returns HTTP 404 if the instance was evicted or never registered.
  - `DELETE /api/v1/registry/deregister`: Removes instance upon graceful shutdown.
  - `GET /api/v1/registry/services`: Returns comprehensive mesh overview with total and healthy instance counts.
  - `GET /api/v1/registry/services/<service_name>`: Returns active nodes for a given service, with optional `healthy_only=false` flag.
  - `GET /api/v1/registry/instances/<instance_id>`: Retrieves single instance metadata.
  - `GET /health`: Registry vitality probe reporting active instance count.
- **Heartbeat Watchdog (`HeartbeatWatchdog`):**
  - Background daemon thread executing periodic sweeps.
  - Safely joins on shutdown without hanging application lifecycle.

---

## 3. Verification & Quality Gates Report

All 5 automated quality gates were executed locally and passed with zero errors:

```text
======================================================================
  PULSEMESH MICROSERVICES: 5-STAGE AUTOMATED QUALITY GATES
======================================================================
  * FLAKE8 LINTER        : [OK] PASSED (0.89s)
  * BLACK FORMATTER      : [OK] PASSED (0.90s)
  * ISORT IMPORTS        : [OK] PASSED (0.22s)
  * BANDIT SECURITY      : [OK] PASSED (0.52s)
  * PYTEST COVERAGE      : [OK] PASSED (2.17s)
======================================================================
>> ALL QUALITY GATES PASSED! Safe to commit and deploy.
```

### Test Suite Metrics
- Total Tests: **17 passed** (0 failed, 0 skipped).
- Branch Coverage: **98.03%** (384 statements, 4 missed; 72 branches, 5 partial). Exceeds the 90.0% project threshold.
- Zero-Emoji Compliance: 100% verified across all code, tests, documentation, and commit messages.

---

## 4. Key Engineering Insights & Decisions

1. **Heartbeat Leases vs. Active Health Check Polling:**
   Having the registry poll hundreds of microservices creates $O(N)$ outbound network sockets and can trigger false-positive evictions during brief CPU spikes. An inbound heartbeat lease model inverts the responsibility: microservices publish cheap heartbeat pings every few seconds, and the registry merely tracks elapsed time since last renewal.
2. **Re-entrant Mutex Locks in Memory-First Registries:**
   While distributed consensus systems like Raft or etcd are utilized for multi-datacenter setups, in-process thread-safe registries (`threading.RLock`) provide sub-millisecond discovery lookups with zero network overhead.
3. **Structured Correlation Logging:**
   Attaching `correlation_id` directly to thread-local contexts ensures that every microservice log statement automatically outputs its parent trace ID, enabling unified log aggregation across asynchronous boundaries.

---

## 5. Next Steps

Day 1 delivers the operational foundation of PulseMesh. **Day 2** will focus on:
- Resilient Inter-Service HTTP Client (`PulseClient`).
- Martin Fowler Circuit Breaker Pattern Engine (`CLOSED`, `OPEN`, `HALF-OPEN`).
- Multi-Strategy Load Balancer implementing Round-Robin, Least-Connections, Latency-Weighted, and Random selection.
- Resilience testing for circuit tripping, self-healing resets, and load distribution.
