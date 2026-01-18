---
Date: 2026-01-18
Project: Data Visualization Engine
Tech Stack: Python, Matplotlib, NumPy
Status: 📈 Analysis Active
Tags:
  - "[[Data Science]]"
  - "[[Matplotlib]]"
  - "[[NumPy]]"
  - "[[Financial Modeling]]"
---
## 1. The Initiative
Week 4 focuses on **Data Visualization**.
The goal for Day 1 was to build a "Financial Comparison Tool." Instead of just plotting a static line, I simulated two competing assets ("Crypto" vs. "Gold") to understand volatility vs. stability. The script generates new random market data every time it runs.

## 2. The Concepts

### Concept A: The Random Walk (`np.cumsum`)
To simulate realistic stock movement, I didn't just pick random numbers. I used a **Cumulative Sum**.
* **Logic:** Start at $100. Generate a list of "steps" (e.g., +1, -2, +3).
* **Math:** The price at Day 3 is $100 + 1 - 2 + 3 = $102.
* **Code:** `prices = 100 + np.cumsum(changes)`
* *Why?* This ensures the price "walks" from the previous day's value rather than jumping randomly around zero.

### Concept B: Semantic Shading (`fill_between`)
A chart is hard to read if lines just cross each other. I added semantic meaning using conditional fill.
* **Green Zone:** Shaded where Crypto > Gold.
* **Red Zone:** Shaded where Gold > Crypto.
* **Technique:** `interpolate=True` was critical here. It calculates the exact sub-pixel intersection point so the coloring doesn't look "blocky" or jagged.

### Concept C: Data Storytelling (Annotations)
A graph without labels is just art. I used `plt.annotate` to programmatically find the **All-Time High** (ATH) and point to it with an arrow.
* **Dynamic:** The script searches the array `np.max(price_a)` to find the value, so the arrow always points to the correct peak, no matter how the random simulation runs.

## 3. The Code Specimen
*The logic for conditional shading:*
```python
# Green when winning
plt.fill_between(days, price_a, price_b, where=(price_a > price_b), 
                 interpolate=True, color='#00ff88', alpha=0.1)

# Red when losing
plt.fill_between(days, price_a, price_b, where=(price_a <= price_b), 
                 interpolate=True, color='#ff0055', alpha=0.1)
````

## 4. Visual Proof

_The generated chart showing the volatility of Crypto compared to the stability of Gold._

![Day1](../../../Picture/week4/day1.png)