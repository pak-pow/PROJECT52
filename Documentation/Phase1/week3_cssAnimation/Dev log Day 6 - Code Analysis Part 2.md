---
Date: 2026-01-16
Project: Week 3 - Museum of Motion
Topic: HTML Body Structure
Tags:
  - "[[HTML Semantics]]"
  - "[[CSS Classes]]"
  - "[[DOM Structure]]"
  - "[[Documentation]]"
---
## 1. High-Level Architecture
The visible page is divided into three semantic zones. This helps screen readers navigate the content and improves SEO.

```html
<body>
    <header>...</header> <main>...</main>     <footer>...</footer> </body>
````

---

## 2. Line-by-Line Analysis: The Gallery Exhibits

I dissected the `<body>` tag to understand how the grid and animations are structured.

### Zone 1: The Header

``` HTML
<header>
    <h1>The Museum of Motion</h1> <p>A Pure Css Animation Showcase</p> </header>
```

### Zone 2: The Main Gallery (`<main>`)

This section contains 9 cards, each following a specific structural pattern.

#### The Pattern (The "Card")

Every exhibit uses this identical wrapper structure:

``` HTML
<div class="card">
    <h2>Name</h2>

    <div class="display-case">
        </div>

    <p>Technique: <code>Code</code></p>
</div>
```

#### Detailed Breakdown of Key Exhibits

**Exhibit 1: The Interactive Element**

``` HTML
<button class="exhibit-1"> Hover me </button>
```

**Exhibit 2: The Empty Container**

``` HTML
<div class="exhibit-2"></div>
```

Exhibit 4: The 3D Complex

This exhibit requires a nested structure to handle 3D space.


``` HTML
<div class="display-case scene">
    <div class="flip-card">
        <div class="card-face front">Front</div>
        <div class="card-face back">Back</div>
    </div>
</div>
```

**Exhibit 6: Data Attributes**


``` HTML
<div class="glitch" data-text="ERROR 404"> ERROR 404</div>
```

**Exhibit 9: The Skeleton Loader**

``` HTML
<div class="skeleton-card">
    <div class="skeleton-line title"></div>
    <div class="skeleton-line text"></div>
    <div class="skeleton-line text short"></div>
    </div>
```

### Zone 3: The Footer


``` HTML
<footer>
    <p>Created by Vincent...</p>
</footer>
```

---

## 3. Key Takeaways

1. **Semantic Tags:** Using `<header>`, `<main>`, and `<footer>` is better practice than using `<div>` for everything.
    
2. **Empty Divs:** In CSS animation, empty `<div>`s are often used as "canvases" for shapes (circles, lines) created purely with code.
    
3. **Class Modifiers:** Classes like `.short` or `.scene` are added to existing elements to modify their behavior without writing entirely new HTML structures.