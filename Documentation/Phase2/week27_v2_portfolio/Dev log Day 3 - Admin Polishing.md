---
date: 2026-06-28
project: Project 52
week: 27
module: Portfolio v2 (Full Stack Upgrade)
topic: Day 3 - Dashboard Stats & Custom Modals
author: Paul Vincent Aguirre
status: Complete
Tags:
  - "[[Flask]]"
  - "[[JavaScript]]"
  - "[[Dev Log]]"
  - "[[User Experience]]"
  - "[[Custom Modals]]"
  - "[[Dashboard Analytics]]"
---

# 📝 DEV LOG: WEEK 27 - DAY 3 

## 1. Goal of the Day
The focus for Day 3 was to elevate the administrative dashboard's user experience. We wanted to add three high-fidelity analytics cards at the top of the admin panel (Unread Messages, Total Projects, and Completed/Live Projects) and replace the default browser `confirm()` dialogue boxes with custom promise-based glassmorphic modals.

---

## 2. Admin Dashboard Stats Cards
We added the HTML structure and CSS grids for the analytics panel, and wired the numbers to count dynamically:

* **Grid & Card Layout:** Configured a responsive CSS Grid container right below the navigation headers. The three glassmorphic cards display visual emoji icons, clean labels, and large statistics numbers.
* **Animated Value Transitions:** Coded a requestAnimationFrame-driven `animateNumberValue(el, target)` function in JavaScript. On loading or deleting items, the cards animate their counts from the current value to the new value using a smooth quadratic ease-out transition.
* **Velocity Metrics:** The first card tracks active unread messages, the second tracks total projects count, and the third filters local state to show Completed/Live projects.

---

## 3. Promise-Based Custom Dialog Modals
To prevent abrupt system alerts and maintain visual consistency with the dark theme, we replaced all deletion confirmation prompts:

* **Glassmorphism Modals:** Created a backdrop-blur overlay `#confirm-modal` containing a modal card with customizable title, text, cancel buttons, and delete actions.
* **Asynchronous Handler:** Developed a reusable `showConfirm(title, message)` function that opens the overlay modal and resolves a Promise to `true` or `false` depending on button clicks.
* **Sensitive Actions integration:** Updated the event listeners for message deletion and project deletion in `admin.js` to `await` the Promise return value before calling delete endpoints and updating the local view.

---

## 4. Verification & Clean-up
* **Backend Stability:** Verified that all backend pytest files compile and run successfully (39 out of 39 passed).
* **Git Clean-up:** Checked and verified git commits, ensuring all changes were properly tracked and committed under clean, structured messages.

---
