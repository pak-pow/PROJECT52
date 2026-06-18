---
date: 2026-06-18
project: Project 52
week: 25
module: Recipe Sharing Platform (Full Stack)
topic: Day 6 - System Audit, Security Refactor, & UI Overhaul
Tags:
  - "[[System Architecture]]"
  - "[[Security]]"
  - "[[CRUD]]"
  - "[[UI/UX]]"
  - "[[Dev Log]]"
---
# 📝 DEV LOG: WEEK 25 - DAY 6

**1. Goal of the Sprint**

The original plan was just to build an upload form, but after reviewing the code, the goals expanded. The focus shifted to fixing old bugs, securing the app, adding the ability to edit recipes, and completely upgrading the design so it looks and feels like a professional product.

## 2. Backend Security & Bug Fixes

- **Closing Database Connections:** Fixed a major issue where the app was leaving database connections open, which wastes memory. The app now properly closes these connections after every request.
- **Saving Database Changes:** Fixed a bug where database changes weren't actually saving because of a missing set of parentheses (`()`) in the code.
- **Preventing Crashes and Securing Access:** Stopped the server from crashing when a user searches for a recipe that doesn't exist. Also improved security by only allowing trusted local connections to access the database.
- **Preventing Duplicate Test Data:** Updated the test data script so it checks if the sample recipes are already there before adding them. This stops annoying duplicates from stacking up every time you restart the server.

## 3. Frontend Security & Cleanup

- **Stopping Malicious Text:** Fixed a security hole where users could type harmful code into the forms. The app now safely cleans and filters all text before displaying it on the screen.
- **Centralized Settings:** Removed scattered web addresses hidden in the code. Created a single settings file (`config.js`) so the app's main connections are managed in one easy-to-find spot.
- **Handling Missing Images:** Added a backup plan for images. If a recipe photo is missing or fails to load, a clean placeholder image shows up instead of a broken link icon.

## 4. New Features

- **Editing Recipes:** Finished the core app features by adding an "Edit" button. Users can now open a form that's already filled out to change their existing recipes.
- **Page Numbers:** Added page limits so the app doesn't try to load thousands of recipes at once. Added "Previous/Next" buttons to make browsing faster and easier.
- **Better Notifications:** Removed the ugly, standard browser pop-up alerts. Replaced them with sleek, custom notifications that slide in to tell the user if an action succeeded or failed.

## 5. UI & Design Overhaul

- **Redesigned Recipe View:** Removed the old drop-down text. Built a wide, clean pop-up window that splits the recipe image and instructions into a nice side-by-side layout.
- **Keyboard Shortcuts:** Made the app easier to use by allowing users to press the `Escape` key to close any open pop-up windows.
- **Professional Icons:** Replaced the basic text emojis with sharp, professional-looking icons from a design library, making the app look much more polished.
- **Fixing the Grid:** Corrected a styling typo so the main screen actually looks like a proper grid of cards instead of one long, stacked list.