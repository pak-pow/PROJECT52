---
date: 2026-07-22
project: Social Media Feed
topic: New Features - Image Preview, Repost, Who to Follow, Edit Profile
Tags:
  - "[[Flask]]"
  - "[[SQLite]]"
  - "[[Python]]"
  - "[[JavaScript]]"
  - "[[Feature Development]]"
  - "[[Dev Log]]"
---

# 📝 DEV LOG: WEEK 30 - DAY 4

**Core Objective:** Build all remaining Day 4 features — image compose preview, repost toggle, who to follow sidebar, and edit profile modal — with a separate commit for each.

## 1. The Initiative & Context
Day 3 was a stability pass. Day 4 shifted back to features. Four things were marked as missing from a fully functional social feed: the compose box had a blind file input with no visual feedback, the 🔁 repost button was rendered on every card but completely dead, the "Who to Follow" sidebar panel was an empty placeholder, and the "Edit Profile" button on your own profile did nothing. All four were shipped today.

## 2. Feature 1 — Image Preview in Compose Box
**Problem:** The compose box had a 📷 icon that opened a file picker, but once you selected a file nothing happened visually. You had no confirmation the image was attached, no way to remove it without clearing the whole browser state, and no sense of what would actually get posted.

**Implementation:**
- Added `<div id="compose-image-preview">` containing an `<img>` and a ✕ remove button directly inside the compose area, hidden by default.
- Wired a `change` event listener on `#compose-image-input` using `FileReader.readAsDataURL()` to generate an in-memory preview URL and set it as the `img.src`, then removed the `hidden` class.
- The ✕ remove button (`#compose-remove-image`) clears `input.value`, resets `img.src = ""`, and re-hides the preview.
- Fixed the CSS class mismatch in `compose.css` — the existing style used `.remove-image` but the HTML used `.image-preview-remove`. Unified to `.image-preview-remove`, added `cursor: pointer`, `border: none`, and `transition`.
- On successful post submit, `compose-image-preview` is also hidden and the input cleared.

**Files changed:**
- `frontend/public/index.html` — preview div + remove button
- `frontend/src/main.js` — `change` + `click` event listeners
- `frontend/src/assets/compose.css` — fixed class name, added `display: inline-block` and `max-width: 100%`

---

## 3. Feature 2 — Repost Toggle
**Problem:** Every post card rendered a 🔁 button with a count. Clicking it did nothing. There was no backend endpoint, no model function, and the button had no event listener.

**Architecture decision:** A repost is stored as a regular `posts` row with `repost_of_id` set to the original post's ID and `content = ''`. This means reposts naturally appear in feeds, can be deleted like any post, and repost count is just `SELECT COUNT(*) FROM posts WHERE repost_of_id = ?`. Toggle = check if a row with `(user_id, repost_of_id)` exists, delete if yes, create if no.

**Backend:**

Added to `post_model.py`:
- `has_reposted(user_id, post_id)` — checks existence
- `get_reposted_post_ids(user_id, post_ids)` — batch check for feed serialization
- `toggle_repost(user_id, post_id)` — creates or deletes the repost row, returns `{"reposted": bool, "count": int}`

Added to `post_routes.py`:
- `POST /api/posts/<id>/repost` — requires auth, blocks self-repost (returns 400), calls `toggle_repost`, returns result.

Updated `serializers.py`:
- `serialize_post()` gained a `reposted_ids` parameter and now returns `"reposted_by_me": bool`.

Updated all feed routes (`/feed`, `/explore`, `GET /posts/<id>`, `GET /users/<username>/posts`) to batch-fetch `reposted_ids` and pass them to `serialize_post`.

**Frontend:**

- Added `apiRepostPost(postId)` to `postApi.js`.
- Imported `apiRepostPost` in `main.js`.
- Updated `renderPostCard` innerHTML to set `reposted` class on initial render if `post.reposted_by_me`.
- Added `repostBtn.dataset.count` for raw count storage (same pattern as `likeBtn.dataset.count` from Day 3).
- Wired full optimistic UI: toggle green class + update count immediately, confirm or roll back from API response, show toast "Reposted!" / "Repost removed." on success.
- Self-repost blocked on backend returns a 400 with `error` field which the frontend shows via rollback + toast.
- Added `.repost-btn.reposted { color: #22c55e; }` to `base.css`.

**Files changed:**
- `backend/app/models/post_model.py`
- `backend/app/routes/post_routes.py`
- `backend/app/routes/user_routes.py`
- `backend/app/services/serializers.py`
- `frontend/src/api/postApi.js`
- `frontend/src/assets/base.css`
- `frontend/src/main.js`

---

## 4. Feature 3 — Who to Follow Sidebar
**Problem:** The `<aside class="right-panel">` contained `<div id="suggestions-list"></div>` which was permanently empty. The sidebar showed "Who to follow" as a heading with nothing under it.

**Backend:**

Added `GET /api/users/suggestions` to `user_routes.py`:
- Returns up to 5 users the current user is not yet following, ordered by follower count DESC.
- SQL uses a subquery to exclude the current user's own account and all users they already follow.
- Also added `get_db` to the imports at the top of `user_routes.py` (was missing, was using services via other modules).

**Frontend:**

Added `apiGetSuggestions()` to `userApi.js` — returns `[]` on failure so the sidebar degrades gracefully.

Added `loadSuggestions()` to `main.js`:
- Shows "Loading…" skeleton text while fetching.
- Shows "You're following everyone!" if the list is empty.
- For each user: renders a mini card with avatar div (initials fallback + image load), display name, `@username`, and a "Follow" button.
- Clicking the name/username area calls `loadProfile(username)`.
- Clicking Follow calls `apiToggleFollow(username)` inline — on success, button changes to "Following" with a filled style. If somehow un-followed from suggestions (shouldn't happen normally), the item is removed from the list.
- `loadSuggestions()` is called at the end of `bootApp()` so it populates immediately on login.

Added CSS in `base.css`:
- `.suggestion-item` — flex row with gap and divider
- `.suggestion-info` — cursor pointer, hover name colour change
- `.suggestion-follow-btn` — pill shape matching follow button style elsewhere

**Files changed:**
- `backend/app/routes/user_routes.py`
- `frontend/src/api/userApi.js`
- `frontend/src/assets/base.css`
- `frontend/src/main.js`

---

## 5. Feature 4 — Edit Profile Modal
**Problem:** Viewing your own profile showed an "Edit Profile" button. Clicking it did nothing — there was no event listener, no modal, no API call, and `apiUpdateProfile` was never imported anywhere in `main.js`.

**Backend fixes:**
`update_user_profile()` in `user_model.py` previously returned `None`. Updated to re-fetch the updated row with `SELECT * FROM users WHERE id = ?` after commit and return it.

`PUT /api/users/me` in `user_routes.py` previously returned just `{"message": "..."}`. Updated to include `display_name`, `bio`, `avatar_path`, and `username` in the response so the frontend can update state without a second GET.

**Frontend:**

Added `apiUpdateProfile({ displayName, bio, avatarFile })` to `userApi.js`:
- Builds a `FormData` object (backend reads `request.form` + `request.files`).
- Sends `PUT /api/users/me` via `fetchAuth`.

Added Edit Profile modal HTML to `index.html`:
- Same `modal-overlay` / `modal-card` pattern as the compose modal.
- Header: ✕ close, title, Save button.
- Body: avatar preview `div.avatar-xl` + 📷 Change label + hidden file input, then display name `<input>`, then bio `<textarea>` with a live char counter.
- Error div below body for inline validation messages.

Added `openEditProfileModal(profile, profileContainer)` function to `main.js`:
- Pre-fills display name and bio from the passed `profile` object.
- Shows current avatar image (load attempt via `avatarUrl()`).
- Wires `change` on avatar file input → `FileReader` → updates avatar preview in-modal.
- Wires bio `input` event → live counter `160 - bioInput.value.length`.
- Close: X button, backdrop click both call `closeModal()` which hides the overlay and removes the bio listener.
- Save: validates non-empty display name, calls `apiUpdateProfile`, on success:
  - Closes modal.
  - Shows toast "Profile updated! ✨".
  - Updates `sidebarDisplayName.textContent` live.
  - Calls `saveSession()` with new display name / avatar path to keep localStorage in sync.
  - Refreshes sidebar avatar image with cache-bust `?t=Date.now()`.
  - Calls `loadProfile(username)` to re-render the profile page with new data.

Hooked up in `loadProfile()`: after rendering profile HTML, if `isMe` is true, attaches a click listener on `#edit-profile-btn` → `openEditProfileModal(profile, container)`.

Added CSS in `profile.css`:
- `.edit-profile-body` — column flex direction
- `.edit-profile-avatar-section` — centered column with avatar + label
- `.edit-field-input` — styled text input matching the dark theme
- `.edit-bio-counter`, `.edit-profile-error`

**Files changed:**
- `backend/app/models/user_model.py`
- `backend/app/routes/user_routes.py`
- `frontend/public/index.html`
- `frontend/src/api/userApi.js`
- `frontend/src/assets/profile.css`
- `frontend/src/main.js`

