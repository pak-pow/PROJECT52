---
date: 2026-04-15
project: Project 52 - Phase 2 Hub
topic: Monorepo Architecture & Local File Routing
Tags:
  - "[[Architecture]]"
  - "[[HTML5]]"
  - "[[CSS3]]"
  - "[[Dev Log]]"
---

# 📝 DEV LOG: PHASE 2 - INTEGRATION HUB

**Core Objective:** Design and implement a centralized "Command Center" dashboard to unify all independent Phase 2 modules (Todo API, Todo UI, Auth System, and Static Blog) into a single, easily navigable interface.

## 1. Architecture Decision: The Monorepo
During the deployment phase of the Week 16 Static Blog, an architectural crossroad was reached: Deploy the static site to its own independent GitHub repository (Multi-repo) or maintain everything within the overarching `PROJECT52-PHASE2` folder. 

The **Monorepo** approach was selected. 
* **The Benefit:** Keeping all Phase 2 code in one singular vault guarantees that the entire ecosystem can be cloned, modified, and run locally without having to manage multiple git remotes or track down disparate repositories. It keeps the workspace highly organized.

## 2. Local File Routing Implementation
Because the modules are maintained locally, the Hub (`index.html`) relies on strict relative file paths rather than external URLs.
* **Standard Routing:** The Week 14 UI and Week 16 Blog were linked using standard root-level relative paths: `href="./week16_static_blog/index.html"`.
* **Deep Directory Routing:** The Week 15 Auth System required pathing specifically into its source folder to bypass backend files: `href="./week15_auth_system/src/index.html"`.

## 3. UI/UX Design of the Hub
The Command Center was designed to be lightweight, fast, and visually distinct from the projects it links to.
* **Styling:** A dark-mode, (`#0a0a0a` background, monospace typography).
* **Visual Hierarchy:** CSS gradients (`linear-gradient`) and text-clipping were utilized for the main header, while interactive hover states (translating the Y-axis and glowing box-shadows) were applied to the anchor tags to provide immediate tactile feedback during navigation.
* **Categorization:** Links pointing to APIs/Backends were given a distinct green neon styling (`.api-link`) to visually separate them from the standard white Frontend links, instantly communicating to the user what type of environment they are about to open.

## 4. Output
A single click from the root directory launches a dashboard granting immediate access to every decoupled frontend, backend, and static module built over the last four weeks.

---
