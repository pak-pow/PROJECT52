---
Date: Jan 02, 2026
Project: Personal Portfolio v1
Tech Stack: HTML5, CSS3 (Flexbox)
Status: ✅ Bugs Resolved & Features Added
Tags:
  - "[[Refactoring]]"
  - "[[Bug Fix]]"
  - "[[Architecture]]"
---
## 1. The Refactor (Multi-Page Architecture)
I transitioned the application from a **Single-Page Application (SPA)** to a **Multi-Page Architecture**.
* **Created:** `about.html` as a dedicated route for personal branding.
* **Deep Linking:** Implemented navigation logic (`index.html#projects`) to ensure users land on the correct section even when navigating from a different page.

## 2. The Patch Notes (Bug Fixes)

### 🐛 Bug: Floating Footer
**Symptoms:** On pages with low content (`about.html`), the footer floated in the middle of the screen, breaking immersion.
**Diagnosis:** The `<body>` height was only equal to the content height.

**Fix:** Implemented the **Flexbox Sticky Footer** pattern.

```css
body {
    min-height: 100vh;      /* Force body to be at least screen height */
    display: flex;
    flex-direction: column;
}
main {
    flex: 1; /* Grow to fill available empty space */
}
````

### 🐛 Bug: Unclickable Cards

Symptoms: Project cards required users to hunt for a text link.

Fix: Wrapped the entire .project-card container in an anchor (`<a>`) tag, effectively expanding the "Hitbox" to the entire component.

Style Adjustment: Removed default blue/underline styles from these wrapper links to maintain the card aesthetic.

## 3. Feature Additions

- **Home Briefing:** Added a "Who is Vincent?" summary section to the landing page to improve user retention before they scroll to projects.
- **GitHub Integration:** "View My Code" button now executes an external jump to the GitHub profile.
