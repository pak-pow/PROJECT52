---
Date: 2026-01-21
Project: Data Visualization Engine
Tech Stack: Python, Matplotlib
Status: 🎛️ Dashboard Operational
Tags:
  - "[[Subplots]]"
  - "[[Object-Oriented Plotting]]"
  - "[[Data Dashboards]]"
---
## 1. The Initiative
Day 4 focused on **Aggregation**.
Previously, I created isolated charts. Today, I combined them into a unified **Mission Control Dashboard**.
The goal was to render 4 distinct visualizations (Line, Histogram, Scatter, Bar) into a single 2x2 grid window using Matplotlib's subplot system.

## 2. The Concepts

### Concept A: The Object-Oriented Interface
In Days 1-3, I used the "State-based" interface (`plt.plot()`). It was simple but limited.
Today, I switched to the "Object-Oriented" interface:
* **`fig` (The Figure):** The physical window or canvas.
* **`axs` (The Axes):** The individual sub-plots drawn on the canvas.
* **Why?** It gives precise control. I can tell `axs[0,0]` to be a Line Chart and `axs[1,1]` to be a Bar Chart without them conflicting.

### Concept B: The Grid System (`plt.subplots`)
I initialized a 2x2 grid:
```python
fig, axs = plt.subplots(2, 2, figsize=(12, 10))
````

This creates a matrix of charts accessible by index:

- Top-Left: `axs[0, 0]`
    
- Top-Right: `axs[0, 1]`
    
- Bottom-Left: `axs[1, 0]`
    
- Bottom-Right: `axs[1, 1]`
    

### Concept C: Tight Layout

When packing 4 charts together, labels often overlap.

plt.tight_layout() is an automatic layout engine that adjusts padding between subplots to ensure text remains readable.

## 3. The Code Specimen

_Targeting a specific quadrant for plotting:_

``` Python
# BOTTOM LEFT [1, 0]: Correlation Scatter Plot
axs[1, 0].scatter(study_hours, grades, color='cyan')
axs[1, 0].set_title('Study vs Grades')
axs[1, 0].set_facecolor('#222')
```

## 4. Visual Proof

_The 2x2 Data Command Center showing diverse dataset visualizations._

![Day4](../../../Picture/week4/day4.png)
