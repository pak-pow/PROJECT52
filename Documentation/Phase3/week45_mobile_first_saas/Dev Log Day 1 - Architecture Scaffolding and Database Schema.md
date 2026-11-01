---
date: 2026-11-01
project: PulseSub Mobile-First SaaS Subscription & Entitlements Platform
topic: Day 1 - Architecture Scaffolding and Database Schema
Tags:
  - "[[SaaS-Billing]]"
  - "[[Subscription-Lifecycle]]"
  - "[[SQLite-WAL]]"
  - "[[Architecture-Scaffolding]]"
  - "[[Mobile-First]]"
  - "[[Dev Log]]"
---
# DEV LOG: WEEK 45 - DAY 1

**Core Objective:** Scaffold the production repository architecture for PulseSub, establish the SQLite WAL relational schema for subscription lifecycles, billing invoices, usage records, plans, and Stripe-compatible webhooks, construct domain models and seeding engines, and build the mobile-first frontend viewport shell backed by 26 automated tests passing with 93% coverage and 0 linter/security warnings.

---

## 1. The Big Picture and Simple Explanation

Subscription management and billing engines are among the most business-critical systems in software engineering. Unlike standard CRUD applications, SaaS billing systems must handle:
1. **Deterministic Lifecycle States:** Subscriptions transition through strict states (`trialing`, `active`, `past_due`, `canceled`, `expired`). Invalid state jumps (such as marking an expired subscription directly as trialing) must be blocked at both the database and application levels.
2. **Proration and Invoice Invariants:** Mid-cycle upgrades and downgrades require exact linear time calculations to credit unused prepaid days and compute the immediate balance due. Invoices must record exact historical line items that cannot be retroactively modified once paid.
3. **High-Throughput Metered Consumption:** Users consume resources (API requests, file storage, project counts) continuously. Usage tracking must support atomic increments and idempotency keys to prevent duplicate billing.
4. **Idempotent Webhook Processing:** Inbound payment gateway events (e.g., Stripe webhooks) can arrive out of order or be retried multiple times. The platform must verify cryptographic HMAC-SHA256 signatures, enforce a timestamp replay tolerance window, and record event identifiers to guarantee once-only execution.
5. **Mobile-First Responsive Interface:** Modern users and founders manage apps on smartphones first. The user experience requires bottom tab navigation, touch targets of at least 48px, swipeable cards, and a built-in device simulator frame.

In Day 1, we established the core foundation for **PulseSub**:
- Clean separation between backend domain services, SQLite WAL persistence, and mobile-first frontend controllers.
- Relational schema in `backend/data/schema.sql` covering `users`, `plans`, `subscriptions`, `invoices`, `usage_records`, and `webhook_events`.
- Robust SQLite connection manager with WAL mode, foreign key enforcement, busy timeout protection, and dictionary-accessible row factories.
- Domain dataclasses (`Plan`, `Subscription`, `Invoice`, `UsageRecord`, `WebhookEventRecord`) with JSON serialization and deserialization.
- Seeding engine initializing all 4 default tiers (`Free`, `Starter`, `Pro`, `Enterprise`) and demo accounts.
- Mobile-first responsive app shell with sticky header, bottom navigation bar, and dark/light theme switching with localStorage persistence.

```
+-----------------------------------------------------------------------------+
|                          PulseSub Core Entity Model                         |
|                                                                             |
|      +---------------------+          +---------------------------+         |
|      |        users        |          |           plans           |         |
|      +---------------------+          +---------------------------+         |
|      | id (PK, usr_...)    |          | id (PK, plan_...)         |         |
|      | email (UNIQUE)      |          | name (Free/Starter/Pro...) |         |
|      | password_hash       |          | tier (free/pro/...)       |         |
|      | role (user/admin)   |          | price_monthly_cents       |         |
|      +---------------------+          | price_annual_cents        |         |
|         |               |             | limits_json (quotas)      |         |
|         | 1:N           | 1:N         | features_json (flags)     |         |
|         v               v             +---------------------------+         |
|  +---------------+  +---------------+               |                       |
|  | subscriptions |  |   invoices    |               | 1:N                   |
|  +---------------+  +---------------+               |                       |
|  | id (PK)       |  | id (PK)       |<--------------+                       |
|  | user_id (FK)  |  | user_id (FK)  |                                       |
|  | plan_id (FK) -+  | sub_id (FK)   |                                       |
|  | interval      |  | amount_due    |                                       |
|  | status        |  | amount_paid   |                                       |
|  | period_start  |  | line_items    |                                       |
|  | period_end    |  | status        |                                       |
|  +---------------+  +---------------+                                       |
|         |                                                                   |
|         | 1:N                                                               |
|         v                                                                   |
|  +---------------+                    +---------------------------+         |
|  | usage_records |                    |      webhook_events       |         |
|  +---------------+                    +---------------------------+         |
|  | id (PK)       |                    | id (PK)                   |         |
|  | user_id (FK)  |                    | event_id (UNIQUE, evt_...) |         |
|  | metric_key    |                    | event_type                |         |
|  | quantity      |                    | payload_json              |         |
|  | idempotency   |                    | status (pending/processed)|         |
|  +---------------+                    +---------------------------+         |
+-----------------------------------------------------------------------------+
```

---

## 2. Key Engineering Implementations

### SQLite WAL Mode and Strict Foreign Keys
To prevent concurrent read/write locks, the database connection manager explicitly configures SQLite WAL (Write-Ahead Logging) and `NORMAL` synchronous mode for disk-based databases, with a 5000ms busy timeout:

```python
def get_db_connection(db_path: Optional[Union[str, Path]] = None) -> sqlite3.Connection:
    target_path = db_path if db_path is not None else Config.DB_PATH
    is_memory = str(target_path) == ":memory:"

    if not is_memory:
        p = Path(target_path)
        p.parent.mkdir(parents=True, exist_ok=True)

    conn = sqlite3.connect(
        str(target_path),
        timeout=10.0,
        detect_types=sqlite3.PARSE_DECLTYPES | sqlite3.PARSE_COLNAMES,
    )
    conn.row_factory = sqlite3.Row

    cursor = conn.cursor()
    cursor.execute("PRAGMA foreign_keys = ON;")
    if not is_memory:
        cursor.execute("PRAGMA journal_mode = WAL;")
        cursor.execute("PRAGMA synchronous = NORMAL;")
    cursor.execute("PRAGMA busy_timeout = 5000;")
    cursor.close()

    return conn
```

### Plan Tier Specifications and JSON Gatekeeping
Four standard tiers were seeded with quantitative limits and boolean feature flags:
- **Free:** $0/mo, 1 project, 100 API calls/mo, basic analytics, community support.
- **Starter:** $19/mo ($190/yr), 5 projects, 5,000 API calls/mo, export data, email support.
- **Pro:** $49/mo ($490/yr), unlimited projects, 50,000 API calls/mo, custom domains, team seats, priority support.
- **Enterprise:** $199/mo ($1,990/yr), unlimited projects, 500,000 API calls/mo, audit logs, SSO, dedicated SLA.

### Mobile-First Design System Tokens
The frontend layout was built from the ground up for mobile devices (max-width: 600px, bottom navigation height: 64px, minimum touch target: 48px) with dual theme variables (`--bg-primary`, `--border-color`, `--accent-primary`) supporting instant toggle between dark and light modes.

---

## 3. Verification & Quality Gates

All 5 project quality gates passed with zero warnings:

1. **Pytest Test Suite:** 26/26 tests passing in 0.64 seconds.
2. **Test Coverage:** 93% line coverage across `app` and `data` packages.
3. **Flake8 Linter:** 0 errors/warnings at 88-character maximum line length.
4. **Black Formatter:** Code formatted and verified across all modules.
5. **isort:** All module and third-party imports sorted and separated.
6. **Bandit Security Audit:** 0 security vulnerabilities identified across 714 lines of code.

```text
============================= test session starts =============================
collected 26 items

tests/test_connection.py::test_get_db_connection_in_memory PASSED        [  3%]
tests/test_connection.py::test_get_db_connection_file_based PASSED       [  7%]
tests/test_connection.py::test_row_factory_access PASSED                 [ 11%]
tests/test_connection.py::test_close_db_connection PASSED                [ 15%]
tests/test_connection.py::test_init_db_missing_schema_raises PASSED      [ 19%]
tests/test_health.py::test_health_endpoint_healthy PASSED                [ 23%]
tests/test_health.py::test_health_degraded_when_db_fails PASSED          [ 26%]
tests/test_health.py::test_security_headers_present PASSED               [ 30%]
tests/test_health.py::test_static_index_route PASSED                     [ 34%]
tests/test_health.py::test_static_css_route PASSED                       [ 38%]
tests/test_health.py::test_static_js_route PASSED                        [ 42%]
tests/test_health.py::test_config_retrieval PASSED                       [ 46%]
tests/test_models.py::test_plan_serialization PASSED                     [ 50%]
tests/test_models.py::test_subscription_serialization PASSED             [ 53%]
tests/test_models.py::test_invoice_serialization_and_from_row PASSED     [ 57%]
tests/test_models.py::test_usage_record_serialization_and_from_row PASSED [ 61%]
tests/test_models.py::test_webhook_event_record_serialization_and_from_row PASSED [ 65%]
tests/test_schema.py::test_schema_tables_exist PASSED                    [ 69%]
tests/test_schema.py::test_schema_indexes_exist PASSED                   [ 73%]
tests/test_schema.py::test_users_unique_email_constraint PASSED          [ 76%]
tests/test_schema.py::test_plans_tier_and_price_check_constraints PASSED [ 80%]
tests/test_schema.py::test_foreign_key_cascade_deletion PASSED           [ 84%]
tests/test_seed.py::test_seed_plans_created PASSED                       [ 88%]
tests/test_seed.py::test_seed_database_idempotency PASSED                [ 92%]
tests/test_seed.py::test_demo_user_and_subscription PASSED               [ 96%]
tests/test_seed.py::test_hash_password_deterministic PASSED              [100%]

=============================== tests coverage ================================
TOTAL: 330 statements, 23 misses, 93% coverage
============================= 26 passed in 0.64s ==============================
```

---

## 4. Next Steps

With Day 1 architecture, database schema, and test scaffolding complete, **Day 2** will focus on:
- Subscription lifecycle state machine (`SubscriptionFSM`).
- Linear proration engine for calculating exact upgrade/downgrade balances and credits.
- Invoice generation on tier changes and period renewals.
