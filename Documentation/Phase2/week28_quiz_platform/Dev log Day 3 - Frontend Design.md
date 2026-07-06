---
date: 2026-07-06
project: Project 52
week: 28
module: Quiz Platform with Leaderboards
topic: Day 3 - Frontend UI Shell & Design System
author: Paul Vincent Aguirre
status: Complete
Tags:
  - "[[HTML]]"
  - "[[CSS]]"
  - "[[Frontend]]"
  - "[[Design System]]"
  - "[[Dev Log]]"
  - "[[UI]]"
---
# DEV LOG: WEEK 28 - DAY 3

## 1. Goal of the Day
Build the entire frontend visual layer from scratch — the HTML app shell with all four views and a complete dark-mode CSS design system. No JavaScript today; the goal was to have a fully styled, browser-viewable UI before wiring up any logic.

---

## 2. Day Split Decision
Originally Day 3 was planned to cover the full frontend (HTML + CSS + JS). We decided to split it into two focused days:

- **Day 3** — UI Shell & Design System (HTML + CSS)
- **Day 4** — JavaScript & API Integration (JS only)

This makes it easier to catch design issues early and keeps each day's scope manageable.

---

## 3. HTML App Shell (`frontend/public/index.html`)

Built as a **Single Page Application (SPA)** shell — one HTML file with four `<section>` views toggled by a CSS `.active` class. JavaScript will swap which view is active at runtime.

### Views

| View ID | Purpose |
|---|---|
| `#view-catalog` | Quiz selection screen with cards for each available quiz |
| `#view-quiz` | Active quiz runner — question card, options, countdown timer |
| `#view-results` | Score summary with per-question correct/incorrect breakdown |
| `#view-leaderboard` | Top 10 ranked scores for the completed quiz |

### Notable HTML decisions

- **Static preview content** was added to every view so the UI is fully visible in the browser without any JavaScript running. This makes design review much faster.
- **Skeleton loader cards** were added to the catalog grid — two placeholder cards that pulse while JS loads real data from the API.
- **Username modal** (`#modal-username`) is a fixed overlay that appears before submitting answers, prompting the user for a display name for the leaderboard.
- **SVG countdown timer ring** is built inline in the HTML with two `<circle>` elements — a background track and a foreground stroke whose `stroke-dashoffset` is animated by JS in Day 4.
- All interactive elements have unique IDs for clean JavaScript selection (e.g. `#btn-next`, `#btn-quit-quiz`, `#input-username`).

**Commit:** `feat(week28): add frontend HTML shell with all four views and username modal`

---

## 4. CSS Design System (`frontend/src/assets/style.css`)

Built a complete design system from scratch using **CSS custom properties** as the foundation.

### Color Tokens
```css
--bg:        #0a0a0a    /* Page background */
--surface:   #111111    /* Card background */
--surface-2: #1a1a1a    /* Input / secondary surface */
--surface-3: #222222    /* Hover / skeleton fill */
--border:    #2a2a2a    /* Default border */
--blue:      #4196f3    /* Primary accent */
--purple:    #a855f7    /* Gradient end */
--green:     #22c55e    /* Correct answer */
--red:       #ef4444    /* Wrong answer / timer danger */
--muted:     #666666    /* Dimmed text */
```

### Key Components Built

**Quiz Cards (Catalog)**
- Hover lift effect: `translateY(-4px)` + blue glow `box-shadow`
- Subtle gradient overlay on hover (4% opacity)
- Category badge with pill styling
- Skeleton loader variant with pulse animation

**Option Buttons (Quiz Runner)**
- Four states: default / `.selected` (blue) / `.correct` (green) / `.incorrect` (red)
- Smooth border-color and background transitions
- Disabled state for after-answer lock

**Countdown Timer Ring**
- SVG `stroke-dashoffset` animation via CSS variable `--ring-circumference: 326.73px` (2π × 52)
- Default stroke color: `var(--blue)`
- `.danger` class (≤ 10 seconds): stroke turns red + pulse opacity animation

**Progress Bar**
- Linear bar across the top of the quiz header
- Gradient fill (`var(--gradient)`)
- Smooth `width` transition on question advance

**Breakdown Rows (Results)**
- Left border accent — green for correct, red for incorrect
- Icon, question text, and answer text in a flex row

**Leaderboard Table**
- `.rank-gold`, `.rank-silver`, `.rank-bronze` row tints
- Medal emoji badges for top 3
- Numbered badge for positions 4+

**Modal Overlay**
- `backdrop-filter: blur(6px)` frosted glass effect
- Fade + slide-up animation on open

### View Transition Animation
All views animate in with `fadeSlideUp`:
```css
@keyframes fadeSlideUp {
  from { opacity: 0; transform: translateY(20px); }
  to   { opacity: 1; transform: translateY(0); }
}
```

### Responsive Design
Mobile-first breakpoint at `640px`:
- Catalog grid collapses from 2-col to 1-col
- Options grid collapses from 2-col to 1-col
- Action buttons stack vertically
- Font sizes and padding scale down

**Commit:** `feat(week28): add full dark-mode design system CSS with all view styles and responsive layout`

---
