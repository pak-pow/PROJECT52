---
date: 2026-11-05
project: PulseSub Mobile-First SaaS Subscription & Entitlements Platform
topic: Day 6 - Mobile-First Responsive App and Sandbox Console
Tags:
  - "[[Mobile-First]]"
  - "[[Frontend-Architecture]]"
  - "[[Device-Simulator]]"
  - "[[Web-Crypto-API]]"
  - "[[Webhook-Console]]"
  - "[[Dev Log]]"
---
# DEV LOG: WEEK 45 - DAY 6

**Core Objective:** Build a native-feeling, mobile-first responsive frontend application shell featuring a bottom navigation bar, a desktop viewport simulator for mobile form-factors (iPhone 15, Pixel 8, iPad), interactive pricing matrix with live proration previews, metered resource gauges with capacity alert badges (80%, 90%, 100%), customer billing portal, and an interactive Stripe webhook sandbox console powered by the browser Web Crypto API.

---

## 1. The Big Picture and Simple Explanation

A SaaS subscription platform is only as effective as the subscriber's ability to manage their plan and visualize resource consumption on the go. Traditional enterprise billing dashboards are cluttered and desktop-centric. PulseSub is built mobile-first from the ground up:

1. **Touch-First Navigation:** A persistent bottom tab bar (Overview, Usage, Plans, Portal, Sandbox) following iOS Human Interface Guidelines and Android Material Design principles with ergonomic 48px touch targets and haptic visual feedback.
2. **Device Viewport Simulator:** For developers and desktop users, an embedded device frame simulator provides realistic viewports:
   - iPhone 15: 393 x 852 px with a dynamic island bezel.
   - Pixel 8: 412 x 915 px with a centered camera cutout.
   - iPad: 820 x 1180 px tablet frame.
   - Fluid Responsive: 100% responsive fluid view for desktop monitors.
3. **Interactive Pricing & Proration:** A pricing matrix featuring a monthly vs. annual billing toggle (highlighting annual savings) and a bottom-sheet modal calculating exact linear proration credits and charges before confirming upgrades.
4. **Real-Time Resource Meters:** Dynamic progress bars visualizing consumption of API calls, team seats, and storage with automated visual capacity alert badges (80% warning, 90% critical, 100% exceeded) and one-click consumption simulators.
5. **Customer Billing Portal:** Self-serve subscription management (current plan, renewal countdown, schedule cancellation, resumption) and invoice statement table with one-click printable HTML receipts.
6. **Stripe Webhook Sandbox Console:** An in-browser developer tool that drafts standard Stripe webhook events (`customer.subscription.created`, `customer.subscription.updated`, `customer.subscription.deleted`, `invoice.payment_succeeded`, `invoice.payment_failed`), signs payloads in real time using the browser Web Crypto API (HMAC-SHA256), and dispatches them to the backend gateway.

```
+-----------------------------------------------------------------------------+
|                     PulseSub Mobile-First Application Shell                 |
|                                                                             |
|   +---------------------------------------------------------------------+   |
|   | Simulator Top Bar: [Viewport: iPhone 15 | Theme: Dark | Demo: Alice]|   |
|   +---------------------------------------------------------------------+   |
|                                                                             |
|            +---------------------------------------------------+            |
|            | iPhone 15 Frame (393px x 852px)                   |            |
|            |                                                   |            |
|            |  +---------------------------------------------+  |            |
|            |  | Dynamic Island / Header Bar                 |  |            |
|            |  +---------------------------------------------+  |            |
|            |  |                                             |  |            |
|            |  |  Active View Container:                     |  |            |
|            |  |  - Overview: Plan status, quick metrics     |  |            |
|            |  |  - Usage: Resource meters, threshold badges |  |            |
|            |  |  - Plans: Matrix, toggle, proration sheet   |  |            |
|            |  |  - Portal: Statements, pay, cancel/resume   |  |            |
|            |  |  - Sandbox: Web Crypto HMAC-SHA256 console  |  |            |
|            |  |                                             |  |            |
|            |  +---------------------------------------------+  |            |
|            |  | Bottom Tab Navigation (O | U | P | B | S)   |  |            |
|            |  +---------------------------------------------+  |            |
|            +---------------------------------------------------+            |
|                                                                             |
+-----------------------------------------------------------------------------+
```

---

## 2. Key Frontend Components and Implementations

### A. Modular Architecture
The frontend is structured without external framework overhead, using modern ES6 modules, CSS custom property tokens, and vanilla DOM manipulation:

```text
frontend/
├── public/
│   └── index.html             # Mobile viewport shell, simulator toolbar, modals
└── src/
    ├── css/
    │   └── base.css           # Design tokens, device frames, responsive media queries
    └── js/
        ├── api.js             # Client API wrapper with Bearer token authentication
        ├── app.js             # Master controller, tab switching, global state
        ├── device_frame.js    # Viewport switcher controller
        ├── portal_view.js     # Subscription management and invoice receipt viewer
        ├── pricing_view.js    # Plan matrix and proration preview bottom-sheet
        ├── sandbox_console.js # Web Crypto HMAC-SHA256 webhook simulator
        └── usage_view.js      # Metered resource gauges and capacity alert badges
```

### B. Device Viewport Simulator (`device_frame.js`)
To facilitate desktop testing of mobile viewport constraints, `DeviceFrame` dynamically updates CSS classes and dimensions on `#device-wrapper`:

```javascript
export const DeviceFrame = {
    devices: {
        iphone15: { width: 393, height: 852, name: "iPhone 15" },
        pixel8: { width: 412, height: 915, name: "Pixel 8" },
        ipad: { width: 820, height: 1180, name: "iPad" },
        fluid: { width: "100%", height: "100%", name: "Fluid Responsive" },
    },
    setDevice(deviceId) {
        const frame = document.getElementById("device-wrapper");
        frame.className = `device-wrapper device-${deviceId}`;
        if (deviceId === "fluid") {
            frame.style.width = "100%";
            frame.style.height = "100%";
            frame.style.maxWidth = "none";
        } else {
            const dev = this.devices[deviceId];
            frame.style.width = `${dev.width}px`;
            frame.style.height = `${dev.height}px`;
            frame.style.maxWidth = `${dev.width}px`;
        }
    }
};
```

### C. Live Linear Proration Bottom-Sheet Modal (`pricing_view.js`)
When an existing subscriber selects a different plan tier or billing interval, `PricingView` calls the dry-run `/api/v1/subscriptions/change-tier` endpoint with `preview_only: true` and renders the exact financial credit/charge breakdown in an accessible bottom-sheet modal:

```javascript
async previewProration(targetPlanId, targetInterval) {
    const res = await Api.changeTier(targetPlanId, targetInterval, true);
    const p = res.proration;
    const formatCents = (c) => `$${(c / 100).toFixed(2)}`;

    const modalBody = document.getElementById("bottom-sheet-body");
    modalBody.innerHTML = `
        <div class="proration-breakdown">
            <h4>Proration Settlement Preview</h4>
            <div class="proration-row">
                <span>Unused Time Credit:</span>
                <span class="credit-val">-${formatCents(p.unused_credit_cents)}</span>
            </div>
            <div class="proration-row">
                <span>New Tier Charge:</span>
                <span>${formatCents(p.new_plan_charge_cents)}</span>
            </div>
            <div class="proration-row total-row">
                <strong>Net Amount Due Today:</strong>
                <strong>${formatCents(p.net_amount_due_cents)}</strong>
            </div>
        </div>
    `;
    this.openBottomSheet();
}
```

### D. In-Browser Web Crypto HMAC-SHA256 Signing (`sandbox_console.js`)
The webhook sandbox console enables developers to test webhook ingress without running external CLI tools like Stripe CLI. It uses the native `window.crypto.subtle` API to compute standard HMAC-SHA256 signatures:

```javascript
async generateSignature(payloadString, secretKey, timestamp) {
    const signedPayload = `${timestamp}.${payloadString}`;
    const encoder = new TextEncoder();
    const keyData = encoder.encode(secretKey);
    const messageData = encoder.encode(signedPayload);

    const cryptoKey = await window.crypto.subtle.importKey(
        "raw",
        keyData,
        { name: "HMAC", hash: "SHA-256" },
        false,
        ["sign"]
    );

    const signatureBuffer = await window.crypto.subtle.sign(
        "HMAC",
        cryptoKey,
        messageData
    );

    const signatureHex = Array.from(new Uint8Array(signatureBuffer))
        .map((b) => b.toString(16).padStart(2, "0"))
        .join("");

    return `t=${timestamp},v1=${signatureHex}`;
}
```

### E. Real-Time Resource Meters with Capacity Alerts (`usage_view.js`)
`UsageView` queries `/api/v1/entitlements/my-features` and `/api/v1/usage/summary` to render live resource gauges. Threshold badges (`80%`, `90%`, `100%`) alert subscribers when approaching or exceeding quotas:

```javascript
renderMeterCard(metricKey, label, status) {
    const isUnlimited = status.unlimited;
    const used = status.usage;
    const limit = status.limit;
    const pct = status.percentage;
    const alertLevel = status.threshold_alert;

    let badgeClass = "badge-normal";
    if (alertLevel === "80%") badgeClass = "badge-warning";
    if (alertLevel === "90%") badgeClass = "badge-critical";
    if (alertLevel === "100%") badgeClass = "badge-exceeded";

    return `
    <div class="meter-card">
        <div class="meter-header">
            <span class="meter-name">${label}</span>
            ${alertLevel ? `<span class="capacity-badge ${badgeClass}">${alertLevel} CAPACITY</span>` : ""}
        </div>
        <div class="meter-bar-track">
            <div class="meter-bar-fill ${badgeClass}" style="width: ${pct}%"></div>
        </div>
        <div class="meter-footer">
            <span>${used} / ${isUnlimited ? "Unlimited" : limit}</span>
            <span>${pct}% used</span>
        </div>
    </div>
    `;
}
```

---

## 3. Strict Quality and Zero-Emoji Compliance

In compliance with project specifications, the frontend code, CSS, HTML, and comments strictly contain 0 emojis. All UI tab bar icons and controls use typography monograms (`O`, `U`, `P`, `B`, `S`) and styled CSS glyphs.

- All static routes verified via `tests/test_health.py` (serving `/`, `/css/base.css`, and `/js/app.js`).
- Complete compatibility across responsive breakpoints from 320px mobile screens to 1440px+ desktop monitors.
