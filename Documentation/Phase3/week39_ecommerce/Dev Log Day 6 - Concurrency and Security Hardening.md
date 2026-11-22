---
date: 2026-09-26
project: Production E-Commerce Platform v1
topic: Day 6 - Concurrency and Security Hardening
Tags:
  - "[[E-Commerce]]"
  - "[[Security]]"
  - "[[Concurrency]]"
  - "[[Testing]]"
  - "[[Dev Log]]"
---
# DEV LOG: WEEK 39 - DAY 6

**Core Objective:** Harden the ShopPulse e-commerce platform against real-world vulnerabilities and concurrency race conditions, featuring SQLite busy-timeout configuration, atomic stock decrement queries, XSS sanitization, promo code validation, and a multi-threaded stress test suite.

---

## 1. The Big Picture and Simple Explanation

During high-demand flash sales or product drops, multiple shoppers may attempt to purchase the final unit of stock at the exact same millisecond. If the application separates the stock verification from the stock deduction (`SELECT stock` followed by `UPDATE stock`), concurrent requests can read the same stock count simultaneously, causing severe inventory overselling.

In Day 6, we hardened the platform against concurrency conflicts and malicious input:
1. **Atomic SQL Stock Deduction:** Replaced multi-step check-then-act logic with a single atomic conditional query: `UPDATE products SET stock_quantity = stock_quantity - ? WHERE id = ? AND stock_quantity >= ?`. The database engine guarantees mutual exclusion, allowing only one thread to purchase the final item.
2. **SQLite Concurrency Resilience:** Configured `PRAGMA busy_timeout = 30000;` and a 30-second connection timeout, allowing concurrent write transactions to queue cleanly rather than failing with lock errors.
3. **Cross-Site Scripting (XSS) Sanitization:** Neutralized injection vectors by sanitizing customer names and shipping addresses using `html.escape()` prior to persistence and invoice generation.
4. **Mandatory Webhook Cryptographic Verification:** Removed mock signature fallbacks in the Stripe webhook route, enforcing valid HMAC-SHA256 signatures on all inbound notifications.
5. **Input Boundary Fuzzing Protection:** Capped coupon codes at 30 characters with strict regex validation to block buffer overruns and SQL fuzzing attacks.

```
+-----------------------------------------------------------------------------+
|                     High-Concurrency Atomic Stock Reservation               |
|                                                                             |
|      +---------------------------------------------------------------+      |
|      | 10 Concurrent Shopper Threads Competing for 1 Remaining Item  |      |
|      +-------------------------------+-------------------------------+      |
|                                      |                                      |
|                                      v                                      |
|      +---------------------------------------------------------------+      |
|      | Single Atomic SQL Query:                                      |      |
|      | UPDATE products SET stock_quantity = stock_quantity - 1       |      |
|      | WHERE id = ? AND stock_quantity >= 1                          |      |
|      +-------------------------------+-------------------------------+      |
|                                      |                                      |
|                     +----------------+----------------+                     |
|                     |                                 |                     |
|             [cur.rowcount == 1]               [cur.rowcount == 0]           |
|                     v                                 v                     |
|      +-------------------------------+    +--------------------------+      |
|      | Transaction Winner (1 Thread) |    | Contention (9 Threads)   |      |
|      | - Inventory Reserved          |    | - Rollback Transaction   |      |
|      | - Order Created & Authorized  |    | - Return HTTP 409        |      |
|      | - Final Stock = 0             |    | - Zero Overselling       |      |
|      +-------------------------------+    +--------------------------+      |
+-----------------------------------------------------------------------------+
```

---

## 2. Key Engineering Implementations

### Database Concurrency Resilience (`app/db.py`)
- Configured connection timeout parameter: `sqlite3.connect(..., timeout=30.0)`.
- Enforced busy timeout pragma on every connection checkout: `PRAGMA busy_timeout = 30000;`. SQLite automatically queues concurrent write locks up to 30 seconds before timing out.

### Atomic Inventory Deductions (`app/models/order_model.py`)
- Rewrote checkout reservations to enforce atomic row-level conditions:
  ```python
  cur.execute(
      """
      UPDATE products
      SET stock_quantity = stock_quantity - ?
      WHERE id = ? AND stock_quantity >= ?
      """,
      (item["quantity"], item["product_id"], item["quantity"])
  )
  if cur.rowcount == 0:
      conn.rollback()
      raise InsufficientStockError(f"Insufficient stock for product {item['product_id']}")
  ```
- Sanitized `customer_name` and `shipping_address` with `html.escape()`, ensuring stored XSS attacks cannot execute in printable receipts.

### Input Hardening and Webhook Verification
- Promo code boundaries: Enforced strict 30-character length limits and alphanumeric constraints in `app/routes/cart_routes.py`.
- Webhook signature enforcement: Required valid `Stripe-Signature` headers on all incoming webhook payloads, returning HTTP 400 immediately if missing or invalid.

### Concurrency and Security Test Suite (`tests/test_concurrency_and_security.py`)
- **10-Thread Flash Sale Concurrency Test:** Executed 10 concurrent threads attempting to purchase 1 remaining inventory item. Asserted exactly 1 thread succeeds (HTTP 201) while 9 receive HTTP 409 Conflict, and final stock balances at exactly 0.
- **SQL Injection Fuzzing Test:** Verified parameterized queries neutralize SQL injection payloads across search, filtering, and promo codes.
- **XSS Neutralization Test:** Asserted script tags in customer addresses render harmlessly as escaped entities.
- **Webhook Forgery Verification:** Confirmed forged or unsigned webhook requests are rejected with HTTP 400.

---

## 3. Verification & Quality Gates Report

The security defenses and concurrency guarantees were validated across automated tests:

| Test Suite Module | Target Scope | Tests Passed | Status |
| :--- | :--- | :---: | :---: |
| `tests/test_concurrency_and_security.py` | 10-thread flash sale race condition, XSS, fuzzing | 6 | Passed |
| `tests/test_orders_and_webhooks.py` | Order state machine, payment flows, webhook idempotency | 11 | Passed |
| `tests/test_cart.py` | Cart operations and pricing calculations | 13 | Passed |
| `tests/test_products.py` | Catalog listings and search queries | 12 | Passed |
| `tests/test_health_and_version.py` | Liveness, readiness, and version probes | 4 | Passed |
| `tests/test_coverage_boost.py` | Error handlers and edge conditions | 6 | Passed |

### Quality Gates Execution Summary
- **Total Tests:** 52 automated tests passing in 1.48 seconds.
- **Branch Coverage:** 95.39% total branch coverage.
- **Security Audit:** Bandit scan clean with 0 issues.
- **Concurrency Result:** 10 concurrent threads completed with 0 overselling.

---

## 4. Key Engineering Insights & Decisions

- **Atomic Queries Prevent Race Conditions:** Combining the stock availability check and update into a single conditional SQL statement eliminates race windows without requiring heavy external distributed lock services.
- **Busy Timeouts Enable SQLite Multi-Threading:** Configuring `busy_timeout` prevents immediate `database is locked` crashes during write contention by queueing locks gracefully.
- **Input Sanitization at the Model Boundary:** Sanitizing text fields like shipping addresses before database persistence protects both internal admin consoles and public invoice displays.

---

## 5. Next Steps

- Finalize UI bug fixes, replace heavy CSS icons with inline SVGs, and integrate Week 39 into the Phase 3 master hub portal.
- Complete system handover and technical documentation in `week39_ecommerce_store/README.md`.
