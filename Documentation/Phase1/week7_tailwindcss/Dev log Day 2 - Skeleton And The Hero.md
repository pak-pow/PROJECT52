---
Date: 2026-02-09
Project: Nexus Landing Page
Tech Stack: HTML5, Tailwind CSS
Tags:
  - "[[Glassmorphism]]"
  - "[[Sticky Nav]]"
  - "[[Gradients]]"
---

## 1. The Initiative
Day 2 focused on **Structural Foundations**.
We moved from a blank page to a professional "Hero" layout. The goal was to create a high-impact first impression using modern UI trends like blurred backgrounds and glowing text.

## 2. The Concepts

### Concept A: The Sticky Glass Nav
We created a navigation bar that stays fixed at the top of the viewport.
* **Sticky:** `sticky top-0 z-50` ensures it floats above other content.
* **Glass Effect:** `backdrop-blur-lg` creates the frosted glass look by blurring whatever scrolls behind it.

### Concept B: The Glow (Ambient Lighting)
Modern sites often use "blobs" of color to create depth.
* **Technique:** An empty `div` positioned absolutely behind the text.
* **Code:** `blur-[100px]` diffuses the color so it looks like light, not a shape.

## 3. The Code Specimen
*The specific combination of classes that creates the gradient text:*
```html
<span class="bg-clip-text text-transparent bg-gradient-to-r from-blue-400 via-purple-500 to-pink-500">
    one block at a time.
</span>
````

## 4. The Output
A fully responsive, dark-mode landing page with a sticky header and interactive hover states.
