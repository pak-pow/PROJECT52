---
date: 2026-03-23
project: REST API for Todo App
topic: Dynamic Routing, PUT & DELETE Methods
Tags:
  - "[[Python]]"
  - "[[Flask]]"
  - "[[API]]"
  - "[[Backend]]"
  - "[[CRUD]]"
  - "[[Dev Log]]"
---
# 📝 DEV LOG: WEEK 13 - DAY 3

**Core Objective:** Finalize the CRUD lifecycle of the REST API by implementing the `PUT` (Update) and `DELETE` (Destroy) HTTP methods, utilizing dynamic URL routing to target specific data entries.

## 1. The Initiative & Context
With data creation and retrieval fully operational, the application required mutation capabilities. The objective for Day 3 was to engineer endpoints capable of capturing dynamic variables from the URL path (the specific Task ID) to update existing records or remove them entirely from the in-memory state array.

## 2. Architectural Decisions & Concepts

### Concept A: Dynamic URL Routing
Instead of hardcoding paths, I utilized Flask's dynamic routing syntax: `@app.route('/todos/<int:todo_id>')`. 
* This tells the server to expect an integer at the end of the URL and automatically passes that integer into the backend function as the `todo_id` parameter, allowing the engine to know exactly which task the client wants to modify.

### Concept B: The PUT Route (Data Mutation)
* Iterated through the global `todos` list to find the dictionary matching the captured `todo_id`.
* Utilized Python's dictionary `.get()` method to safely extract new values from the incoming JSON payload. If the payload omitted a field (e.g., they only wanted to update "completed" but not the "task" name), the `.get()` method safely falls back to the existing data, preventing accidental data wiping.

### Concept C: The DELETE Route (State Filtering)
* Engineered a destructive route that modifies the global state list.
* Utilized a Python list comprehension (`[todo for todo in todos if todo["id"] != todo_id]`) to rapidly filter the array. It rebuilds the database by keeping every task *except* the one that matches the target ID, effectively deleting it from memory.

## 3. The Output & Result
The API now fully supports all four fundamental CRUD operations. It successfully intercepts dynamic variables, modifies specific data structures in real-time, filters out deleted objects, and returns the appropriate HTTP status codes (200 OK, 201 Created, 404 Not Found).

---