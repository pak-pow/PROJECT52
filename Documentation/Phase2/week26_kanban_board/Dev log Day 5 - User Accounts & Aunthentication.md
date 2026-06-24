---
date: 2026-06-24
project: Project 52
week: 26
module: Kanban Board (Trello Clone)
topic: Day 5 - User Accounts & Authentication
author: Paul Vincent Aguirre
status: Complete
Tags:
  - "[[User Authentication]]"
  - "[[Data Privacy]]"
  - "[[Dev Log]]"
  - "[[User Experience]]"
  - "[[Authentication]]"
---

# 📝 DEV LOG: WEEK 26 - DAY 5 (USER ACCOUNTS & AUTHENTICATION)

## 1. Goal of the Day
The focus for Day 5 was to transform our Kanban application into a secure, multi-user system. We wanted to let different people create accounts, log in, and work on their own private boards without seeing or interfering with anyone else's projects.

---

## 2. Database Upgrades & Data Safety
To support multiple accounts, we updated the backend database structure and moved the existing boards over to the new format:

* **Linked Tables:** We added a `users` table for credentials, a `sessions` table to track logged-in users, and added a user connection to the `boards` table.
* **Saving Old Boards:** To make sure we didn't lose any of your existing work, we wrote a migration tool that automatically created a default `guest` account and transferred all 3 of your original boards to it. 
* **Database Fixes:** We repaired a couple of minor typos in the database setup file (a trailing comma and a missing closing bracket) that were blocking clean database launches.

---

## 3. Account Sign-Up & Log-In
We built secure backend pathways and a beautiful frontend screen to handle user entry:

* **Premium Entry Screen:** We designed a clean, centered login page. It lets you type in your username and password, with a simple toggle link to switch between signing in or registering a new account.
* **Protected Passwords:** Passwords are never saved as plain text. The system automatically turns them into complex, encrypted hashes using secure encryption standards.
* **Tokens Instead of Cookies:** When you log in, the server gives your browser a long random text key (a token). This key is saved in your browser's local memory and sent along with every API request so the server knows who you are. This setup completely protects us from cookie-hijacking attacks.

---

## 4. Keeping Data Private (Security Guards)
To make sure a user cannot view or alter another user's board by editing the URL address:

* **Board Isolation:** The server checks every board request to ensure it belongs to the logged-in user. If a user tries to access a board that isn't theirs, the server returns a `404 Not Found` response.
* **Column & Card Ownership Guards:** When you edit columns or tasks, the server verifies that their parent board is owned by you.
* **Rigorous Test Validation:** We created a suite of **11 user isolation test cases** to prove that User B cannot read, rename, edit, or delete any boards, columns, or cards owned by User A.

---

## 5. UI and Layout Polish
To make the app feel cohesive and premium:
* **Dynamic Header Bar:** We added a sticky top navigation header showing "Kanban Space", greeting you by name, and including a "Log Out" button.
* **Perfect Alignment:** Adjusted button heights in the board header to match the search input bar exactly, keeping the layout clean and balanced.
* **Theme Leak Cleanup:** We fixed an issue where custom board accent colors (like the pink color from the Habit Tracker) would bleed onto the main workspaces dashboard. Now, returning to the dashboard or signing out instantly resets the color scheme to the default brand indigo theme.