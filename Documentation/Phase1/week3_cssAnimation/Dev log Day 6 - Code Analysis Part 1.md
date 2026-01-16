---
Date: 2026-01-16
Project:
  - Week 3 - Museum of Motion
  - CSS Animation Showcase
Topic: HTML Fundamentals
Tags:
  - "[[HTML Structure]]"
  - "[[Meta Tags]]"
  - "[[Link Attribute]]"
  - "[[Documentation]]"
---
## 1. Line-by-Line Analysis: The HTML Head
I broke down the standard HTML boilerplate to understand the purpose of every tag before the visible `<body>` content begins.

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=0.75">
    <title>Week 3: Museum of Motion</title>
    <link rel="stylesheet" href="style.css">
    </head>
````

---

## 2. Deep Dive: The `rel` Attribute

In the `<link>` tag, the `rel` attribute stands for **Relationship**. It defines how the current document relates to the linked resource.

### The Analogy

> **Think of it like addressing a letter:**
> 
> - `href`: The **Address** (Where is the file?)
>     
> - rel: The Mail Type (Is it a bill? An invitation? A catalog?)
>     Without rel, the browser finds the file but doesn't know how to use it.
>     

### Common `rel` Values

|**Value**|**Code Example**|**Purpose**|
|---|---|---|
|**Stylesheet**|`<link rel="stylesheet" href="style.css">`|Tells the browser to use this file to **style** the page.|
|**Icon**|`<link rel="icon" href="favicon.ico">`|Sets the small icon visible in the browser tab.|
|**Canonical**|`<link rel="canonical" href="...">`|Tells search engines which URL is the "master copy" (SEO).|
|**Preload**|`<link rel="preload" href="font.woff2">`|Instructs the browser to download a heavy resource early (Performance).|
|**Nofollow**|`<a rel="nofollow" href="...">`|(On links) Tells search engines not to "trust" or follow this link.|

---

## 3. Key Takeaway

HTML is not just structure; it is **Metadata**.

- The `<!DOCTYPE>` tells the browser _how_ to read.
- The `<meta>` tags tell the browser _how_ to display (viewport) and decode (charset).
- The `rel` attribute provides **Context** to external links, turning a raw file path into a functional resource.