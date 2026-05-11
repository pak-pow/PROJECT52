---
date: 2026-05-11
project: Project 52 - Week 20
module: E-Commerce Product Catalog
topic: State Management and Array Filtering
Tags:
  - "[[Vanilla JS]]"
  - "[[State Management]]"
  - "[[Data Structures]]"
  - "[[Dev Log]]"
---
# DEV LOG: WEEK 20, DAY 2

## 1. Executive Summary
Day 2 focused on transforming the static product grid into a highly interactive, state-driven storefront. The core objective was implementing a multi-layered filtering system that responds instantly to user input (search text and category selection) without requiring page reloads or additional network requests.

## 2. Global State Architecture
Instead of reading the DOM to figure out what to display, we implemented a "Single Source of Truth" using JavaScript variables:
* `allProducts`: Caches the initial JSON payload from the Python API to prevent unnecessary network calls.
* `activeCategory`: A string tracking the currently selected sidebar filter (defaulting to `'All'`).
* `searchQuery`: A string tracking the current sanitized text inside the search input.

## 3. The Filtering Pipeline (`applyFilters`)
Engineered a functional processing pipeline that sequentially reduces the data array based on the current state:
1. **Category Pass:** Utilized `Array.prototype.filter()` to return only objects matching the `activeCategory` (unless 'All' is selected).
2. **Search Pass:** Chained another `.filter()` using `String.prototype.includes()` to match the `searchQuery` against the product names (converted to lowercase for case-insensitivity).
3. **Re-rendering:** Passed the highly specific, filtered array subset directly back into the `renderProducts()` UI function.

## 4. Event-Driven Reactivity
* Attached an `input` listener to the search bar to capture keystrokes in real-time.
* Attached `click` listeners to the category NodeList, managing the `.active` CSS utility class to provide immediate visual feedback to the user before triggering the filtering pipeline.
