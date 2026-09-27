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

When an online store hosts a popular product release, hundreds of customers may attempt to purchase the exact same item at the exact same second. If the application reads inventory before updating it (`SELECT stock` followed by `UPDATE stock`), two or more checkout requests can read the same stock count simultaneously, leading to overselling.

Today, we hardened the backend against concurrency conflicts and malicious input:
1. **Eliminating Race Conditions:** We changed inventory deduction to an atomic, conditional SQL update: `UPDATE products SET stock = stock - ? WHERE id = ? AND stock >= ?`. Even if multiple checkouts hit the database simultaneously, the database guarantees that only one request succeeds per available unit of inventory.
2. **Preventing Database Lock Failures:** Under concurrent threads, SQLite can throw `OperationalError: database is locked`. We configured a 30-second connection timeout and `PRAGMA busy_timeout = 30000`, allowing transactions to wait gracefully rather than crashing.
3. **Cross-Site Scripting (XSS) Sanitization:** Customer names and shipping addresses are sanitized using `html.escape()` before being stored and rendered in printable invoices.
4. **Strict Webhook Verification:** Removed default fallback signatures in the Stripe webhook endpoint to ensure only authentic cryptographic signatures are accepted.
5. **Promo Code Fuzzing Protection:** Capped coupon codes at 30 characters and added strict format checks to block SQL fuzzing and injection attempts.

```mermaid
graph TD
    subgraph ConcurrentCheckouts["10 Concurrent Shoppers Competing for 1 Item"]
        T1["Thread 1"]
        T2["Thread 2"]
        TN["Threads 3-10"]
    end

    T1 --> AtomicSQL["Atomic SQL Reservation (stock >= 1)"]
    T2 --> AtomicSQL
    TN --> AtomicSQL

    AtomicSQL -->|Winner (1 Order)| Success["Order Created & Payment Authorized"]
    AtomicSQL -->|Losers (9 Orders)| OutOfStock["409 Insufficient Stock Error (Zero Overselling)"]
```

---

## 2. Key Modules and Features Implemented Today

### Database Concurrency Resilience (`app/db.py`)
- Configured `sqlite3.connect(..., timeout=30.0)` for all database operations.
- Executed `PRAGMA busy_timeout = 30000;` on connection creation, allowing SQLite to queue concurrent writes for up to 30 seconds instead of throwing lock errors.

### Atomic Inventory Protection (`app/models/order_model.py`)
- Rewrote checkout stock deductions to enforce row-level conditions: `WHERE id = ? AND stock_quantity >= ?`.
- Verified row counts after execution (`cur.rowcount == 0` triggers an immediate rollback), completely preventing negative inventory balances.
- Sanitized `customer_name` and `shipping_address` with `html.escape()` to protect admin and invoice views against stored XSS attacks.

### Promo Validation Hardening (`app/routes/cart_routes.py`)
- Restricted coupon code length to a maximum of 30 characters.
- Added strict whitespace and format checks to reject malformed or fuzzing payloads early before database querying.

### Webhook Security (`app/routes/webhook_routes.py`)
- Enforced mandatory `Stripe-Signature` headers on all incoming webhook requests.
- Removed default mock signature fallbacks, returning HTTP 400 immediately if signature headers are absent or forged.

### Security and Concurrency Test Suite (`tests/test_concurrency_and_security.py`)
- Created a dedicated security test module containing 6 automated tests:
  - **10-Thread Flash Sale Race Condition Test**: Verified that when 10 threads attempt to purchase the final unit of stock simultaneously, exactly 1 succeeds and 9 receive clean 409 errors. Total stock finishes at exactly 0.
  - **SQL Injection Fuzzing Test**: Verified parameterized queries prevent SQL injection payloads in search and cart routes.
  - **XSS Sanitization Test**: Verified script injection payloads in customer addresses are neutralized.
  - **Promo Code Fuzzing Test**: Confirmed long strings and malicious characters are rejected.
  - **Webhook Forgery Test**: Verified forged signatures receive HTTP 400 responses.

---

## 3. Key Takeaways and Next Steps

- **Atomic Queries Prevent Race Conditions:** Never separate the check from the update in high-concurrency environments. Combining them into a single conditional query eliminates overselling.
- **Configure SQLite Timeouts for Concurrency:** Setting `PRAGMA busy_timeout` gives SQLite the resilience needed to handle bursty concurrent web traffic smoothly.
- **Ready for Day 7:** With concurrency hardened and security verified, Day 7 will focus on final storefront polish, bug fixes, master portal integration, and project handover.
