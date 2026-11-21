---
date: 2026-11-17
project: PulseMesh Microservices Architecture & Distributed Systems Platform
topic: Day 3 - User Service & Catalog Inventory Service
Tags:
  - "[[Microservices]]"
  - "[[Database-per-Service]]"
  - "[[Atomic-Reservation]]"
  - "[[Saga-Compensation]]"
  - "[[SQLite-WAL]]"
  - "[[Dev Log]]"
---
# DEV LOG: WEEK 47 - DAY 3

**Core Objective:** Implement the first two domain microservices for PulseMesh following the strict Database-per-Service architectural pattern, developing the User & Identity Service (`services/user/`) with isolated SQLite WAL persistence (`users.db`) and API key authentication, the Catalog & Inventory Service (`services/catalog/`) with isolated `catalog.db` and an Atomic Stock Reservation Engine supporting 2-phase saga commits and compensating releases (`/reserve`, `/release`, `/commit`), and the background automated Service Registrar (`shared/registration.py`) backed by 56 passing tests with 94.80% branch coverage and 0 linter/security warnings.

---

## 1. The Big Picture and Simple Explanation

A defining invariant of microservices architecture is **Database-per-Service**. In monolithic applications, multiple domains join tables across foreign keys (`JOIN users ON orders.user_id = users.id`). While convenient, shared databases create tight coupling, prevent independent schema migrations, and cause cross-service database locks:
1. **Loose Coupling & Schema Autonomy:** User Service must be able to alter its table schemas without coordinating with Catalog or Order services. Each service owns its private database file (`users.db`, `catalog.db`), and all inter-service communication occurs strictly via versioned HTTP APIs.
2. **Distributed Data Integrity without 2PC:** Because databases are completely isolated, traditional distributed two-phase commits (2PC) are avoided due to lock-holding overhead and coordinator blocking. Instead, systems use **Sagas**: breaking multi-service operations into local transactions with explicit compensating actions.
3. **Atomic Inventory Reservation:** In e-commerce checkout flows, stock must not be permanently deducted until payment succeeds. However, stock cannot remain unreserved while the customer pays, or another user could purchase it (race condition). The inventory engine implements a three-step lifecycle:
   - `RESERVE`: Atomically holds stock for an order; rolls back immediately if any SKU has insufficient inventory.
   - `RELEASE`: Compensating transaction executed if payment fails or order cancels; releases reserved stock back to the available pool.
   - `COMMIT`: Permanently decrements physical stock upon successful payment settlement.

```
+-----------------------------------------------------------------------------------------+
|                  Database-per-Service & Atomic Reservation Lifecycle                    |
|                                                                                         |
|       User Service (Port 8002)                    Catalog Service (Port 8003)           |
|       +-----------------------------+             +-----------------------------+       |
|       | users.db (SQLite WAL)       |             | catalog.db (SQLite WAL)     |       |
|       | * users                     |             | * products (stock_quantity) |       |
|       | * addresses                 |             | * inventory_reservations    |       |
|       +-----------------------------+             +-----------------------------+       |
|                                                                                         |
|       ===========================================================================       |
|                            Atomic Stock Reservation Engine                              |
|       ===========================================================================       |
|                                                                                         |
|       1. POST /api/v1/inventory/reserve                                                 |
|          Order Saga starts -> Checks: available_stock = stock_quantity - reserved       |
|          If all items available: Insert reservation (status: 'RESERVED')                |
|          If ANY item insufficient: Immediate rollback -> 400 Insufficient Stock         |
|                                    |                                                    |
|                    +---------------+---------------+                                    |
|                    |                               |                                    |
|         (Payment Fails / Cancel)         (Payment Succeeds)                             |
|                    |                               |                                    |
|                    v                               v                                    |
|       2. POST /inventory/release        3. POST /inventory/commit                       |
|          Compensating action:              Permanent settlement:                        |
|          status -> 'RELEASED'              stock_quantity = stock_quantity - qty        |
|          Stock returns to pool             status -> 'COMMITTED'                        |
+-----------------------------------------------------------------------------------------+
```

---

## 2. Key Engineering Implementations

### Automated Service Registrar & Heartbeat Client (`shared/registration.py`)
- **Lifecycle Management:**
  - `ServiceRegistrar` automates node registration upon service startup.
  - Supports both direct in-process store registration (for fast hermetic testing) and remote HTTP REST calls against the Discovery Registry (`POST /api/v1/registry/register`).
  - Spawns a background daemon thread emitting periodic heartbeats (`POST /api/v1/registry/heartbeat`) with auto-reconnection and re-registration on expired leases.
  - Graceful shutdown hook (`deregister()`) stopping heartbeat loops and notifying the registry.

### User & Identity Microservice (`services/user/`)
- **Isolated Storage Architecture (`users.db`):**
  - SQLite in Write-Ahead Logging (`PRAGMA journal_mode=WAL; PRAGMA busy_timeout=30000;`).
  - `users`: `id`, `email`, `full_name`, `role`, `api_key`, `is_active`, `created_at`.
  - `addresses`: `id`, `user_id`, `street`, `city`, `country`, `postal_code`, `is_default`, `created_at`.
- **REST Endpoints:**
  - `GET /health`: Vitality probe reporting node state and instance ID.
  - `POST /api/v1/users`: Creates user account with auto-generated UUID (`usr_...`) and unique API key (`pk_...`). Rejects duplicate emails with HTTP 400.
  - `GET /api/v1/users/<user_id>`: Retrieves user profile along with linked shipping/billing addresses.
  - `GET /api/v1/users`: Lists registered user accounts with pagination controls.
  - `POST /api/v1/users/<user_id>/addresses`: Adds postal address, managing default address flags.
  - `POST /api/v1/users/authenticate`: Verifies credentials by API key, returning user identity or HTTP 401 Unauthorized.
- **Distributed Tracing:** Automatically extracts, binds, and appends `X-Correlation-ID` headers to all responses.

### Catalog & Inventory Microservice (`services/catalog/`)
- **Isolated Storage Architecture (`catalog.db`):**
  - SQLite WAL mode with 30-second busy timeout locks.
  - `products`: `id`, `sku`, `name`, `description`, `price_cents`, `stock_quantity`, `is_active`, `created_at`.
  - `inventory_reservations`: `id`, `order_id`, `product_id`, `quantity`, `status` (`RESERVED`, `COMMITTED`, `RELEASED`), `created_at`, `updated_at`.
- **Atomic Stock Reservation Engine (`InventoryEngine`):**
  - **All-or-Nothing Reservation (`POST /api/v1/inventory/reserve`):**
    - Validates available stock for every line item in an order batch: `available_stock = stock_quantity - sum(active reserved)`.
    - If any item in the order has insufficient stock, the entire transaction rolls back immediately with HTTP 400 `Insufficient stock` without leaving orphaned partial reservations.
    - Idempotency guard: If the order is already in `RESERVED` status, returns existing reservation metadata without duplicate deductions.
  - **Compensating Action (`POST /api/v1/inventory/release`):**
    - Distributed saga compensation endpoint.
    - In the event of payment failure or downstream cancellation, marks all active reservations for `order_id` as `RELEASED`, instantly returning the locked items to the available inventory pool.
  - **Commit Action (`POST /api/v1/inventory/commit`):**
    - Executed upon successful order completion and payment settlement.
    - Permanently decrements the physical `stock_quantity` on the `products` table and updates reservation statuses to `COMMITTED`.
  - **Reservation Audit (`GET /api/v1/inventory/orders/<order_id>`):**
    - Returns full reservation audit history for a given order ID.

---

## 3. Verification & Quality Gates Report

All 5 automated enterprise quality gates passed cleanly with zero warnings:

```text
======================================================================
  PULSEMESH MICROSERVICES: 5-STAGE AUTOMATED QUALITY GATES
======================================================================
  * FLAKE8 LINTER        : [OK] PASSED (0.90s)
  * BLACK FORMATTER      : [OK] PASSED (0.33s)
  * ISORT IMPORTS        : [OK] PASSED (0.27s)
  * BANDIT SECURITY      : [OK] PASSED (0.51s)
  * PYTEST COVERAGE      : [OK] PASSED (2.92s)
======================================================================
>> ALL QUALITY GATES PASSED! Safe to commit and deploy.
```

### Test Suite Metrics
- Total Tests: **56 passed** (0 failed, 0 skipped), up from 38 in Day 2:
  - `test_user_service.py` (7 tests)
  - `test_catalog_service.py` (8 tests)
  - `test_service_registration.py` (3 tests)
  - Pre-existing Day 1 and Day 2 test suites (38 tests)
- Branch Coverage: **94.80%** (1245 statements, 43 missed; 256 branches, 31 partial). Exceeds the 90.0% requirement.
- Zero-Emoji Compliance: 100% verified across all source code, tests, documentation, and commit messages.

---

## 4. Key Engineering Insights & Decisions

1. **Calculated Available Stock vs. Pre-decremented Stock:**
   Deducting `stock_quantity` immediately upon reservation requires rolling back the counter upon payment failure. If the database crashes mid-rollback, inventory is permanently lost. Storing reservations in a dedicated `inventory_reservations` table and computing `available_stock = stock_quantity - SUM(RESERVED)` ensures that uncommitted reservations can be audited, expired, or released without touching physical stock records.
2. **Batch Reservation Atomicity:**
   If a customer orders 3 items and items 1 and 2 are in stock while item 3 is out of stock, reserving items 1 and 2 before failing leaves orphaned stock locks. The reservation engine validates all batch items in Phase 1 before inserting any reservation records in Phase 2.
3. **Database-per-Service Enforces Physical Isolation:**
   By creating completely separate SQLite database connection pools (`users.db` and `catalog.db`), schema modifications, index rebuilds, or write transactions in one service can never lock or block the other.

---

## 5. Next Steps

With User and Catalog domain microservices operational, **Day 4** will focus on:
- Asynchronous CloudEvents Message Broker & Dead-Letter Queue (`shared/message_broker.py`).
- Order & Saga Coordinator Microservice (`services/order/`) with multi-step orchestration and automated compensating rollbacks (`saga.py`).
- Notification & Audit Microservice (`services/notification/`) with idempotent event consumer deduplication.
