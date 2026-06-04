---
date: 2026-06-04
project: Project 52
week: 23
module: Full-Stack Expense Tracker with Charts
topic: Day 6 - Real-Time Filtering & CSV Export
author: Paul Vincent Aguirre
status: In Progress
Tags:
  - "[[Software Engineering]]"
  - "[[Data Export]]"
  - "[[Dev Log]]"
---
# DEV LOG: WEEK 23 - DAY 6 

## 1. Goal of the Day
Now that the app looks great and runs securely, Day 6 was all about adding "Power User" features. We wanted to give users the ability to search through their data easily and actually own their financial records by exporting them. 

## 2. Real-Time Data Filtering
Before today, the dashboard just dumped every single expense on the screen at once. We built a system to let users slice and dice their data.

* **Smart Database Queries:** We upgraded the backend Python code so it can now listen for specific requests, like "Only give me expenses from May 1st to May 31st" or "Only show me Food." We kept the code clean by putting this logic directly into our Database Models.
* **The Filter Bar:** We added a clean, minimalist filter bar above the table where users can pick a start date, end date, and category. 
* **Real-Time Updates:** Instead of making the user click a clunky "Filter" button, we wired the inputs so that the exact millisecond you change a date or pick a category, the table and the chart instantly redraw themselves to match. It feels incredibly fast and responsive.

## 3. CSV Data Export
A true financial app lets users take their data with them. We built a system to let users export their expenses to Excel or Google Sheets.

* **How it Works:** We added a "⬇ Download CSV" button. When clicked, the JavaScript looks at whatever data is currently showing on your screen (respecting your active filters). 
* **Building the File:** The code takes that data, wraps the text in quotes (so commas inside descriptions don't break the file), and stitches it together into a standard CSV (Comma Separated Values) format.
* **Instant Download:** Finally, the code uses a neat browser trick to bundle that text into a hidden file and forces your computer to download it instantly as `expenses_export_[DATE].csv`.

