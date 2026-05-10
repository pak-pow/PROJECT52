---
date: 2026-05-10
project: Project 52 - Phase 2 (Integration)
module: Week 19 - Markdown Note-Taking App
topic: Weekly Retrospective
Tags:
  - "[[Full-Stack Architecture]]"
  - "[[REST API]]"
---
# WEEK 19

## 1. The Core Achievement
This week marked the transition from isolated scripts to a fully integrated ecosystem. I successfully decoupled the Presentation Layer (Frontend) from the Data/Logic Layer (Backend) and established a communication bridge using the HTTP protocol. 

## 2. Key Engineering Concepts Mastered
1. **REST API Design:** Built a Python Flask server from scratch, explicitly defining routes for `GET`, `POST`, `PUT`, and `DELETE` methods.
2. **CORS (Cross-Origin Resource Sharing):** Learned how browser security models block unauthorized local connections, and how to configure a Flask firewall (`@app.after_request`) to whitelist specific domains, headers, and preflight (`OPTIONS`) checks.
3. **Vanilla JS State Management:** Evolved past simple DOM manipulation by introducing a global state memory (`currentNote`, `isDirty`). This allowed the application to make intelligent UI decisions, such as warning the user before discarding unsaved work.
4. **Third-Party Integrations:** Injected an external parsing engine (`marked.js`) into the application, mapping raw textarea data streams into compiled HTML in real-time.
5. **Cybersecurity Fundamentals:** Identified and patched a critical Path Traversal vulnerability by utilizing `werkzeug.utils.secure_filename` to sanitize all incoming payload data before touching the local operating system.

## 3. The Path Forward
The architecture established this week serves as the blueprint for all future web-based applications in Project 52. The mental model of `Client -> Fetch -> API -> OS -> API -> JSON -> Client` is now locked in.
