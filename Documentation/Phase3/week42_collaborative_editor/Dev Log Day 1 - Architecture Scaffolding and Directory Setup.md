---
date: 2026-10-13
project: Real-Time Collaborative Editor
topic: Day 1 - Architecture Scaffolding and Directory Setup
Tags:
  - "[[Collaborative-Editing]]"
  - "[[Operational-Transformation]]"
  - "[[WebSockets]]"
  - "[[Architecture]]"
  - "[[Dev Log]]"
---
# DEV LOG: WEEK 42 - DAY 1

**Core Objective:** Kick off Week 42 (Real-Time Collaborative Editor - PulsePad) by establishing the 7-day implementation roadmap, designing the Operational Transformation (OT) architecture, and scaffolding the complete backend and frontend directory structure with clean baseline placeholder files.

---

## 1. The Big Picture and Simple Explanation

In traditional single-user applications, updating a text document is trivial: the client sends the entire string or a patch, and the server saves it. However, in real-time collaborative editing (such as Google Docs or Notion), multiple collaborators type simultaneously across distributed networks with varying latencies.

If two users edit character offsets simultaneously without coordinated synchronization:
1. **Lost Updates:** A user saving later in time will overwrite the earlier user's input.
2. **Offset Drift:** If User A inserts a word at character index 0, User B's intended deletion at character index 10 will mistakenly delete the wrong characters unless shifted forward by the length of User A's insertion.

To resolve this deterministically, **PulsePad** utilizes **Operational Transformation (OT)**:
- Instead of transmitting full text snapshots on every keypress, clients emit atomic operations (`retain`, `insert`, `delete`).
- The server acts as the linear source of truth, assigning sequential revision numbers ($v_0, v_1, v_2, \dots$).
- When concurrent operations arrive, they are mathematically transformed against concurrent history so that all clients converge to the exact same text document.

```
+-----------------------------------------------------------------------------+
|                                PulsePad Day 1 Architecture                  |
|                                                                             |
|  [ Frontend Workbench ]                   [ Backend Server (Flask) ]        |
|  - HTML Shell (index.html)                - App Factory (create_app)        |
|  - CSS Tokens & Layout                    - SQLite Connection & Schema      |
|  - Client OT Engine (stub)                - OT Primitives (TextOperation)   |
|  - Presence & Simulator (stubs)           - SocketIO Room Manager (stub)    |
|                                           - REST Endpoints (/health, docs)  |
+-----------------------------------------------------------------------------+
```

---

## 2. Directory Tree Scaffolding Created Today

The baseline directory layout and placeholder files were scaffolded cleanly:

```
week42_collaborative_editor/
├── backend/
│   ├── app/
│   │   ├── config/
│   │   │   ├── __init__.py
│   │   │   └── settings.py
│   │   ├── db/
│   │   │   ├── __init__.py
│   │   │   └── connection.py
│   │   ├── engine/
│   │   │   ├── __init__.py
│   │   │   ├── compose.py
│   │   │   ├── document.py
│   │   │   ├── invert.py
│   │   │   ├── operation.py
│   │   │   └── transform.py
│   │   ├── models/
│   │   │   ├── __init__.py
│   │   │   ├── document_model.py
│   │   │   └── revision_model.py
│   │   ├── routes/
│   │   │   ├── __init__.py
│   │   │   ├── document_routes.py
│   │   │   └── health_routes.py
│   │   ├── services/
│   │   │   ├── __init__.py
│   │   │   ├── document_service.py
│   │   │   └── revision_service.py
│   │   └── socket/
│   │       ├── __init__.py
│   │       ├── editor_events.py
│   │       └── room_manager.py
│   ├── data/
│   │   ├── schema.sql
│   │   └── seed.py
│   ├── tests/
│   │   ├── __init__.py
│   │   ├── conftest.py
│   │   ├── test_e2e_convergence.py
│   │   ├── test_ot_engine.py
│   │   ├── test_presence.py
│   │   ├── test_routes.py
│   │   └── test_socket_sync.py
│   └── run.py
├── frontend/
│   ├── public/
│   │   └── index.html
│   └── src/
│       ├── assets/
│       │   ├── base.css
│       │   ├── editor.css
│       │   └── timeline.css
│       ├── engine/
│       │   ├── client_ot.js
│       │   └── cursor_tracker.js
│       ├── socket/
│       │   └── socket_client.js
│       ├── ui/
│       │   ├── presence_view.js
│       │   ├── split_simulator.js
│       │   └── timeline_view.js
│       └── main.js
└── README.md
```

---

## 3. Database Schema Setup

A transactional SQLite schema was defined in `backend/data/schema.sql` supporting:
- `documents`: Primary document records with title, content body, and monotonic version counter.
- `revisions`: Ordered sequence of applied operations with author metadata and JSON payload.
- `snapshots`: Periodic full document snapshots for fast state recovery.

---

## 4. Next Steps for Day 2

Day 1 completed the scaffolding and project baseline.
On **Day 2**, we will focus exclusively on implementing the core **Operational Transformation (OT) Engine**:
- Pairwise transformation algorithm $T(op1, op2) \to (op1', op2')$ for insert/insert, insert/delete, and delete/delete conflicts.
- Operation composition $compose(op1, op2)$ to coalesce consecutive keystrokes.
- Operation inversion $invert(op, state)$ to support undo and redo.
