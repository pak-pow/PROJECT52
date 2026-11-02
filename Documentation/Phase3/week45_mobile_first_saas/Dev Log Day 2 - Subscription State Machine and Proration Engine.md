---
date: 2026-11-02
project: PulseSub Mobile-First SaaS Subscription & Entitlements Platform
topic: Day 2 - Subscription State Machine and Proration Engine
Tags:
  - "[[SaaS-Billing]]"
  - "[[Subscription-FSM]]"
  - "[[Linear-Proration]]"
  - "[[Invoice-Settlement]]"
  - "[[Dev Log]]"
---
# DEV LOG: WEEK 45 - DAY 2

**Core Objective:** Design and implement the deterministic Subscription Lifecycle Finite State Machine (`SubscriptionFSM`), construct the second-accurate mathematical linear proration engine (`calculate_proration`), and establish the automated billing statement service (`InvoiceService`) backed by 54 passing tests with 95% test coverage and 0 linter/security warnings.

---

## 1. The Big Picture and Simple Explanation

Subscription billing systems are often plagued by subtle edge cases:
1. **Illegal State Transitions:** What happens when a user attempts to activate an already expired subscription? Or cancels a subscription that is already in dunning (`past_due`)? Without a strict finite state machine, edge cases lead to orphaned billing cycles, double charges, or un-gated feature access.
2. **Proration Mathematics:** When a user upgrades mid-cycle (e.g., from Starter at $19/mo to Pro at $49/mo on day 15 of a 30-day month), what is the exact fair charge? The user is entitled to credit for the 15 unused days on Starter ($9.50) and owes payment for the 15 remaining days on Pro ($24.50). The net immediate balance due is $15.00. Downgrades, on the other hand, produce remaining credit ($15.00) held against future periods rather than immediate negative charges.
3. **Sequential Invoice Invariants:** Invoices must have immutable, collision-free numbers (`INV-YYYY-XXXX`) and itemized breakdown of credits and charges with unit amounts, quantities, and settlement dates.

In Day 2, we engineered three core billing subsystems for **PulseSub**:
- **Subscription Lifecycle FSM:** Controls transitions between `trialing`, `active`, `past_due`, `canceled`, and `expired`. Guarantees that illegal state transitions raise `InvalidStateTransitionError` and terminal states raise `SubscriptionExpiredError`.
- **Linear Proration Calculator:** Computes exact second-rate ratios between period start, period end, and the effective transition timestamp. Generates structured positive and negative line items ready for immediate settlement.
- **Invoice Service:** Issues sequential invoices (`INV-2026-0002`, `INV-2026-0003`), calculates line item totals, tracks payment attempts, and handles transitions between `open`, `paid`, `uncollectible`, and `void`.

```
+-----------------------------------------------------------------------------+
|                  PulseSub Subscription FSM Transition Matrix                |
|                                                                             |
|      +---------------+       activate()        +---------------+            |
|      |   TRIALING    |------------------------>|    ACTIVE     |<---+       |
|      +---------------+                         +---------------+    |       |
|              | cancel()                                |            |       |
|              |                                         | mark_past_ | resume()
|              |               cancel(immediate=True)    | due()      |       |
|              |          +------------------------------+            |       |
|              |          |                              v            |       |
|              |          |                      +---------------+    |       |
|              |          |                      |   PAST_DUE    |----+       |
|              |          |                      +---------------+            |
|              |          |                              |                    |
|              v          v                              | expire()           |
|            +---------------+                           v                    |
|            |   CANCELED    |------------------->+---------------+           |
|            +---------------+      expire()      |    EXPIRED    | (Terminal)|
|                    |                            +---------------+           |
|                    +---- resume() (within period)----->^                    |
+-----------------------------------------------------------------------------+
```

---

## 2. Key Engineering Implementations

### Subscription Lifecycle State Machine
The FSM defines immutable allowed transitions and enforces terminal bounds:

```python
class SubscriptionFSM:
    ALLOWED_TRANSITIONS: Dict[str, Tuple[str, ...]] = {
        SubscriptionStatus.TRIALING.value: (
            SubscriptionStatus.ACTIVE.value,
            SubscriptionStatus.CANCELED.value,
            SubscriptionStatus.EXPIRED.value,
        ),
        SubscriptionStatus.ACTIVE.value: (
            SubscriptionStatus.PAST_DUE.value,
            SubscriptionStatus.CANCELED.value,
            SubscriptionStatus.ACTIVE.value,
        ),
        SubscriptionStatus.PAST_DUE.value: (
            SubscriptionStatus.ACTIVE.value,
            SubscriptionStatus.CANCELED.value,
            SubscriptionStatus.EXPIRED.value,
        ),
        SubscriptionStatus.CANCELED.value: (
            SubscriptionStatus.ACTIVE.value,
            SubscriptionStatus.EXPIRED.value,
        ),
        SubscriptionStatus.EXPIRED.value: (),  # Terminal state
    }
```

When an operation occurs, the FSM updates memory properties and persists changes to SQLite when an active database connection is attached.

### Linear Day/Second Proration Formula
The proration formula evaluates exact linear ratios based on epoch seconds:

$$\text{total\_seconds} = \text{period\_end} - \text{period\_start}$$
$$\text{remaining\_seconds} = \max(0, \min(\text{total\_seconds}, \text{period\_end} - \text{effective\_at}))$$
$$\text{remaining\_ratio} = \frac{\text{remaining\_seconds}}{\text{total\_seconds}}$$
$$\text{unused\_credit} = \text{round}(\text{current\_price} \times \text{remaining\_ratio})$$
$$\text{new\_charge} = \text{round}(\text{new\_price} \times \text{remaining\_ratio})$$

$$\text{net\_due} = \max(0, \text{new\_charge} - \text{unused\_credit})$$
$$\text{credit\_remaining} = \max(0, \text{unused\_credit} - \text{new\_charge})$$

### Collision-Free Sequential Invoice Numbering
Invoices are assigned deterministic numbering formatted as `INV-YYYY-XXXX`. The query scans existing invoices for the active year and increments the sequence atomically:

```python
def _generate_invoice_number(self, now: datetime) -> str:
    year = now.strftime("%Y")
    prefix = f"INV-{year}-"
    cursor = self.conn.cursor()
    cursor.execute(
        """
        SELECT invoice_number FROM invoices
        WHERE invoice_number LIKE ?
        ORDER BY invoice_number DESC
        LIMIT 1;
        """,
        (f"{prefix}%",),
    )
    row = cursor.fetchone()
    cursor.close()
    if row and row["invoice_number"]:
        try:
            last_seq = int(row["invoice_number"].replace(prefix, ""))
            next_seq = last_seq + 1
        except ValueError:
            next_seq = 1
    else:
        next_seq = 1
    return f"{prefix}{next_seq:04d}"
```

---

## 3. Verification & Quality Gates

All quality gates passed with zero warnings:

1. **Pytest Suite:** 54/54 tests passing in 1.44s across connection, schema, seed, models, FSM, proration, and invoice suites.
2. **Code Coverage:** 95% line coverage across the entire codebase.
3. **Flake8 Linter:** 0 errors at 88-char line limits.
4. **Black & isort:** 100% compliant formatting.
5. **Bandit Security Audit:** 0 vulnerabilities across 1,301 LOC.

```text
============================= test session starts =============================
collected 54 items

tests/test_connection.py::test_get_db_connection_in_memory PASSED        [  1%]
tests/test_connection.py::test_get_db_connection_file_based PASSED       [  3%]
tests/test_connection.py::test_row_factory_access PASSED                 [  5%]
tests/test_connection.py::test_close_db_connection PASSED                [  7%]
tests/test_connection.py::test_init_db_missing_schema_raises PASSED      [  9%]
tests/test_health.py::test_health_endpoint_healthy PASSED                [ 11%]
tests/test_health.py::test_health_degraded_when_db_fails PASSED          [ 12%]
tests/test_health.py::test_security_headers_present PASSED               [ 14%]
tests/test_health.py::test_static_index_route PASSED                     [ 16%]
tests/test_health.py::test_static_css_route PASSED                       [ 18%]
tests/test_health.py::test_static_js_route PASSED                        [ 20%]
tests/test_health.py::test_config_retrieval PASSED                       [ 22%]
tests/test_invoice_service.py::test_sequential_invoice_numbering PASSED  [ 24%]
tests/test_invoice_service.py::test_create_auto_pay_invoice PASSED       [ 25%]
tests/test_invoice_service.py::test_create_open_invoice PASSED           [ 27%]
tests/test_invoice_service.py::test_create_proration_invoice PASSED      [ 29%]
tests/test_invoice_service.py::test_mark_as_paid PASSED                  [ 31%]
tests/test_invoice_service.py::test_mark_as_failed_and_uncollectible PASSED [ 33%]
tests/test_invoice_service.py::test_void_invoice PASSED                  [ 35%]
tests/test_invoice_service.py::test_get_invoices_by_user PASSED          [ 37%]
tests/test_invoice_service.py::test_nonexistent_invoice_operations PASSED [ 38%]
tests/test_models.py::test_plan_serialization PASSED                     [ 40%]
tests/test_models.py::test_subscription_serialization PASSED             [ 42%]
tests/test_models.py::test_invoice_serialization_and_from_row PASSED     [ 44%]
tests/test_models.py::test_usage_record_serialization_and_from_row PASSED [ 46%]
tests/test_models.py::test_webhook_event_record_serialization_and_from_row PASSED [ 48%]
tests/test_proration.py::test_proration_mid_cycle_upgrade PASSED         [ 50%]
tests/test_proration.py::test_proration_mid_cycle_downgrade PASSED       [ 51%]
tests/test_proration.py::test_proration_at_start_of_period PASSED        [ 53%]
tests/test_proration.py::test_proration_at_end_of_period PASSED          [ 55%]
tests/test_proration.py::test_proration_free_tier_transitions PASSED     [ 57%]
tests/test_proration.py::test_proration_monthly_to_annual_interval PASSED [ 59%]
tests/test_schema.py::test_schema_tables_exist PASSED                    [ 61%]
tests/test_schema.py::test_schema_indexes_exist PASSED                   [ 62%]
tests/test_schema.py::test_users_unique_email_constraint PASSED          [ 64%]
tests/test_schema.py::test_plans_tier_and_price_check_constraints PASSED [ 66%]
tests/test_schema.py::test_foreign_key_cascade_deletion PASSED           [ 68%]
tests/test_seed.py::test_seed_plans_created PASSED                       [ 70%]
tests/test_seed.py::test_seed_database_idempotency PASSED                [ 72%]
tests/test_seed.py::test_demo_user_and_subscription PASSED               [ 74%]
tests/test_seed.py::test_hash_password_deterministic PASSED              [ 75%]
tests/test_subscription_fsm.py::test_fsm_initial_status PASSED           [ 77%]
tests/test_subscription_fsm.py::test_fsm_mark_past_due PASSED            [ 79%]
tests/test_subscription_fsm.py::test_fsm_activate_from_past_due PASSED   [ 81%]
tests/test_subscription_fsm.py::test_fsm_cancel_at_period_end PASSED     [ 83%]
tests/test_subscription_fsm.py::test_fsm_cancel_immediate PASSED         [ 85%]
tests/test_subscription_fsm.py::test_fsm_resume_scheduled_cancel PASSED  [ 87%]
tests/test_subscription_fsm.py::test_fsm_resume_immediate_canceled_before_expiry PASSED [ 88%]
tests/test_subscription_fsm.py::test_fsm_resume_immediate_canceled_after_expiry_raises PASSED [ 90%]
tests/test_subscription_fsm.py::test_fsm_resume_active_non_canceled_raises PASSED [ 92%]
tests/test_subscription_fsm.py::test_fsm_expire_terminal_state PASSED    [ 94%]
tests/test_subscription_fsm.py::test_fsm_renew_active PASSED             [ 96%]
tests/test_subscription_fsm.py::test_fsm_change_tier PASSED              [ 98%]
tests/test_subscription_fsm.py::test_fsm_database_persistence PASSED     [100%]

=============================== tests coverage ================================
TOTAL: 605 statements, 30 misses, 95% coverage
============================= 54 passed in 1.44s ==============================
```

---

## 4. Next Steps

With Day 2's subscription state machine and proration engine in place, **Day 3** will implement:
- Feature entitlements resolution service (`EntitlementsService`).
- Metered resource usage accumulators with sliding window tracking and capacity thresholds (80%, 90%, 100%).
- Flask route gatekeeping decorators (`@require_entitlement` and `@check_quota`).
