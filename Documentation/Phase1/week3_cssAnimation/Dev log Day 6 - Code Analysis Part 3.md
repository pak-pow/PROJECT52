---
Date: 2026-01-16
Project: Week 3 - Museum of Motion
Topic: CSS Styling & Animation Logic
Tags:
  - "[[CSS Grid]]"
  - "[[Keyframes]]"
  - "[[3D Transforms]]"
  - "[[Pseudo-elements]]"
  - "[[Documentation]]"
---
## 1. The Layout Engine (Architecture)
The gallery uses a "responsive-by-default" strategy, meaning it adapts to screen sizes without complex media queries.

### The Foundation
```css
* {
    box-sizing: border-box; 
    /* CRITICAL: Changes how width is calculated. 
       Without this, adding padding makes an element wider. 
       With this, padding stays INSIDE the box. */
}
````

### The Grid System

``` CSS
.gallery {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(400px, 1fr));
}
```

- **`repeat(auto-fit, ...)`**: The browser calculates how many columns fit on the screen automatically.
- **`minmax(400px, 1fr)`**: "Columns must be at least 400px. If there's extra space, share it equally (`1fr`)."

---

## 2. The Animation Engine (Motion)

The museum demonstrates three distinct types of motion.

### Type A: Transitions (State Change)

- **Used in:** Exhibit 1 (The Morph)
- **Logic:** `transition: all 0.3s ease-in-out;`
- **Concept:** Interpolates values smoothly between State A (Normal) and State B (Hover).

### Type B: Keyframes (Loops)

- **Used in:** Exhibit 2 (Loader), Exhibit 8 (Neon)
- **Logic:** `@keyframes spin { 0% {...} 100% {...} }`
- **Concept:** A timeline script that runs continuously.
- **Physics:**
    
    - `linear`: Constant speed (Robot/Machine).
    - `ease-in-out`: Slow start/stop (Organic/Breathing).

### Type C: Physics Simulation (Gravity)

- **Used in:** Exhibit 5 (Bouncing Ball)
- **Logic:** `cubic-bezier(0.7, 0.04, 0.9, 1)`    
- **Concept:** Overrides standard timing to mimic mass and weight. The curve defines acceleration (gravity pulling down) and deceleration (momentum fighting gravity).

---

## 3. The 3D Engine (Depth)

Exhibit 4 (The Flip) proves that CSS is a 3D rendering engine. It requires a strict 3-stage hierarchy:

1. **The Scene (Parent):** `perspective: 1000px;` establishes the vanishing point.
2. **The Object (Child):** `transform-style: preserve-3d;` prevents the browser from flattening the object.
3. **The Faces (Grandchild):** `backface-visibility: hidden;` ensures we don't see the "back" of a texture when it turns around.    

---

## 4. Advanced Visual Effects (Illusion)

### The Glitch (Pseudo-elements)

Instead of adding extra HTML tags, we use CSS to clone the text:

- `content: attr(data-text);` pulls the text "ERROR 404" from the HTML.
- `clip-path: inset(...)` slices the clone into visible strips.
- `animation` moves those strips rapidly to create noise.

### The Shimmer (Skeleton Loading)

We do not move the `div`. We move the **Background**:

- **Setup:** Create a gradient that is 200% as wide as the box.
- **Action:** Slide the background position from left (`-200%`) to right (`200%`).
- **Result:** Illusion of light passing through solid matter.