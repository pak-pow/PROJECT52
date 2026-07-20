---
date: 2026-07-20
project: Social Media Feed
topic: Profile Page, Post Detail, Infinite Scroll, Bug Fixes
Tags:
  - "[[Flask]]"
  - "[[SQLite]]"
  - "[[Python]]"
  - "[[JavaScript]]"
  - "[[Full Stack]]"
  - "[[Dev Log]]"
---
# 📝 DEV LOG: WEEK 30 - DAY 2

**Core Objective:** Build the Profile page UI, Post Detail with reply threads, wire up infinite scroll, and fix critical bugs that were breaking the backend startup and the frontend user experience.

## 1. The Initiative & Context

Day 1 left the backend fully built and the frontend as a basic shell. Day 2 focused on making the app actually usable — clicking on a user goes somewhere, clicking a post opens it with replies, the feed doesn't run out of posts, and the whole thing doesn't look like it's crashing every time you press a button.

## 2. Features Built

### Feature A: Profile Page UI

Clicking any username anywhere in the app now navigates to a full profile page. The profile renders a gradient banner, avatar (real image with initials fallback), display name, bio, follower/following/post counts, and a follow/unfollow button. The follow button updates its state and follower count optimistically without a page reload. When viewing your own profile, it shows an "Edit Profile" button instead of the follow button. The user's posts are listed below the profile header.

### Feature B: Post Detail Page with Reply Threads

Clicking anywhere on a post card (except action buttons) opens the post detail page. The detail view renders the original post in a larger format, followed by an inline reply compose box, then all replies threaded below it. Submitting a reply prepends it instantly to the thread without a full page reload.

### Feature C: Infinite Scroll via IntersectionObserver

Both the Home feed and Explore feed now support infinite scroll. An `IntersectionObserver` watches a hidden loader element at the bottom of each list. When it enters the viewport, the next page of posts is fetched using cursor-based `?before=<id>` pagination. The loader disappears once all posts have been fetched.

### Feature D: Avatar Image Loading

Every post card, sidebar user block, and profile header now attempts to load the user's real avatar image from the API. If no avatar exists, it falls back to a styled initials badge. The avatar loads asynchronously after the card renders so it never blocks the UI.

## 3. Project Structure Updated

```
week30_social_feed/
└── frontend/
    └── src/
        ├── assets/
        │   ├── base.css     # Added: delete-btn hover, new post slide-in animation
        │   └── feed.css     # Added: post detail card variant, reply compose box
        └── main.js          # Full rewrite — profile, post detail, infinite scroll,
                             # avatar loading, click-through nav, reply submit
```

## 4. Debugging

### Bug 1: Backend Won't Start — `unable to open database file`

- **Issue:** Running `python run.py` crashed immediately with `sqlite3.OperationalError: unable to open database file`.
- **Root Cause:** `DB_PATH` points to `backend/data/social.db`, but the `data/` directory was never created on disk. Git only tracks files, not empty folders — so `schema.sql` and `seed.py` were there but the actual directory SQLite needed to write the `.db` file into didn't exist.
- **Resolution:** Added `os.makedirs(os.path.dirname(app.config["DB_PATH"]), exist_ok=True)` in `create_app()` so the `data/`, `uploads/avatars/`, and `uploads/posts/` directories are all auto-created on first run.

### Bug 2: No CSS or JS Loading — App Rendered Completely Unstyled

- **Issue:** The app opened in the browser with zero styling — raw HTML with no layout, no colors, no fonts.
- **Root Cause:** All CSS and JS paths in `index.html` were written as `src/assets/base.css` and `src/main.js`. But `index.html` lives inside `public/`, so the browser looked for `public/src/assets/` which doesn't exist. The actual files are at `frontend/src/assets/`.
- **Resolution:** Changed all paths from `src/` to `../src/` to correctly resolve one level up from `public/`.

### Bug 3: Every Button Click Looked Like a Full Page Reload

- **Issue:** Clicking the Post button after composing, or clicking the Home nav link, caused the entire feed to disappear and reappear with a skeleton loader — indistinguishable from a page reload.
- **Root Cause 1:** The compose submit handler called `resetFeed(); loadFeed()` after every post — wiping the entire feed list and re-fetching from the API.
- **Root Cause 2:** Clicking the Home nav link while already on the home page also triggered `resetFeed(); loadFeed()`.
- **Resolution:** After a successful post, the new post is now prepended directly to the top of the feed with a smooth `slide-in` animation — no reload. Clicking Home while already on Home now just smooth-scrolls to the top instead of re-fetching.

