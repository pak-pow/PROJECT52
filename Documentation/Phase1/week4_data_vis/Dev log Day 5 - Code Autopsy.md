---
Date: 2026-01-22
Project: Data Visualization Engine
Tech Stack: Python, Matplotlib
Tags:
  - "[[Code Autopsy]]"
  - "[[Review]]"
  - "[[Data Science]]"
---
## 1. The Mission
Week 4 was dedicated to **Data Visualization**.
I moved from "Terminal Output" (Text) to "Graphical Output" (Insight). I learned to use **Matplotlib** and **NumPy** to generate, process, and visualize synthetic data.

## 2. The Arsenal (Skills Acquired)
* **The Line Chart (`plt.plot`):** For visualizing Time Series and Trends. Used `fill_between` to show positive/negative performance gaps.
* **The Histogram (`plt.hist`):** For visualizing Frequency and Distribution (The Bell Curve).
* **The Scatter Plot (`plt.scatter`):** For visualizing Correlation. Used `np.polyfit` (Linear Regression) to draw mathematical trend lines.
* **The Dashboard (`plt.subplots`):** For visualizing multiple datasets simultaneously using the Object-Oriented interface.

## 3. The Shift
The biggest technical leap this week was moving from **State-based** plotting (Days 1-3) to **Object-Oriented** plotting (Day 4).
* *Old Way:* `plt.plot()` -> Implicitly targets the current window.
* *New Way:* `axs[0,0].plot()` -> Explicitly targets a specific quadrant.

## 4. Final Status
The Data Visualization Engine is fully operational.
* **Total Scripts:** 4 (`main.py`, `histogram.py`, `scatter.py`, `dashboard.py`)
* **Libraries:** Matplotlib, NumPy
* **Visual Proof:** See `assets/images/` for the complete gallery.