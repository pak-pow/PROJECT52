---
date: 2026-10-16
project: Real-Time Collaborative Editor
topic: Day 4 - Multi-User Presence Awareness and Remote Cursors
Tags:
  - "[[Collaborative-Editing]]"
  - "[[Presence]]"
  - "[[Remote-Cursors]]"
  - "[[Operational-Transformation]]"
  - "[[WebSockets]]"
  - "[[Dev Log]]"
---
# DEV LOG: WEEK 42 - DAY 4

**Core Objective:** Implement the multi-user presence engine, awareness indicators, and dynamic remote cursor/selection tracking for PulsePad. Establish deterministic user color derivation, real-time caret position broadcasting over WebSockets, and mathematical OT cursor shifting when concurrent text mutations occur.

---

## 1. The Big Picture and Simple Explanation

In collaborative text editors like Google Docs, seeing where other users are typing is critical for avoiding collision and knowing which paragraph peers are focused on.

However, remote cursors are not static numbers:
1. **The Cursor Drift Problem:** Suppose Bob has his cursor parked at character offset 10. If Alice inserts 5 characters at the very start of the document (offset 0), Bob's cursor would now point to the wrong word unless it is automatically pushed forward to offset 15!
2. **The Deletion Collapse Problem:** If Alice deletes characters 5 through 15, Bob's cursor at offset 10 was inside the deleted text. The system must collapse Bob's cursor to the start of the deletion (offset 5) rather than leaving it in an invalid or shifted state.

On Day 4, **PulsePad** solves this by integrating **OT Cursor Shifting**:
- When client operations are applied to the server document, every remote collaborator's stored cursor and text selection range is automatically transformed against the applied operation.
- Users are assigned deterministic, persistent pastel identity colors based on their name.
- Real-time events broadcast remote carets, selection highlights, typing indicators, and periodic heartbeats.

```
                      Alice types at index 0: Insert "Hello " (len 6)
                                       |
    Before Edit:                       v                      After Edit:
    Doc: "world"                       |                      Doc: "Hello world"
    Bob Cursor: 0 (before "world")     |                      Bob Cursor: 6 (before "world")
    Charlie Cursor: 5 (after "world")  |                      Charlie Cursor: 11 (after "world")
```

---

## 2. Key Modules Implemented Today

### A. Cursor and Selection Transformation (`app/engine/cursor.py`)
- `get_user_color(user_name)`: Computes a deterministic MD5 hash of the username modulo 10 to select from a curated palette of distinct accessible pastel colors (`PRESENCE_COLORS`).
- `transform_cursor_position(cursor_pos, operation)`:
  - Iterates through `TextOperation` components (`retain`, `insert`, `delete`).
  - Automatically shifts the cursor forward by insertion lengths when text is inserted before or at the cursor offset.
  - Automatically collapses the cursor to the deletion origin if the cursor falls strictly within a deleted span.
  - Clamps cursor boundaries cleanly between 0 and `operation.target_length`.
- `transform_selection(start, end, operation)`: Transforms both the anchor and active head of a text selection range simultaneously.

### B. Presence-Aware Room Manager (`app/socket/room_manager.py`)
- Extended `Participant` entity with:
  - `cursor_pos: int`
  - `selection_end: Optional[int]`
  - `is_typing: bool`
  - `last_active: float` (epoch timestamp)
  - `status: str` ("active" | "idle")
- `update_participant_cursor`: Updates user caret offset and selection endpoints while updating activity timestamps.
- `update_participant_typing`: Updates typing indicator state and refreshes activity.
- `prune_inactive_participants`: Evicts or identifies inactive clients who exceed timeout thresholds.
- Integrated cursor shifting in `apply_client_operation`: Every committed edit automatically transforms the cursor positions of all other peers in the room on the server.

### C. Socket.IO Presence Event Listeners (`app/socket/editor_events.py`)
- `cursor_move`: Receives local cursor and selection updates and broadcasts `cursor_update` to all peers in the room.
- `typing_status`: Receives typing state toggles and broadcasts `typing_update` to peers.
- `heartbeat`: Receives periodic client keep-alive pings and confirms with `heartbeat_ack`.

---

## 3. Verification and Quality Gates

A total of 42 automated tests were executed, including 9 new comprehensive presence and cursor tracking tests:

- **Color Derivation:** Verified deterministic palette selection across varying usernames and empty strings.
- **Insert Shifting:** Verified insertions before, at, and after cursor offsets shift correctly.
- **Delete Shifting & Collapse:** Verified deletions before the cursor shift backward, deletions containing the cursor collapse to the deletion origin, and deletions after the cursor leave it untouched.
- **Selection Transformation:** Verified range selections shrink or shift accurately under concurrent edits.
- **Socket Broadcasts:** Verified `cursor_move`, `typing_status`, and `heartbeat` events and peer notifications.
- **Server-side Cursor Shifting:** Verified that committing an edit by Alice automatically adjusts Bob's stored server-side cursor from 3 to 9.
- **5/5 Quality Gates Passing:**
  - `flake8`: 0 warnings (88-char limit).
  - `black`: 0 formatting differences.
  - `isort`: Clean import sorting.
  - `bandit`: 0 security issues.
  - `pytest`: 42 passed, 94% coverage.

---

## 4. Next Steps for Day 5

With the mathematical transformation engine, WebSocket synchronization protocol, and presence awareness fully functional on the backend, **Day 5** will focus on:
- Developing the **Collaborative Frontend Editor Workbench**:
  - Wiring the textarea and DOM overlay with the client-side OT state machine.
  - Rendering visual floating remote cursors with user name tags and selection highlight overlays.
  - Building the interactive split-screen dual-client simulator to test multi-user collaboration in real time directly inside a single browser window.
