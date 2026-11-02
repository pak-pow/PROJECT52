---
date: 2026-11-02
project: PulseSub Mobile-First SaaS Subscription & Entitlements Platform
topic: Day 3 - Feature Entitlements and Metered Usage Engine
Tags:
  - "[[SaaS-Billing]]"
  - "[[Entitlements-Engine]]"
  - "[[Metered-Usage]]"
  - "[[Route-Decorators]]"
  - "[[Dev Log]]"
---
# DEV LOG: WEEK 45 - DAY 3

**Core Objective:** Build the feature entitlements gatekeeping engine (`EntitlementsService`), construct the high-throughput metered resource accumulator with idempotent deduplication (`UsageService`), and implement route gatekeeping decorators (`@require_entitlement` and `@check_quota`) backed by 71 passing tests with 95% code coverage and 0 linter/security warnings.

---

## 1. The Big Picture and Simple Explanation

In modern Software-as-a-Service platforms, access control extends beyond coarse-grained role-based permissions (admin vs user). The system must continuously evaluate **entitlements**:
1. **Boolean Feature Toggles:** Can this user configure custom domains, invite team members, or export CSV audit logs? If their subscription tier lacks the required flag, the request must be intercepted with HTTP 403 Forbidden and a structured upgrade payload pointing to the pricing matrix.
2. **Quantitative Metered Limits:** How many projects can this user create? How many API requests or megabytes of storage can they consume in their current billing cycle? The system must determine their usage within their exact subscription period (`current_period_start` to `current_period_end`).
3. **Capacity Threshold Monitoring:** Waiting until a user hits 100% of their quota leads to unexpected service disruption. The metered usage engine must evaluate capacity thresholds at 80%, 90%, and 100%, injecting transparent quota headers (`X-Quota-Limit`, `X-Quota-Remaining`, `X-Quota-Used`, `X-Quota-Alert`) on every response.
4. **Idempotent Usage Accumulation:** In distributed or retry-heavy environments, network drops can cause API clients to retry consumption requests. Recording usage with an `idempotency_key` ensures that retried requests never double-count against the user's quota.

In Day 3, we developed three foundational services for **PulseSub**:
- **UsageService:** Manages discrete consumption records, prevents duplicate increments via unique idempotency keys, sums usage over active billing intervals, and computes capacity percentages.
- **EntitlementsService:** Resolves active subscriptions or falls back seamlessly to the Free tier plan. Provides instant boolean feature checks and quantitative quota evaluations.
- **Gatekeeping Decorators:** Python function decorators `@require_entitlement` (returning HTTP 403) and `@check_quota` (returning HTTP 402 with overage payloads and transparent response headers).

```
+-----------------------------------------------------------------------------+
|                     PulseSub Entitlements Resolution Flow                   |
|                                                                             |
|      Client Request  ----->  @require_entitlement("custom_domains")         |
|                                         |                                   |
|                                         v                                   |
|                               EntitlementsService                           |
|                                         |                                   |
|                          +--------------+--------------+                    |
|                          |                             |                    |
|                          v                             v                    |
|                 Active Subscription?           No Active Sub?               |
|                          |                             |                    |
|                          v                             v                    |
|                 Resolve Plan (Pro)           Fallback (Free Plan)           |
|                          |                             |                    |
|                          v                             v                    |
|                 "custom_domains" in?          "custom_domains" in?          |
|                      YES (True)                     NO (False)              |
|                          |                             |                    |
|                          v                             v                    |
|                  Allow Request (200)          HTTP 403 Forbidden            |
|                                               {error: "feature_not_         |
|                                                entitled", upgrade: "/#plans"|
+-----------------------------------------------------------------------------+
```

---

## 2. Key Engineering Implementations

### Idempotent Metered Usage Accumulation
To ensure reliable billing under network retries, `UsageService.record_usage` inspects the unique `idempotency_key`. If the key already exists in SQLite, the existing record is returned without incrementing usage:

```python
def record_usage(
    self,
    user_id: str,
    metric_key: str,
    quantity: int = 1,
    subscription_id: Optional[str] = None,
    idempotency_key: Optional[str] = None,
    recorded_at: Optional[datetime] = None,
) -> UsageRecord:
    if quantity <= 0:
        raise ValueError(f"Quantity must be strictly positive, got {quantity}.")

    now = recorded_at or datetime.now(timezone.utc)
    iso_now = now.strftime("%Y-%m-%d %H:%M:%S")
    cursor = self.conn.cursor()

    if idempotency_key is not None:
        cursor.execute(
            "SELECT * FROM usage_records WHERE idempotency_key = ?;",
            (idempotency_key,),
        )
        existing_row = cursor.fetchone()
        if existing_row:
            cursor.close()
            return UsageRecord.from_row(existing_row)

    record_id = f"use_{uuid.uuid4().hex[:12]}"
    cursor.execute(
        """
        INSERT INTO usage_records (
            id, user_id, subscription_id, metric_key,
            quantity, idempotency_key, recorded_at
        ) VALUES (?, ?, ?, ?, ?, ?, ?);
        """,
        (record_id, user_id, subscription_id, metric_key, quantity, idempotency_key, iso_now),
    )
    self.conn.commit()
    cursor.close()
    return UsageRecord(...)
```

### Quota Threshold Calculations
Capacity thresholds alert clients proactively at 80%, 90%, and 100% of allowed units:

```python
@staticmethod
def evaluate_quota_status(current_usage: int, limit: int) -> Dict[str, Any]:
    if limit == -1:
        return {
            "usage": current_usage,
            "limit": -1,
            "remaining": -1,
            "percentage": 0.0,
            "is_over_quota": False,
            "threshold_alert": None,
            "unlimited": True,
        }

    remaining = max(0, limit - current_usage)
    percentage = round((current_usage / limit) * 100, 1) if limit > 0 else 100.0
    is_over_quota = current_usage >= limit

    threshold_alert: Optional[str] = None
    if current_usage >= limit:
        threshold_alert = "100%"
    elif limit > 0 and current_usage >= (limit * 0.9):
        threshold_alert = "90%"
    elif limit > 0 and current_usage >= (limit * 0.8):
        threshold_alert = "80%"

    return {
        "usage": current_usage,
        "limit": limit,
        "remaining": remaining,
        "percentage": min(100.0, percentage),
        "is_over_quota": is_over_quota,
        "threshold_alert": threshold_alert,
        "unlimited": False,
    }
```

### Route Gatekeeping Decorators
The decorators seamlessly intercept requests, evaluate entitlements via SQLite, and inject transparent telemetry headers into HTTP responses:

```python
@check_quota("monthly_api_calls", cost=5, record=True)
def my_api_endpoint():
    return jsonify({"data": "processed"})
```

If the quota is exceeded, the client receives:
```json
HTTP/1.1 402 Payment Required
Content-Type: application/json

{
    "error": "quota_exceeded",
    "message": "Quota limit exceeded for 'monthly_api_calls'.",
    "metric": "monthly_api_calls",
    "current_usage": 50000,
    "limit": 50000,
    "requested": 5,
    "upgrade_url": "/#plans"
}
```

---

## 3. Verification & Quality Gates

All 5 quality gates passed with zero warnings:

1. **Pytest Suite:** 71/71 tests passing in 1.76s across connection, schema, seed, models, FSM, proration, invoices, usage service, entitlements service, and decorators.
2. **Code Coverage:** 95% line coverage across the entire codebase.
3. **Flake8 Linter:** 0 errors at 88-char limits.
4. **Black & isort:** 100% compliant formatting.
5. **Bandit Security Audit:** 0 vulnerabilities across 1,697 LOC.

```text
============================= test session starts =============================
collected 71 items

tests/test_connection.py::test_get_db_connection_in_memory PASSED        [  1%]
tests/test_connection.py::test_get_db_connection_file_based PASSED       [  2%]
tests/test_connection.py::test_row_factory_access PASSED                 [  4%]
tests/test_connection.py::test_close_db_connection PASSED                [  5%]
tests/test_connection.py::test_init_db_missing_schema_raises PASSED      [  7%]
tests/test_decorators.py::test_require_entitlement_granted PASSED        [  8%]
tests/test_decorators.py::test_require_entitlement_denied PASSED         [  9%]
tests/test_decorators.py::test_require_entitlement_denied_for_enterprise PASSED [ 11%]
tests/test_decorators.py::test_check_quota_permitted_and_headers_injected PASSED [ 12%]
tests/test_decorators.py::test_check_quota_exceeded_returns_402 PASSED   [ 14%]
tests/test_entitlements_service.py::test_get_active_subscription_demo_user PASSED [ 15%]
tests/test_entitlements_service.py::test_get_user_plan_subscribed_and_free PASSED [ 16%]
tests/test_entitlements_service.py::test_check_feature_access PASSED     [ 18%]
tests/test_entitlements_service.py::test_can_consume_metric PASSED       [ 19%]
tests/test_entitlements_service.py::test_get_user_entitlements_snapshot PASSED [ 21%]
tests/test_health.py::test_health_endpoint_healthy PASSED                [ 22%]
tests/test_health.py::test_health_degraded_when_db_fails PASSED          [ 23%]
tests/test_health.py::test_security_headers_present PASSED               [ 25%]
tests/test_health.py::test_static_index_route PASSED                     [ 26%]
tests/test_health.py::test_static_css_route PASSED                       [ 28%]
tests/test_health.py::test_static_js_route PASSED                        [ 29%]
tests/test_health.py::test_config_retrieval PASSED                       [ 30%]
tests/test_invoice_service.py::test_sequential_invoice_numbering PASSED  [ 32%]
tests/test_invoice_service.py::test_create_auto_pay_invoice PASSED       [ 33%]
tests/test_invoice_service.py::test_create_open_invoice PASSED           [ 35%]
tests/test_invoice_service.py::test_create_proration_invoice PASSED      [ 36%]
tests/test_invoice_service.py::test_mark_as_paid PASSED                  [ 38%]
tests/test_invoice_service.py::test_mark_as_failed_and_uncollectible PASSED [ 39%]
tests/test_invoice_service.py::test_void_invoice PASSED                  [ 40%]
tests/test_invoice_service.py::test_get_invoices_by_user PASSED          [ 42%]
tests/test_invoice_service.py::test_nonexistent_invoice_operations PASSED [ 43%]
tests/test_models.py::test_plan_serialization PASSED                     [ 45%]
tests/test_models.py::test_subscription_serialization PASSED             [ 46%]
tests/test_models.py::test_invoice_serialization_and_from_row PASSED     [ 47%]
tests/test_models.py::test_usage_record_serialization_and_from_row PASSED [ 49%]
tests/test_models.py::test_webhook_event_record_serialization_and_from_row PASSED [ 50%]
tests/test_proration.py::test_proration_mid_cycle_upgrade PASSED         [ 52%]
tests/test_proration.py::test_proration_mid_cycle_downgrade PASSED       [ 53%]
tests/test_proration.py::test_proration_at_start_of_period PASSED        [ 54%]
tests/test_proration.py::test_proration_at_end_of_period PASSED          [ 56%]
tests/test_proration.py::test_proration_free_tier_transitions PASSED     [ 57%]
tests/test_proration.py::test_proration_monthly_to_annual_interval PASSED [ 59%]
tests/test_schema.py::test_schema_tables_exist PASSED                    [ 60%]
tests/test_schema.py::test_schema_indexes_exist PASSED                   [ 61%]
tests/test_schema.py::test_users_unique_email_constraint PASSED          [ 63%]
tests/test_schema.py::test_plans_tier_and_price_check_constraints PASSED [ 64%]
tests/test_schema.py::test_foreign_key_cascade_deletion PASSED           [ 66%]
tests/test_seed.py::test_seed_plans_created PASSED                       [ 67%]
tests/test_seed.py::test_seed_database_idempotency PASSED                [ 69%]
tests/test_seed.py::test_demo_user_and_subscription PASSED               [ 70%]
tests/test_seed.py::test_hash_password_deterministic PASSED              [ 71%]
tests/test_subscription_fsm.py::test_fsm_initial_status PASSED           [ 73%]
tests/test_subscription_fsm.py::test_fsm_mark_past_due PASSED            [ 74%]
tests/test_subscription_fsm.py::test_fsm_activate_from_past_due PASSED   [ 76%]
tests/test_subscription_fsm.py::test_fsm_cancel_at_period_end PASSED     [ 77%]
tests/test_subscription_fsm.py::test_fsm_cancel_immediate PASSED         [ 78%]
tests/test_subscription_fsm.py::test_fsm_resume_scheduled_cancel PASSED  [ 80%]
tests/test_subscription_fsm.py::test_fsm_resume_immediate_canceled_before_expiry PASSED [ 81%]
tests/test_subscription_fsm.py::test_fsm_resume_immediate_canceled_after_expiry_raises PASSED [ 83%]
tests/test_subscription_fsm.py::test_fsm_resume_active_non_canceled_raises PASSED [ 84%]
tests/test_subscription_fsm.py::test_fsm_expire_terminal_state PASSED    [ 85%]
tests/test_subscription_fsm.py::test_fsm_renew_active PASSED             [ 87%]
tests/test_subscription_fsm.py::test_fsm_change_tier PASSED              [ 88%]
tests/test_subscription_fsm.py::test_fsm_database_persistence PASSED     [ 90%]
tests/test_usage_service.py::test_record_usage_standard PASSED           [ 91%]
tests/test_usage_service.py::test_record_usage_idempotency PASSED        [ 92%]
tests/test_usage_service.py::test_record_usage_invalid_quantity PASSED   [ 94%]
tests/test_usage_service.py::test_get_period_usage PASSED                [ 95%]
tests/test_usage_service.py::test_get_user_metric_usage_with_subscription PASSED [ 97%]
tests/test_usage_service.py::test_evaluate_quota_status_unlimited PASSED [ 98%]
tests/test_usage_service.py::test_evaluate_quota_status_threshold_alerts PASSED [100%]

=============================== tests coverage ================================
TOTAL: 784 statements, 40 misses, 95% coverage
============================= 71 passed in 1.76s ==============================
```

---

## 4. Next Steps

With Day 3 entitlements and metered usage complete, **Day 4** will implement:
- Stripe-compatible webhook verification service with cryptographic HMAC-SHA256 signature checking.
- Timestamp tolerance window (300 seconds) against replay attacks.
- Idempotent event dispatcher and webhook event handlers (`customer.subscription.updated`, `invoice.paid`, `invoice.payment_failed`).
