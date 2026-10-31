---
date: 2026-10-31
project: PulseInfer ML Model Serving API
topic: Day 6 - Modern Interactive Model Workbench and Observability Dashboard
Tags:
  - "[[ML-Model-Serving]]"
  - "[[Frontend-Workbench]]"
  - "[[Observability]]"
  - "[[Concurrency-Benchmarker]]"
  - "[[Dev Log]]"
---
# DEV LOG: WEEK 44 - DAY 6

**Core Objective:** Design and implement a modern interactive Model Workbench UI, real-time Observability Dashboard, and In-Browser Concurrency Benchmarker suite in vanilla ES6 modules, featuring dynamic domain-specific input rendering, version hot-swapping controls, automated schema presets, animated visual inference outputs, and live client-side latency quantile calculations.

---

## 1. The Big Picture and Simple Explanation

A machine learning model serving gateway is incomplete without developer and operator tooling. Developers need to experiment with models directly, evaluate edge cases, and inspect raw predictions; operators need continuous visibility into gateway throughput, cache hit ratios, queue lengths, and tail latency quantiles.

In Day 6, we designed and built the complete user interface layer for **PulseInfer**:
1. **Interactive Model Workbench:** A dynamic playground that adapts its input controls to the selected model domain (NLP text sentence with sentiment bars, Vision base64 upload with pixel preview canvas, and Tabular feature sliders for fraud transaction scoring). Operators can execute hot-swaps between model versions directly from the UI with zero server downtime.
2. **In-Browser Concurrency Benchmarker:** An integrated testing suite allowing engineers to simulate parallel client traffic (5 to 100 concurrent workers) against configurable workloads (cache saturation stress, dynamic micro-batching throughput stress, and mixed real-world traffic). The benchmarker calculates client-observed round-trip latency quantiles (p50, p90, p95, p99) and displays live streaming logs and operational insight banners.
3. **Observability Dashboard:** A real-time telemetry dashboard visualizing sliding-window percentiles, gateway throughput in requests per second, cache occupancy and memory efficiency, per-model inference breakdowns, and persistent SQLite WAL execution audit logs with live 3-second auto-refresh.

```
+-----------------------------------------------------------------------------+
|               PulseInfer Modern Frontend Architecture                       |
|                                                                             |
|  [Single-Page Web Application: vanilla ES6 modules & custom dark CSS]       |
|                                                                             |
|  +--------------------+  +--------------------+  +-----------------------+  |
|  | Inference          |  | Concurrency        |  | Observability         |  |
|  | Workbench          |  | Benchmarker        |  | Dashboard             |  |
|  |                    |  |                    |  |                       |  |
|  | - Domain Inputs    |  | - 5-100 Workers    |  | - Global Latencies    |  |
|  | - Quick Presets    |  | - Cache Stress     |  | - p50/p90/p95/p99     |  |
|  | - Hot-Swap Trigger |  | - Batching Stress  |  | - Cache Health        |  |
|  | - Animated Gauges  |  | - Real-time p50-99 |  | - Audit Log Stream    |  |
|  +--------------------+  +--------------------+  +-----------------------+  |
|            |                       |                         |              |
|            +-----------------------+-------------------------+              |
|                                    v                                        |
|             [Fetch API: JSON Payloads & Streaming Responses]                |
|                                    |                                        |
|                                    v                                        |
|  [PulseInfer REST API Gateway: /api/v1/models, /api/v1/metrics, /api/health] |
+-----------------------------------------------------------------------------+
```

---

## 2. Technical Implementation Highlights

### 2.1 Dynamic Model Workbench Controller (`workbench.js`)
- **Domain-Adaptive Form Rendering:**
  - *NLP Sentiment:* Multiline text input with quick preset buttons ("Positive Review", "Negative Complaint", "Neutral Query") and bypass cache/batcher options.
  - *Vision Classifier:* Base64 input and live decoded thumbnail canvas preview with presets ("Landscape Scene", "Document Scan", "Portrait Photo").
  - *Tabular Fraud Scorer:* Grid containing amount numeric input, cardholder age slider, 1-hour transaction velocity, and domestic/foreign dropdown.
- **Visual Result Representations:**
  - Sentiment analysis probabilities are rendered with animated color-coded bars (`positive` emerald, `negative` rose, `neutral` slate).
  - Vision predictions display the detected class tag, top confidence score, and distribution across categories.
  - Fraud scorer displays risk tier pills (`LOW`, `MEDIUM`, `HIGH`), calibrated risk score gauges, and audit warning banners for flagged transactions.
- **Live Version Hot-Swapper:**
  - Fetches registered versions via `GET /api/v1/models/<id>/metadata`.
  - Dispatches `POST /api/v1/models/<id>/reload` and reports swap latency in milliseconds without dropping connections.

### 2.2 In-Browser Concurrency Benchmarker (`benchmarker.js`)
- **Parallel Worker Pool:** Utilizes asynchronous JavaScript worker loops executing across configurable concurrency pools (5 to 100 workers).
- **Client-Side Quantile Computation:** Sorts recorded round-trip latencies on the fly to calculate client-observed `p50`, `p90`, `p95`, and `p99` percentiles.
- **Workload Generator:**
  - *Cache Saturation Stress:* Dispatches identical payloads across all threads, testing the gateway cache response time (< 1ms) and cache hit ratios (> 90%).
  - *Dynamic Batching Stress:* Dispatches diverse unique payloads, coalescing parallel client requests into optimized batch matrix operations.
  - *Mixed Traffic:* 50% identical queries and 50% varied queries simulating real-world production load.
- **Operational Insight Summaries:** Evaluates completed benchmark runs and surfaces clear insights explaining system efficiency, throughput gains, and cache hit acceleration.

### 2.3 Real-Time Observability Dashboard (`dashboard.js`)
- **Telemetry Cards:** Displays global request counts, average sliding-window latencies, cache hit ratios, throughput (req/s), and error rates.
- **Latency Percentiles Breakdown:** Exposes server-computed `p50`, `p90`, `p95`, and `p99` percentiles from the sliding-window telemetry engine.
- **Cache Memory Telemetry:** Tracks live cache occupancy (`size / max_size`), hits, misses, and evictions with a one-click cache purge button (`POST /api/v1/cache/clear`).
- **Audit Logs Table:** Queries `GET /api/v1/logs` with model filter dropdowns and automated 3-second background polling.

---

## 3. Engineering Challenges and Solutions

### Challenge 1: Accurate Round-Trip Latency vs. Server Execution Time
- *Problem:* Network latency, browser event loop overhead, and server processing times can diverge.
- *Solution:* Benchmarker tracks both client-measured total elapsed time (`performance.now()`) and server-measured execution latency reported in response JSON payloads (`latency_ms`), offering a transparent view of network versus backend overhead.

### Challenge 2: Non-Blocking Real-Time Stream Rendering Under Heavy Load
- *Problem:* Updating the DOM table on every completed request during a 100-request benchmark at 200 req/s can freeze the browser main thread.
- *Solution:* Throttled UI updates to batched increments (every 2 completed requests) and capped the live streaming log table to the 20 most recent entries, ensuring 60 FPS smooth rendering during high-concurrency runs.

---

## 4. Verification and Status

- Clean dark-mode responsive UI tested across desktop and tablet screen widths.
- Real-time communication verified against Flask backend with active hot-swapping and cache invalidation.
- All 106 automated backend tests passing with 97% overall coverage.
