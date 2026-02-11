---
date: 2026-02-11
project: Nexus Landing Page
topic: Pricing Section & Advanced CSS Layouts
Tags:
  - "[[Absolute Positioning]]"
  - "[[CSS]]"
  - "[[Tailwind]]"
  - "[[Tailwind Setup]]"
  - "[[Dev Log]]"
  - "[[Documentation]]"
  - "[[CSS Transitions]]"
---
# 📝 DEV LOG: WEEK 07 - DAY 4
**Focus:** Building the "Money Maker" (Pricing Section) using Advanced Positioning.

## 1. The Initiative
Today I built the **Pricing Table**, a critical component for any SaaS landing page.
The goal was to create a 3-card layout where the "Pro" (middle) card visually dominates the screen to guide user choice.

## 2. The Concepts (New Skills)

### Concept A: The "Pop-Out" Effect (Transforms)
To make the middle card stand out, I used a scaling transform that only triggers on larger screens.
* **Code:** `md:scale-105`
* **Result:** The browser zooms the element to 105% of its normal size.
* **Why:** It subtly tells the user "This is the important one" without using words.

### Concept B: Absolute Positioning (The Badge)
I created a "MOST POPULAR" badge that floats exactly on the top border of the card.
* **Technique:**
    1. Set Parent to `relative`.
    2. Set Badge to `absolute`.
    3. Center it using the "50% Rule":
       `top-0 left-1/2 -translate-x-1/2 -translate-y-1/2`
* **Why it works:** `left-1/2` pushes it too far right (starts at center), so `-translate-x-1/2` pulls it back by half its own width to center it perfectly.

### Concept C: UI Components (The Toggle)
I built a "Monthly/Yearly" switch purely with CSS shapes.
* **The Track:** A `div` with `rounded-full` (makes it a pill shape).
* **The Knob:** A smaller white circle inside.
* **State:** Visualized the "Yearly" state by adding `translate-x-6` to the knob.

## 3. The Output
A responsive 3-card pricing grid.
- **Mobile:** Cards stack vertically.
- **Desktop:** Cards sit side-by-side, with the middle card scaled up and highlighted with a purple border (`border-purple-500`) and a floating badge.

---
## 4. Standard Operating Procedures (Shut Down)

**To Close for the Day:**
1. Go to the terminal where `Rebuilding... Done` is showing.
2. Press `Ctrl` + `C` to kill the watcher process.
3. Close VS Code.

**To Resume Tomorrow:**
1. Open Terminal.
2. Run: `npx tailwindcss -i ./src/input.css -o ./src/output.css --watch`

---
**Total Time:** 2 Hours
**Next Steps:** Footer Section
