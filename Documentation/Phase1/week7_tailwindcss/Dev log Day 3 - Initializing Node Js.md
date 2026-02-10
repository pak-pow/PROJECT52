---
date: 2026-02-10
project:
  - Nexus Landing Page
topic: Infrastructure Upgrade
Tags:
  - "[[Build-Pipeline]]"
  - "[[Tailwind Setup]]"
  - "[[Tailwind]]"
  - "[[CSS Grid]]"
  - "[[Programming]]"
  - "[[HTML]]"
  - "[[CSS]]"
---
# 📝 DEV LOG: WEEK 07 - DAY 3

**Focus:** Transitioning from CDN to a Professional Build Pipeline

## 1. The Initiative
Today marked the shift from "Prototyping" to "Engineering."

Previously, I was using the **CDN method** (loading Tailwind via a `<script>` tag). While fast for testing, this is bad practice for production because:
- It downloads the entire library (megabytes of data) every time the page loads.
- It prevents customization of the theme file.
- It relies on an internet connection to work.

**The Goal:** Install the Tailwind CLI (Command Line Interface) locally to compile a custom, lightweight CSS file.

---

## 2. The Struggle (Dependency Hell)
I encountered a critical versioning conflict during installation.

* **The Issue:** Running `npm install -D tailwindcss` installed the brand-new **Version 4.0** by default. Version 4 changes the architecture completely, removing the `bin` folder and breaking standard CLI commands.
* **The Symptom:** The terminal reported `"added 1 package"` instead of the expected ~80 dependencies, and commands like `npx tailwindcss init` failed with "command not found."
* **The Fix:** I learned to force `npm` to install a specific, stable version.

**Command Used:**
```bash
npm install -D tailwindcss@3.4.1
````

**Result:** This correctly installed the standard engine (73 packages) and restored the `bin` folder.

---

## 3. The Protocol (Installation & Setup)

### A. Environment Setup
**1. Initialize Node.js Project:**

Creates `package.json` to track my tools.

``` Bash
npm init -y
```

**2. Install Tailwind Engine (Stable v3):**
Downloads the compiler into `node_modules`.

``` Bash
npm install -D tailwindcss@3.4.1
```

**3. Create Configuration File:**
Generates `tailwind.config.js`.

``` Bash
npx tailwindcss init
```
### B. Configuration
I configured `tailwind.config.js` to scan my HTML files for class names:

``` JavaScript
module.exports = {
  content: ["./*.{html,js}"], // Look at ALL HTML files in this folder
  theme: { extend: {} },
  plugins: [],
}
```
### C. The Source File
I created `src/input.css` to act as the "entry point" for the compiler:

``` CSS
@tailwind base;
@tailwind components;
@tailwind utilities;
```

---
## 4. The Build Process (The Watcher)
This was the most critical learning point. I set up a real-time compiler that watches my code changes.
**The Command:**

``` Bash
npx tailwindcss -i ./src/input.css -o ./src/output.css --watch
```

**Breakdown of Flags:**

- `-i ./src/input.css`: **Input.** Take the raw Tailwind directives.    
- `-o ./src/output.css`: **Output.** Spit out a standard CSS file here.
- `--watch`: **Live Mode.** Do not stop; keep running and re-compile every time I save a file.

**Correction Made:**
Initially, I output the file to the root folder (`-o ./output.css`), but my HTML expected it in `src`. I fixed the command to output correctly to `./src/output.css`.

---
## 5. The Feature Grid (CSS Grid & Hover States)
Once the engine was running, I built the **Features Section**.
### Key Techniques Used:
**Responsive Grid Layout:**

``` HTML
<div class="grid grid-cols-1 md:grid-cols-3 gap-8">
```

- `grid-cols-1`: Default mobile view (1 column).
- `md:grid-cols-3`: On medium screens (768px+), switch to 3 columns.

**The "Group Hover" Pattern:**

I wanted the icon to change color when the user hovers over the card.

1. **Parent:** Added `group` class to the card container.
2. **Child:** Added `group-hover:bg-blue-600` to the icon.
3. **Result:** Hovering the parent triggers the child's style.

---

## 6. Standard Operating Procedures (SOP)

### How to Start Working (Start of Day):

1. Open VS Code.    
2. Open Terminal (`Ctrl + ~`).
3. Press **Up Arrow** to find the watch command, or type:
``` Bash
npx tailwindcss -i ./src/input.css -o ./src/output.css --watch
```
4. Wait for `Rebuilding... Done`.

### How to Stop Working (End of Day):

1. Click in the terminal.
2. Press `Ctrl + C` to kill the process.
3. Close VS Code.

---
**Total Time:** 3 Hours
**Mood:** 🟢 Empowered (I own the build process now).