---
Date: 2026-01-14
Project: CSS Animation Showcase
Tech Stack: CSS Steps, Text-Shadow, Gradients
Tags:
  - "[[Discrete Animation]]"
  - "[[Lighting Effects]]"
  - "[[Skeleton Loading]]"
  - "[[UX Patterns]]"
---
## 1. The Initiative
Day 4 focused on **Digital Texture** and **State Indication**.
Previous animations moved objects through space. Today's exhibits moved *light* and *content*. I implemented a Typewriter effect (Discrete animation), a Neon Sign (Lighting simulation), and a Skeleton Loader (State animation).

## 2. The Concepts

### Concept A: Discrete vs. Continuous (`steps()`)
Standard animations (`linear`, `ease`) interpolate values smoothly. A typewriter, however, is mechanical and jagged.
* **The Trick:** Using `steps(30)` forces the browser to jump instantly between frames rather than sliding.

* **Application:** Used on the `width` property to reveal text character-by-character.

### Concept B: Fake Lighting (`text-shadow`)
CSS has no lighting engine. To create a "Neon" effect, I layered multiple shadows.
* **Layer 1:** Sharp, white shadow (The bulb filament).
* **Layer 2:** Soft, colored shadow (The glow).
* **Layer 3:** Wide, diffuse aura (The ambient light).
* **Animation:** Modulating the `opacity` creates a "flickering/bad wiring" effect.

### Concept C: Moving the Landscape (`background-position`)
For the Skeleton Loader (Shimmer), I didn't move the div. I moved the *background*.
* **Technique:** Created a 200% width gradient (Dark -> Light -> Dark).
* **Animation:** Slid the background from left to right. This creates the illusion of a light source passing over a static gray bar.

## 3. The Exhibits

### Exhibit 07: The Typewriter
* **Visual:** Code being typed out one character at a time with a blinking cursor.
* **Key Code:** `animation-timing-function: steps(30, end);`

### Exhibit 08: The Neon Sign
* **Visual:** A buzzing "OPEN" sign with a green/blue aura against a dark background.
* **Key Code:** Stacked `text-shadow` values.

### Exhibit 09: The Shimmer
* **Visual:** A smooth wave of light passing through gray bars, indicating data loading.
* **Key Code:** `linear-gradient` + `background-position`.

## 4. Visual Proof
*The completed gallery featuring all 9 exhibits.*

![Day4](../../../Picture/week3/other_3.png)
