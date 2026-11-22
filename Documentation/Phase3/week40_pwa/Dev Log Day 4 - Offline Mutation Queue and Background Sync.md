---
date: 2026-09-30
project: Progressive Web App (PWA)
topic: Day 4 - Offline Mutation Queue and Background Sync
Tags:
  - "[[PWA]]"
  - "[[Offline-First]]"
  - "[[Background-Sync]]"
  - "[[Sync-Queue]]"
  - "[[Dev Log]]"
---
# DEV LOG: WEEK 40 - DAY 4

**Core Objective:** Implement the offline mutation queue replay engine, Service Worker background synchronization, bidirectional delta reconciliation, and the complete backend sync API with 5/5 passed quality gates.

---

## 1. The Big Picture and Simple Explanation

In traditional web architectures, losing connectivity during user actions leads to immediate failures, broken states, and lost work. In an offline-first Progressive Web App, user interactions are never blocked by network latency or connection drops.

In Day 4, we implemented the complete bidirectional synchronization pipeline:
1. **Instantaneous Local Persistence:** Actions write immediately to local IndexedDB, providing sub-5ms UI updates without network waiting.
2. **Persistent Mutation Logging:** Offline actions are recorded in an append-only `sync_queue` store inside IndexedDB, surviving browser closures or device restarts.
3. **Background Sync Trigger:** When connectivity is restored, the Service Worker fires a `sync` event, or the browser reconnection listener triggers automated replay.
4. **Bidirectional Reconciliation:** Pending local mutations are sent in a batch to the backend (`/api/sync/batch`), where conflicts are resolved using Last-Write-Wins (LWW) based on ISO timestamps. The client then pulls server-side updates via delta queries (`/api/sync/delta`) and merges them into IndexedDB.
5. **Real-Time Visual Indicators:** The header displays live sync status badges ("In Sync", "Syncing...", "X Queued") and provides a manual "Sync Now" button.

```
+-----------------------------------------------------------------------------+
|                     Offline Synchronization Pipeline Architecture           |
|                                                                             |
|      +---------------------------------------------------------------+      |
|      |             User Action (Create / Toggle / Delete Task)       |      |
|      +-------------------------------+-------------------------------+      |
|                                      |                                      |
|                                      v                                      |
|      +---------------------------------------------------------------+      |
|      | Step 1: Write to Local IndexedDB (Instant UI Update < 5ms)    |      |
|      +-------------------------------+-------------------------------+      |
|                                      |                                      |
|                                      v                                      |
|      +---------------------------------------------------------------+      |
|      | Step 2: Buffer Action into Persistent sync_queue Object Store |      |
|      +-------------------------------+-------------------------------+      |
|                                      |                                      |
|                     +----------------+----------------+                     |
|                     | [Device Offline]                | [Device Online]     |
|                     v                                 v                     |
|      +-------------------------------+    +--------------------------+      |
|      | Retain in Queue               |    | Step 3: Send Batch to    |      |
|      | Update Badge ("X Queued")     |    | Flask API (/sync/batch)  |      |
|      +---------------+---------------+    +-------------+------------+      |
|                      |                                  |                   |
|                      | (Network Reconnects)             |                   |
|                      +----------------+-----------------+                   |
|                                       |                                     |
|                                       v                                     |
|      +---------------------------------------------------------------+      |
|      | Step 4: Backend Resolves Conflicts via Last-Write-Wins (LWW)  |      |
|      +-------------------------------+-------------------------------+      |
|                                      |                                      |
|                                      v                                      |
|      +---------------------------------------------------------------+      |
|      | Step 5: Pull Remote Delta Changes (/api/sync/delta)           |      |
|      +-------------------------------+-------------------------------+      |
|                                      |                                      |
|                                      v                                      |
|      +---------------------------------------------------------------+      |
|      | Step 6: Flush Processed Queue & Update Badge to "In Sync"     |      |
|      +---------------------------------------------------------------+      |
+-----------------------------------------------------------------------------+
```

---

## 2. Key Engineering Implementations

### Backend Synchronization API (`backend/app/`)
- Relational schema support: Configured `tasks` table with `is_deleted` soft-deletion flags, `created_at`, and `updated_at` timestamps for delta tracking.
- Batch sync endpoint (`POST /api/sync/batch`): Accepts batched offline mutations, executes parameterized SQLite upserts, applies Last-Write-Wins conflict resolution, and returns processed counts and server timestamps.
- Delta sync endpoint (`GET /api/sync/delta?since=...`): Returns tasks modified or soft-deleted since a given timestamp, enabling incremental reconciliation.
- Full CRUD API (`routes/task_routes.py`): Clean REST endpoints with input validation and standard HTTP response codes.

```python
def reconcile_batch(mutations):
    applied = 0
    with get_db_connection() as conn:
        for mutation in mutations:
            action = mutation["action"]
            task = mutation["payload"]
            existing = TaskModel.get_by_id(task["id"])

            if not existing or task["updated_at"] >= existing["updated_at"]:
                if action in ("CREATE", "UPDATE", "TOGGLE"):
                    TaskModel.upsert(task)
                elif action == "DELETE":
                    TaskModel.soft_delete(task["id"], task["updated_at"])
                applied += 1
        conn.commit()
    return {"applied": applied, "server_time": datetime.utcnow().isoformat()}
```

### Client Synchronization Engine (`frontend/src/`)
- API client (`src/api/taskApi.js`): Lightweight fetch wrapper managing CRUD requests, batch payloads, delta queries, and network failure fallbacks.
- Queue replay engine (`src/storage/syncQueue.js`): Reads stored mutations in FIFO order, dispatches them to `/api/sync/batch`, purges processed IDs from IndexedDB, and merges delta updates.
- Service Worker background sync (`public/sw.js` & `src/swRegister.js`): Listens for `sync` events tagged with `taskpulse-sync` and coordinates background queue flushes across open clients.
- Automated reconnection handling: Subscribed to `window.addEventListener('online')`, automatically triggering reconciliation upon connection recovery.

---

## 3. Verification & Quality Gates Report

The synchronization pipeline was verified through automated tests across both backend and client storage suites:

| Test Suite Module | Target Scope | Tests Passed | Status |
| :--- | :--- | :---: | :---: |
| `tests/test_sync_routes.py` | Batch upserting, delta time-slicing, LWW conflict rules | 6 | Passed |
| `tests/test_task_routes.py` | Task CRUD endpoints, soft-deletions, validation | 5 | Passed |
| `tests/test_health_and_version.py` | Liveness, readiness, and version probes | 3 | Passed |

### Quality Gates Execution Summary
- **Total Tests:** 14 automated tests passing in 0.42 seconds.
- **Branch Coverage:** 95.93% total branch coverage.
- **Linters:** Flake8, Black, isort, and Bandit reported 0 errors.

---

## 4. Key Engineering Insights & Decisions

- **Last-Write-Wins with UTC Timestamps:** Comparing ISO-8601 timestamps on both client and server provides deterministic conflict resolution without requiring complex operational transformation engines.
- **Soft Deletion for Distributed Sync:** Hard-deleting rows on the server prevents clients from discovering which records were removed. Storing `is_deleted = 1` with an update timestamp allows delta queries to transmit deletions cleanly.
- **FIFO Mutation Processing:** Processing offline mutations strictly in chronological order prevents out-of-order race conditions (e.g., executing an update before task creation).

---

## 5. Next Steps

- Implement PWA installability enhancements, custom install promotion banners, and App Shortcuts in `manifest.json`.
- Construct a dedicated Storage & Quota Inspector component (`storageInspector.js`) with persistent storage management and JSON backup export.
