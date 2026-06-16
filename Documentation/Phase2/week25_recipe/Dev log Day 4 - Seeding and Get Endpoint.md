---
date: 2026-06-16
project: Project 52
week: 25
module: Recipe Sharing Platform (Full Stack)
topic: Day 4 - Data Extraction, Media Streaming, & System Seeding
Tags:
  - "[[Flask Routes]]"
  - "[[Data Extraction]]"
  - "[[Database Seeding]]"
  - "[[System Architecture]]"
  - "[[Dev Log]]"
---
# DEV LOG: WEEK 25 - DAY 4

## 1. Goal of the Day
Day 3 was about saving recipes and images to the server. Day 4 was about getting that data back out. I needed to build routes that let the app fetch recipes as a list and display uploaded images in the browser. I also added a script to quickly fill the database with sample recipes so I don't have to manually enter test data every time.

---

## 2. Fetching Recipes from the Database (`recipe_model.py`)
- **Newest first:** Added a function that grabs all recipes from the database, sorted so the most recently added ones show up at the top.
- **Format conversion:** The database returns data in its own internal format, so I converted each recipe into a regular Python dictionary — the format Flask needs to send it back as a proper response.

---

## 3. The Two New Routes (`recipe_routes.py`)
- **GET /api/recipes:** A route that returns the full list of recipes when requested. It asks the database for everything, then sends it back as a neat JSON list.
  
- **GET /uploads/\<filename\>:** A route that serves uploaded images to the browser. Browsers are not allowed to look directly at files sitting on a server's hard drive, so this route acts as the middleman — it finds the right image in the uploads folder and sends it over safely. It also automatically tells the browser what kind of file it is (e.g. a JPEG), so it knows to display it as an image rather than gibberish.

---

## 4. Sample Data Script (`seed.py`)
- **Why it exists:** Instead of manually filling out the recipe upload form over and over just to have test data, I wrote a script that adds 5 ready-made recipes to the database in one go.
- **What it adds:** 5 full recipes, each with a title, description, ingredient list, and numbered steps — written to look like real user submissions.
- **The benefit:** Whenever the database needs to be reset and refilled, just run `python seed.py` and it's done instantly. Makes testing the frontend much faster.