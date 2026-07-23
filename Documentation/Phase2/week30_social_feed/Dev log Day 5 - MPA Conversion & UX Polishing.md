---
date: 2026-07-23
project: Social Media Feed
topic: Multi-Page Architecture, Session Security, Zero-CLS Skeletons & Dev Environment
Tags:
  - "[[Flask]]"
  - "[[SQLite]]"
  - "[[Python]]"
  - "[[JavaScript]]"
  - "[[MPA Architecture]]"
  - "[[Dev Log]]"
---

# 📝 DEV LOG: WEEK 30 - DAY 5

**Core Objective:** Convert the application into a robust Multi-Page Application (MPA), implement timed session security with `sessionStorage`, resolve Cumulative Layout Shift (CLS), and fix development server watching conflicts.

---

## 1. The Initiative & Context
Day 5 focused on transitioning the architecture from a monolithic single-page layout to a clean Multi-Page Application (MPA) with dedicated HTML entry points for every route (`login.html`, `register.html`, `feed.html`, `explore.html`, `profile.html`, `post.html`). Along the way, we addressed critical edge cases in session security, zero-CLS skeleton loading, and Live Server environment configuration.

---

## 2. Feature Highlights & Refactoring

### 2.1 Multi-Page Application Architecture (MPA)
- **Standalone HTML Pages:** Created separate HTML entry points in `frontend/public/` for every page (`login.html`, `register.html`, `feed.html`, `explore.html`, `profile.html`, `post.html`, `index.html`).
- **Page Controllers & Shared Components:** Modularized JS into individual page scripts (`feedPage.js`, `explorePage.js`, `profilePage.js`, `postPage.js`, `loginPage.js`, `registerPage.js`) and shared UI modules (`sidebar.js`, `composeModal.js`, `postCard.js`, `suggestions.js`).
- **Clean Page Navigation:** Sidebar links, hashtags (`#tag`), user mentions (`@user`), and post card clicks navigate cleanly across `.html` files using standard URL parameters (`?u=`, `?tag=`, `?id=`).

### 2.2 Live Session Verification & `sessionStorage` Expiry
- **`sessionStorage` Integration:** Migrated from persistent `localStorage` to `sessionStorage` so closing the browser tab or window automatically terminates the active session.
- **30-Minute Auto-Expiry Timer:** Implemented a sliding `sf_expires_at` timer. Inactive sessions automatically expire and wipe local storage after 30 minutes.
- **Live Token Verification:** Protected pages query `GET /api/auth/me` on initial load. Invalid or expired tokens trigger automatic `clearSession()` and safe redirection to `login.html`.
- **`safeRedirect` Guard:** Guarded page navigation so `window.location.href` is never assigned if the browser is already on the target page, eliminating page reload loops.

### 2.3 Zero-CLS Profile Skeleton & Positioning Context
- **Structural `profileSkeleton`:** Designed a dedicated loading skeleton for profile pages matching the exact 140px banner height, circular avatar, and text dimensions of the profile header.
- **Zero Layout Shift:** Replaced generic card skeletons with `profileSkeleton()`, ensuring zero Cumulative Layout Shift (CLS) when profile data resolves over the network.
- **Relative Positioning Anchors:** Added `position: relative` to `.profile-header`, `.profile-avatar-wrap`, and `.profile-avatar` to guarantee absolute children remain anchored within parent boundaries.

### 2.4 Dev Environment & Live Server Ignore Rules
- **Live Server Ignore Rules:** Configured `.vscode/settings.json` ignore rules for `**/*.db`, `**/*.sqlite`, `**/__pycache__/**`, `**/backend/**`, and `**/data/**`.
- **Eliminated Reload Loops:** Prevented VS Code Live Server from triggering full browser refreshes whenever Flask written SQLite database entries or generated Python cache files.

---

## 3. Summary of Commits
1. `9cc2440` — Un-authenticated post image access (`GET /api/posts/<id>/image`).
2. `d0adb42` — Mobile floating compose FAB button and modal wiring.
3. `05bc3fd` — Debounced user search bar (`GET /api/users/search`).
4. `bfbd41a` — Hashtag linkification and `?tag=` explore feed filter.
5. `6590950` — Conversion to Multi-Page Application (MPA) with standalone HTML pages.
6. `687a9d4` — Duplicate ES module export cleanup in `userApi.js`.
7. `324fa1b` — Timed `sessionStorage` with 30-minute auto-expiry and `safeRedirect` guards.
8. `e01c4cd` — In-place profile DOM updates and softened skeleton shimmer CSS.
9. `c328812` — Zero-CLS `profileSkeleton()` component and relative positioning anchors.
10. `412be75` — Live Server `.vscode/settings.json` ignore rules for SQLite and backend artifacts.

---

## 4. Verification & Testing
- **Automated Backend Tests:** `18/18` pytest unit tests passing cleanly (`python -m pytest tests/ -q`).
- **Manual Verification:** Verified login, register, feed, explore, profile, and post detail pages across multi-page navigation without JS errors or layout shifts.
