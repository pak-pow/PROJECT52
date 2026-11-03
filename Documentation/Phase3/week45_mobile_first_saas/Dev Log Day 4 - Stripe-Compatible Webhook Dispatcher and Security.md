---
date: 2026-11-03
project: PulseSub Mobile-First SaaS Subscription & Entitlements Platform
topic: Day 4 - Stripe-Compatible Webhook Dispatcher and Security
Tags:
  - "[[SaaS-Billing]]"
  - "[[Webhook-Gateway]]"
  - "[[HMAC-SHA256]]"
  - "[[Event-Dispatcher]]"
  - "[[Dev Log]]"
---
# DEV LOG: WEEK 45 - DAY 4

**Core Objective:** Construct a Stripe-compatible webhook gateway featuring cryptographic HMAC-SHA256 signature verification, a 300-second timestamp replay defense, and an idempotent event dispatcher supporting core subscription lifecycle events (`customer.subscription.created`, `customer.subscription.updated`, `customer.subscription.deleted`, `invoice.paid`, `invoice.payment_failed`), verified by 90 passing tests with 95% test coverage and 0 security/linter issues.

---

## 1. The Big Picture and Simple Explanation

In SaaS billing systems, state changes rarely originate solely from direct client interactions. External payment processors like Stripe handle recurring renewals, card declines, dispute resolutions, and automated collection retries asynchronously. To keep the local database synchronized with payment processor state without trust vulnerabilities, the application must implement a secure **Webhook Gateway**:

1. **Cryptographic Payload Verification:** Anyone can send an HTTP POST request to a public webhook URL. To ensure authenticity and tamper-resistance, Stripe hashes the timestamped raw payload with a shared secret using HMAC-SHA256. PulseSub validates this signature before parsing or processing the body.
2. **Replay Attack Prevention:** Malicious actors who intercept valid webhook traffic could attempt to replay requests later (e.g., repeatedly firing an invoice failure event). By enforcing a strict timestamp tolerance window (300 seconds), any replayed request with an outdated timestamp is rejected immediately.
3. **Constant-Time Comparison:** Using naive string equality comparisons exposes systems to timing attacks where attackers guess signatures byte-by-byte. PulseSub utilizes constant-time comparison (`hmac.compare_digest`) to prevent timing side channels.
4. **Idempotent Event Handling:** Payment gateways frequently retry webhooks when network hiccups delay acknowledgments. Storing received event IDs (`evt_...`) in a dedicated `webhook_events` table ensures that re-delivered webhooks are safely acknowledged without executing duplicate database mutations or double-crediting accounts.

```
+-----------------------------------------------------------------------------+
|                      PulseSub Webhook Ingestion Flow                        |
|                                                                             |
|      External Payment Processor (Stripe)                                    |
|                |                                                            |
|                | HTTP POST /api/v1/webhooks/stripe                          |
|                | Header: Stripe-Signature: t=1730600000,v1=a1b2c3...        |
|                v                                                            |
|   +---------------------------------------------------------------------+   |
|   | 1. Webhook Security (`verify_webhook_signature`)                    |   |
|   |    - Parse timestamp 't' and signature 'v1'                         |   |
|   |    - Check timestamp tolerance: |now - t| <= 300s                   |   |
|   |    - Compute HMAC-SHA256(secret, f"{t}.{raw_body}")                |   |
|   |    - Constant-time compare: hmac.compare_digest                     |   |
|   +----------------------------------+----------------------------------+   |
|                                      |                                      |
|                                [Valid Signature]                            |
|                                      v                                      |
|   +---------------------------------------------------------------------+   |
|   | 2. Event Recording & Deduplication (`WebhookDispatcher`)            |   |
|   |    - Check `webhook_events` table for event_id                      |   |
|   |    - If already processed: return 200 {status: "already_processed"} |   |
|   |    - If new: INSERT into `webhook_events` with status 'pending'     |   |
|   +----------------------------------+----------------------------------+   |
|                                      |                                      |
|                                      v                                      |
|   +---------------------------------------------------------------------+   |
|   | 3. Event Dispatcher (`dispatch_event`)                              |   |
|   |    - customer.subscription.created  -> Update local subscription    |   |
|   |    - customer.subscription.updated  -> Sync status & renew dates    |   |
|   |    - customer.subscription.deleted  -> Mark canceled/expired        |   |
|   |    - invoice.paid                  -> Mark invoice paid & activate  |   |
|   |    - invoice.payment_failed        -> Transition to past_due        |   |
|   +----------------------------------+----------------------------------+   |
|                                      |                                      |
|                                      v                                      |
|   +---------------------------------------------------------------------+   |
|   | 4. Finalize & Acknowledge                                           |   |
|   |    - UPDATE `webhook_events` SET status = 'processed'               |   |
|   |    - Return HTTP 200 {received: true, status: "processed"}          |   |
|   +---------------------------------------------------------------------+   |
+-----------------------------------------------------------------------------+
```

---

## 2. Key Engineering Implementations

### Cryptographic Signature Verification (`webhook_security.py`)
PulseSub parses the `Stripe-Signature` header, extracts the UNIX timestamp and signature hash, verifies the replay window tolerance, and generates an HMAC-SHA256 digest over the concatenation of `f"{timestamp}.{raw_payload}"`:

```python
def verify_webhook_signature(
    payload: Union[str, bytes],
    signature_header: str,
    secret: str,
    tolerance_seconds: int = 300,
    current_time: Optional[int] = None,
) -> bool:
    if not signature_header:
        raise WebhookSignatureVerificationError("Missing Stripe-Signature header.")

    elements = [item.strip() for item in signature_header.split(",") if item.strip()]
    timestamp_str: Optional[str] = None
    v1_signatures: list[str] = []

    for elem in elements:
        parts = elem.split("=", 1)
        if len(parts) == 2:
            key, val = parts[0].strip(), parts[1].strip()
            if key == "t":
                timestamp_str = val
            elif key == "v1":
                v1_signatures.append(val)

    if not timestamp_str:
        raise WebhookSignatureVerificationError("Timestamp 't' not found in header.")
    if not v1_signatures:
        raise WebhookSignatureVerificationError("No 'v1' signatures found in header.")

    timestamp = int(timestamp_str)
    now = current_time if current_time is not None else int(time.time())
    if abs(now - timestamp) > tolerance_seconds:
        raise WebhookSignatureVerificationError(
            f"Timestamp outside tolerance window ({tolerance_seconds}s)."
        )

    signed_payload = f"{timestamp}.{raw_str}".encode("utf-8")
    expected_sig = hmac.new(
        secret.encode("utf-8"),
        signed_payload,
        hashlib.sha256,
    ).hexdigest()

    matched = any(
        hmac.compare_digest(expected_sig, candidate)
        for candidate in v1_signatures
    )
    if not matched:
        raise WebhookSignatureVerificationError("Signature verification failed.")
    return True
```

### Idempotent Event Dispatcher (`webhook_dispatcher.py`)
To prevent processing duplicate events while handling state mutations atomically:
- Handled events record state transitions directly onto local `subscriptions` and `invoices`.
- Failed invoices trigger transitions to `past_due` via `SubscriptionFSM`.
- Paid invoices trigger transitions back to `active`.
- Unhandled events are logged and marked as `ignored` with HTTP 200 OK to prevent webhook retries from blocking delivery queues.

```python
def dispatch_event(self, event_data: dict[str, Any]) -> dict[str, Any]:
    event_id = event_data.get("id")
    event_type = event_data.get("type")
    # Step 1: Idempotency check in webhook_events
    existing = self.get_event_by_id(event_id)
    if existing and existing.status == "processed":
        return {"status": "already_processed", "event_id": event_id}

    # Step 2: Route to appropriate handler
    handler = self._handlers.get(event_type)
    if not handler:
        self.mark_event_status(event_id, "ignored")
        return {"status": "ignored", "event_id": event_id}

    result = handler(event_data.get("data", {}).get("object", {}))
    self.mark_event_status(event_id, "processed")
    return {"status": "processed", "event_id": event_id, "result": result}
```

### REST Webhook Endpoints (`routes/webhooks.py`)
Exposes `POST /api/v1/webhooks/stripe` with raw request payload verification and `GET /api/v1/webhooks/events` for audit inspection:
- Validates the presence of `STRIPE_WEBHOOK_SECRET`.
- Handles signature errors with HTTP 400 Bad Request.
- Returns JSON execution reports for client tracking.

---

## 3. Verification & Quality Assurance

All quality gates passed with zero warnings:
1. **Pytest Test Suite:** 90 tests passing in 1.90s.
2. **Code Coverage:** 95% line coverage across the entire codebase.
3. **Flake8 Linter:** 0 errors at 88-char limits.
4. **Black & isort:** 100% compliant formatting.
5. **Bandit Security Audit:** 0 vulnerabilities across 2,137 LOC.

```text
============================= test session starts =============================
collected 90 items

tests/test_connection.py::test_get_db_connection_in_memory PASSED        [  1%]
tests/test_connection.py::test_get_db_connection_file_based PASSED       [  2%]
tests/test_connection.py::test_row_factory_access PASSED                 [  3%]
tests/test_connection.py::test_close_db_connection PASSED                [  4%]
tests/test_connection.py::test_init_db_missing_schema_raises PASSED      [  5%]
tests/test_decorators.py::test_require_entitlement_granted PASSED        [  6%]
tests/test_decorators.py::test_require_entitlement_denied PASSED         [  7%]
tests/test_decorators.py::test_require_entitlement_denied_for_enterprise PASSED [  8%]
tests/test_decorators.py::test_check_quota_permitted_and_headers_injected PASSED [ 10%]
tests/test_decorators.py::test_check_quota_exceeded_returns_402 PASSED   [ 11%]
tests/test_entitlements_service.py::test_get_active_subscription_demo_user PASSED [ 12%]
tests/test_entitlements_service.py::test_get_user_plan_subscribed_and_free PASSED [ 13%]
tests/test_entitlements_service.py::test_check_feature_access PASSED     [ 14%]
tests/test_entitlements_service.py::test_can_consume_metric PASSED       [ 15%]
tests/test_entitlements_service.py::test_get_user_entitlements_snapshot PASSED [ 16%]
tests/test_health.py::test_health_endpoint_healthy PASSED                [ 17%]
tests/test_health.py::test_health_degraded_when_db_fails PASSED          [ 18%]
tests/test_health.py::test_security_headers_present PASSED               [ 20%]
tests/test_health.py::test_static_index_route PASSED                     [ 21%]
tests/test_health.py::test_static_css_route PASSED                       [ 22%]
tests/test_health.py::test_static_js_route PASSED                        [ 23%]
tests/test_health.py::test_config_retrieval PASSED                       [ 24%]
tests/test_invoice_service.py::test_sequential_invoice_numbering PASSED  [ 25%]
tests/test_invoice_service.py::test_create_auto_pay_invoice PASSED       [ 26%]
tests/test_invoice_service.py::test_create_open_invoice PASSED           [ 27%]
tests/test_invoice_service.py::test_create_proration_invoice PASSED      [ 28%]
tests/test_invoice_service.py::test_mark_as_paid PASSED                  [ 30%]
tests/test_invoice_service.py::test_mark_as_failed_and_uncollectible PASSED [ 31%]
tests/test_invoice_service.py::test_void_invoice PASSED                  [ 32%]
tests/test_invoice_service.py::test_get_invoices_by_user PASSED          [ 33%]
tests/test_invoice_service.py::test_nonexistent_invoice_operations PASSED [ 34%]
tests/test_models.py::test_plan_serialization PASSED                     [ 35%]
tests/test_models.py::test_subscription_serialization PASSED             [ 36%]
tests/test_models.py::test_invoice_serialization_and_from_row PASSED     [ 37%]
tests/test_models.py::test_usage_record_serialization_and_from_row PASSED [ 38%]
tests/test_models.py::test_webhook_event_record_serialization_and_from_row PASSED [ 40%]
tests/test_proration.py::test_proration_mid_cycle_upgrade PASSED         [ 41%]
tests/test_proration.py::test_proration_mid_cycle_downgrade PASSED       [ 42%]
tests/test_proration.py::test_proration_at_start_of_period PASSED        [ 43%]
tests/test_proration.py::test_proration_at_end_of_period PASSED          [ 44%]
tests/test_proration.py::test_proration_free_tier_transitions PASSED     [ 45%]
tests/test_proration.py::test_proration_monthly_to_annual_interval PASSED [ 46%]
tests/test_schema.py::test_schema_tables_exist PASSED                    [ 47%]
tests/test_schema.py::test_schema_indexes_exist PASSED                   [ 48%]
tests/test_schema.py::test_users_unique_email_constraint PASSED          [ 50%]
tests/test_schema.py::test_plans_tier_and_price_check_constraints PASSED [ 51%]
tests/test_schema.py::test_foreign_key_cascade_deletion PASSED           [ 52%]
tests/test_seed.py::test_seed_plans_created PASSED                       [ 53%]
tests/test_seed.py::test_seed_database_idempotency PASSED                [ 54%]
tests/test_seed.py::test_demo_user_and_subscription PASSED               [ 55%]
tests/test_seed.py::test_hash_password_deterministic PASSED              [ 56%]
tests/test_subscription_fsm.py::test_fsm_initial_status PASSED           [ 57%]
tests/test_subscription_fsm.py::test_fsm_mark_past_due PASSED            [ 58%]
tests/test_subscription_fsm.py::test_fsm_activate_from_past_due PASSED   [ 60%]
tests/test_subscription_fsm.py::test_fsm_cancel_at_period_end PASSED     [ 61%]
tests/test_subscription_fsm.py::test_fsm_cancel_immediate PASSED         [ 62%]
tests/test_subscription_fsm.py::test_fsm_resume_scheduled_cancel PASSED  [ 63%]
tests/test_subscription_fsm.py::test_fsm_resume_immediate_canceled_before_expiry PASSED [ 64%]
tests/test_subscription_fsm.py::test_fsm_resume_immediate_canceled_after_expiry_raises PASSED [ 65%]
tests/test_subscription_fsm.py::test_fsm_resume_active_non_canceled_raises PASSED [ 66%]
tests/test_subscription_fsm.py::test_fsm_expire_terminal_state PASSED    [ 67%]
tests/test_subscription_fsm.py::test_fsm_renew_active PASSED             [ 68%]
tests/test_subscription_fsm.py::test_fsm_change_tier PASSED              [ 70%]
tests/test_subscription_fsm.py::test_fsm_database_persistence PASSED     [ 71%]
tests/test_usage_service.py::test_record_usage_standard PASSED           [ 72%]
tests/test_usage_service.py::test_record_usage_idempotency PASSED        [ 73%]
tests/test_usage_service.py::test_record_usage_invalid_quantity PASSED   [ 74%]
tests/test_usage_service.py::test_get_period_usage PASSED                [ 75%]
tests/test_usage_service.py::test_get_user_metric_usage_with_subscription PASSED [ 76%]
tests/test_usage_service.py::test_evaluate_quota_status_unlimited PASSED [ 77%]
tests/test_usage_service.py::test_evaluate_quota_status_threshold_alerts PASSED [ 78%]
tests/test_webhook_dispatcher.py::test_subscription_created_handler PASSED [ 80%]
tests/test_webhook_dispatcher.py::test_subscription_updated_handler PASSED [ 81%]
tests/test_webhook_dispatcher.py::test_subscription_deleted_handler PASSED [ 82%]
tests/test_webhook_dispatcher.py::test_invoice_paid_handler PASSED        [ 83%]
tests/test_webhook_dispatcher.py::test_invoice_payment_failed_handler PASSED [ 84%]
tests/test_webhook_dispatcher.py::test_idempotency_prevents_duplicate_processing PASSED [ 85%]
tests/test_webhook_dispatcher.py::test_unknown_event_type_handled PASSED  [ 86%]
tests/test_webhook_dispatcher.py::test_event_without_id_generates_id PASSED [ 87%]
tests/test_webhook_routes.py::test_webhook_missing_signature_returns_400 PASSED [ 88%]
tests/test_webhook_routes.py::test_webhook_invalid_signature_returns_400 PASSED [ 90%]
tests/test_webhook_routes.py::test_webhook_valid_signature_processes_event PASSED [ 91%]
tests/test_webhook_routes.py::test_webhook_missing_secret_returns_500 PASSED [ 92%]
tests/test_webhook_routes.py::test_list_webhook_events PASSED             [ 93%]
tests/test_webhook_security.py::test_verify_valid_signature PASSED       [ 94%]
tests/test_webhook_security.py::test_verify_missing_header_raises PASSED [ 95%]
tests/test_webhook_security.py::test_verify_malformed_header_missing_parts PASSED [ 96%]
tests/test_webhook_security.py::test_verify_timestamp_out_of_tolerance PASSED [ 97%]
tests/test_webhook_security.py::test_verify_invalid_signature_hash PASSED [ 98%]
tests/test_webhook_security.py::test_generate_webhook_signature PASSED   [100%]

=============================== tests coverage ================================
Name                                Stmts   Miss  Cover   Missing
-----------------------------------------------------------------
app\__init__.py                        38      0   100%
app\billing\__init__.py                 8      0   100%
app\billing\invoice_service.py         96      8    92%   99-100, 118, 146, 178, 206, 237, 269
app\billing\proration.py               70      1    99%   138
app\billing\subscription_fsm.py       111      6    95%   85, 126, 151, 169, 192, 225
app\billing\webhook_dispatcher.py      95      8    92%   90-91, 125-126, 156-157, 184-185
app\billing\webhook_security.py        42      1    98%   46
app\config.py                          34      1    97%   47
app\db\__init__.py                      0      0   100%
app\db\connection.py                   43      3    93%   38-39, 57
app\models.py                          88      0   100%
app\routes\__init__.py                  2      0   100%
app\routes\health.py                   21      0   100%
app\routes\webhooks.py                 43      4    91%   49, 78-79, 93
app\services\__init__.py                4      0   100%
app\services\decorators.py             55      4    93%   40, 52, 98, 111
app\services\entitlements_service.py   49      2    96%   68-69
app\services\usage_service.py          73      5    93%   109, 137, 161, 183, 207
data\seed.py                           64      3    95%   158-160
-----------------------------------------------------------------
TOTAL                                 986     46    95%
============================= 90 passed in 1.90s ==============================
```

---

## 4. Next Steps

With Day 4 webhook ingestion and cryptographic verification complete, **Day 5** will implement:
- Customer REST API Gateway for authentication, subscription control, and billing management.
- Routes: `/api/v1/auth/login`, `/api/v1/subscriptions/current`, `/api/v1/subscriptions/subscribe`, `/api/v1/subscriptions/change-tier`, `/api/v1/subscriptions/cancel`, `/api/v1/billing/invoices`.
- Entitlement inspection endpoints (`/api/v1/entitlements/features`, `/api/v1/usage/summary`).
- Complete integration tests and verification for Day 5.
