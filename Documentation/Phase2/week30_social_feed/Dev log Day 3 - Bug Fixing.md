---
date: 2026-07-21
project: Social Media Feed
topic: Deep Audit, Bug Fixes & Stability Pass
Tags:
  - "[[Flask]]"
  - "[[SQLite]]"
  - "[[Python]]"
  - "[[JavaScript]]"
  - "[[Debugging]]"
  - "[[Dev Log]]"
---
# 📝 DEV LOG: WEEK 30 - DAY 3
**Core Objective:** Run a full dual-agent audit across the entire backend and frontend codebase, identify every bug, and fix all critical and high-severity issues before moving forward.

## 1. The Initiative & Context
Before building new features, Day 3 was dedicated entirely to quality. Two audit agents were launched in parallel — one for the backend, one for the frontend — to independently read every file and report every issue. The profile page was completely broken in production, the explore feed returned nothing, avatars were throwing 401 errors, and the delete button was completely dead. All of it was fixed today.

## 2. The Audit Process
Two research agents were spawned simultaneously:
- **Backend Auditor** — read all 17 backend files: models, routes, services, schema, seed, config
- **Frontend Auditor** — read all 11 frontend files: HTML, JS modules, CSS sheets

Each agent produced a severity-ranked report (CRITICAL / HIGH / MEDIUM / LOW). All CRITICAL and HIGH findings were immediately fixed in the same session.

## 3. Bugs Found & Fixed
### Backend — 7 Fixes

**Bug 1: Profile Page Always Showed 0 Followers**
- **Root Cause:** `serialize_user()` in `serializers.py` returned the key `"follower_count"` (singular), but the frontend's `loadProfile()` read `profile.followers_count` (plural). They never matched, so followers always rendered as 0.
- **Resolution:** Renamed the key to `"followers_count"` in the serializer.

**Bug 2: Follow Button Never Updated the Count**
- **Root Cause:** `POST /api/users/<username>/follow` only returned `{"following": bool}`. The frontend tried to read `fData.followers_count` from that response — which was `undefined` — and silently assigned `undefined` to the displayed count.
- **Resolution:** The follow endpoint now calls `get_follower_count()` after the toggle and returns `{"following": bool, "followers_count": int}`.

**Bug 3: Explore Feed Was Always Empty**
- **Root Cause:** `get_explore_feed()` had a hard filter: `AND p.created_at >= datetime('now', '-24 hours')`. All seed data inserted more than 24 hours ago was silently excluded, returning an empty list every time.
- **Resolution:** Removed the 24-hour time window. Explore now returns all posts globally, sorted by like count descending.

**Bug 4: Post Images Never Rendered**
- **Root Cause:** `serialize_post()` returned `image_path` (nullable string), but `renderPostCard()` in the frontend checked `post.has_image` (boolean). The field never existed in the API response.
- **Resolution:** Added `"has_image": bool(row["image_path"])` to the post serializer.

**Bug 5: All Avatars Returned HTTP 401**
- **Root Cause:** `GET /api/users/<username>/avatar` had `@require_auth`. Browser `<img>` tags cannot send custom `Authorization: Bearer` headers — they make plain GET requests. Every avatar image load silently failed with 401.
- **Resolution:** Removed `@require_auth` from the avatar endpoint. Profile avatars are public resources.

**Bug 6: Registration Could 500 Crash**
- **Root Cause:** `create_user()` manually checked for duplicate usernames with `SELECT`, then did the `INSERT`. A race condition between those two operations could trigger `sqlite3.IntegrityError`, which was not caught — resulting in an unhandled server crash.
- **Resolution:** Added `except sqlite3.IntegrityError` block that re-raises as `ValueError`, which the route already handles gracefully.

**Bug 7: CORS Blocked Authorization Headers**
- **Root Cause:** CORS was configured with `CORS(app, resources={"/api/*": {"origins": "*"}})` without explicitly whitelisting the `Authorization` and `Content-Type` headers. Some browsers reject preflight requests that don't declare these.
- **Resolution:** Added `allow_headers=["Content-Type", "Authorization"]` and explicit methods to the CORS config.

### Frontend — 6 Fixes

**Bug 8: Delete Button Did Absolutely Nothing**
- **Root Cause:** `renderPostCard()` rendered the 🗑️ delete button in the HTML via `innerHTML`, but no `.addEventListener()` was ever called on it. Additionally, `apiDeletePost` was never imported in `main.js`.
- **Resolution:** Added `apiDeletePost` to the import statement. Added a `click` listener on `.delete-btn` that confirms deletion, calls the API, removes the card from the DOM on success, and shows a toast.

**Bug 9: Like Count Broke on Popular Posts**
- **Root Cause:** The like button's optimistic update used `parseInt(countEl.textContent)` to read the current count. When `formatCount()` formats numbers like `1500` as `"1.5K"`, `parseInt("1.5K")` returns `1`. A post with 1,500 likes would drop to 2 after clicking like.
- **Resolution:** Stored the raw numeric count in `likeBtn.dataset.count` on render. All subsequent reads and writes use this raw number, and `formatCount()` is only used for display.

**Bug 10: Duplicate Posts from Infinite Scroll Race**
- **Root Cause:** `loadFeed()` and `loadExplore()` only checked `feedDone` before fetching. The `IntersectionObserver` could fire multiple times during the initial page load before the first fetch resolved, triggering concurrent duplicate requests.
- **Resolution:** Added `feedLoading` and `exploreLoading` boolean flags. Each async fetch returns early if a load is already in progress, and resets the flag in all exit paths.

**Bug 11: Clicking @username in Posts Did Nothing**
- **Root Cause:** `linkifyContent()` in `helpers.js` converts `@username` to `<a href="#profile?u=username" class="mention">@username</a>`. The card click handler explicitly ignores clicks on `<a>` tags, and no other listener existed for `.mention` links.
- **Resolution:** Added a delegated `click` listener on `document` that intercepts any click on `.mention`, extracts the username from the text, and calls `loadProfile(username)`.

**Bug 12: "No Replies Yet" Stayed After First Reply**
- **Root Cause:** When the reply compose box submitted successfully, the new reply card was prepended to `repliesContainer`, but the `<p class="empty-state">No replies yet.</p>` placeholder was never removed.
- **Resolution:** Added `repliesContainer.querySelector(".empty-state")?.remove()` before prepending the new reply card.

