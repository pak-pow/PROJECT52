---
date: 2026-10-02
project: Progressive Web App (PWA)
topic: Day 6 - Web Notifications and App Badging
Tags:
  - "[[PWA]]"
  - "[[Notifications]]"
  - "[[App-Badging]]"
  - "[[Service-Worker]]"
  - "[[Dev Log]]"
---
# DEV LOG: WEEK 40 - DAY 6

**Core Objective:** Implement the Web Notifications API wrapper, App Badging API synchronization for pending task counts, Service Worker notification click routing, and header alert toggles.

---

## 1. The Big Picture and Simple Explanation

A key attribute of native operating system applications is the ability to communicate with users outside the boundaries of an active window:
1. **Operating System Notification Banners:** When an urgent or high-priority task is created, or when offline synchronization finishes, the app can display native desktop or mobile notifications via `NotificationService` and `ServiceWorkerRegistration.showNotification()`.
2. **Dynamic Operating System App Badging:** When installed to the Windows taskbar, macOS dock, or mobile home screen, the app icon can display an unread item count. TaskPulse uses the **App Badging API** (`navigator.setAppBadge()`) to keep the badge in sync with pending task counts in real time.
3. **Notification Click Routing:** Clicking on a system alert automatically searches open browser windows, bringing the existing TaskPulse window into focus via the Service Worker's `notificationclick` handler, or launching a new app window if the app was closed.

```
+-----------------------------------------------------------------------------+
|                     Notifications & App Badging Architecture                |
|                                                                             |
|      +---------------------------------------------------------------+      |
|      |             Task Event: Task Created / Marked Complete        |      |
|      +-------------------------------+-------------------------------+      |
|                                      |                                      |
|         +----------------------------+----------------------------+         |
|         |                                                         |         |
|         v                                                         v         |
|  +-------------------------------+             +-------------------------+  |
|  | App Badging API Sync          |             | Priority Task Detection |  |
|  | navigator.setAppBadge(count)  |             | (priority == "urgent")  |  |
|  +---------------+---------------+             +------------+------------+  |
|                  |                                          |               |
|                  v                                          v               |
|  +-------------------------------+             +-------------------------+  |
|  | OS Taskbar / Dock Icon        |             | Permission Check        |  |
|  | Displays Real-Time Counter    |             | Notification.permission |  |
|  +-------------------------------+             +------------+------------+  |
|                                                             |               |
|                                            +----------------+---------------+
|                                            | [Granted]                      |
|                                            v                                |
|                               +-------------------------+                   |
|                               | Service Worker Dispatch |                   |
|                               | showNotification(...)   |                   |
|                               +------------+------------+                   |
|                                            |                                |
|                                            v                                |
|                               +-------------------------+                   |
|                               | Native OS Notification  |                   |
|                               | User Clicks Banner      |                   |
|                               +------------+------------+                   |
|                                            |                                |
|                                            v                                |
|                               +-------------------------+                   |
|                               | sw.js notificationclick |                   |
|                               | Focus Window / Open App |                   |
|                               +-------------------------+                   |
+-----------------------------------------------------------------------------+
```

---

## 2. Key Engineering Implementations

### Notification Service (`frontend/src/utils/notifications.js`)
- Permission lifecycle manager: Evaluates `Notification.permission` and coordinates permission prompts via `requestPermission()`.
- Service Worker notification dispatch: Triggers desktop and mobile alerts via `registration.showNotification(title, options)` with standard icons, badges, tags, and action buttons.
- App Badging API integration: Encapsulates `navigator.setAppBadge(count)` and `navigator.clearAppBadge()`. Sets numeric indicators when pending tasks exist and clears badges when all tasks are completed.

```javascript
export async function updateAppBadge(pendingCount) {
  if ('setAppBadge' in navigator) {
    try {
      if (pendingCount > 0) {
        await navigator.setAppBadge(pendingCount);
      } else {
        await navigator.clearAppBadge();
      }
    } catch (err) {
      console.warn('App Badging failed:', err);
    }
  }
}
```

### Service Worker Event Routing (`frontend/public/sw.js`)
- Notification click handler (`notificationclick`): Intercepts notification clicks, closes the alert banner, queries open client windows, and focuses the existing TaskPulse window without reloading. If no client window is active, it opens the app shell URL.
- Push notification listener (`push`): Supports incoming web push payloads, displaying system notifications while the application is in the background.
- Cache asset update: Added `notifications.js` to the precache asset manifest and bumped cache version to `v3`.

### UI Integration & Header Controls (`src/main.js`, `src/components/header.js`)
- Header notification toggle: Added an interactive alerts button in the navigation header displaying status: `Alerts On`, `Alerts`, or `Blocked`.
- Priority notification trigger: Automatically dispatches a system alert whenever a task marked with "urgent" or "high" priority is created.
- Real-time badge synchronization: Invokes badge updates during every task refresh cycle.

---

## 3. Verification & Quality Gates Report

The notification and badging subsystems were verified through automated checks and browser simulations:

| Verification Target | Component / Module | Status |
| :--- | :--- | :---: |
| Notification Permission Flow | `notifications.js`, permission requests and states | Passed |
| App Badging API | `updateAppBadge()`, badge count set and clear calls | Passed |
| Window Focus on Click | `sw.js` notificationclick event handling | Passed |
| UI Header Alerts Button | Header alert toggle actions and label states | Passed |
| Backend Quality Standards | Flake8, Black, isort, Bandit, Pytest (14 tests) | Passed |

---

## 4. Key Engineering Insights & Decisions

- **App Badges Create Native Feel:** Keeping native taskbar and dock badges synchronized with pending task counts provides continuous utility even when the application window is minimized.
- **Service Worker Notification Routing:** Utilizing `registration.showNotification()` rather than `new Notification()` allows notifications to trigger while the application tab is closed, and enables reliable window focusing via `clients.matchAll()`.
- **Graceful Fallbacks for Unsupported APIs:** Wrapping modern browser capabilities like the App Badging API in feature-detection checks (`'setAppBadge' in navigator`) ensures smooth degradation on unsupported platforms.

---

## 5. Next Steps

- Execute final end-to-end verification, Lighthouse PWA audit compliance checks, and author technical project documentation (`README.md`).
- Update Phase 3 roadmap status and integrate TaskPulse into the production master landing portal.
