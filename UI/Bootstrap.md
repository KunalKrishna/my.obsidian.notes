[Get started with Bootstrap · Bootstrap v5.3](https://getbootstrap.com/docs/5.3/getting-started/introduction/)

# Bootstrap Mental Model
Bootstrap is essentially a **Design System API**. You don't need to learn every class; you just need to understand the **four core namespaces** and the **naming logic**

## 1. The Core Architecture: The "Onion" Rule
Bootstrap follows a strict containment hierarchy. If you break this order, your UI will "break" or look unaligned.
1. **Container (`.container` or `.container-fluid`):** The boundary. It sets the max-width and centers your content.
2. **Row (`.row`):** A wrapper that acts as a Flexbox container. Its only job is to hold Columns.
3. **Column (`.col-*`):** The actual "bucket" where your content (text, buttons, forms) lives.

![[bootstrap.png]]


---
## 2. Deciphering the Naming Convention
Bootstrap classes follow a **Property-Breakpoint-Size** syntax. Once you see the pattern, you can "guess" the class name without looking at docs.
### The Formula: `[Property]-[Breakpoint]-[Value]`
- **Property:** What are you changing? (`m` for margin, `p` for padding, `d` for display, `col` for column).
- **Breakpoint (Optional):** When should this happen? (`sm`, `md`, `lg`, `xl`). If omitted, it applies to _all_ screen sizes.
- **Value:** How much? (Usually `0` to `5` for spacing, or `1` to `12` for grid columns).
**Example:** `mb-md-5`
- `mb`: Margin-Bottom
- `md`: Starting from Medium screens (tablets) and up...
- `5`: Use the maximum spacing value.

---
## 3. The Four Class Groups
To be productive, categorize every class you encounter into one of these four buckets:
### A. Layout (The Skeleton)
These define the structure.
- **Grid:** `row`, `col-6` (takes half the width), `col-md-4` (takes 1/3 width on desktop).
- **Containers:** `container` (fixed width), `container-fluid` (full width).
### B. Components (The Pre-built Widgets)
These are high-level UI elements. They usually require a "base class" plus "modifier classes."
- **Buttons:** `.btn` (base) + `.btn-primary` (blue color).
- **Navbars:** `.navbar` (base) + `.navbar-dark` (theme) + `.fixed-top` (position).
### C. Utilities (The "Quick Fixes")
These do one specific CSS job. They are the most used for "tweaking."
- **Spacing:** `mt-3` (margin-top), `py-2` (padding top & bottom).
- **Flexbox:** `d-flex` (turn on flex), `justify-content-center` (center horizontally).
- **Colors:** `text-success` (green text), `bg-light` (grey background).
### D. Forms (The Input Layer)
Standardizing user input.
- **Input fields:** `.form-control`.
- **Labels:** `.form-label`.
- **Layout:** `.mb-3` is the standard "gap" between form rows.

---
## 4. The "Ninja" Workflow

Instead of starting from scratch, follow this 3-step loop:
1. **Identify the Component:** "I need a Navigation Bar."
2. **Find the Base Code:** Go to [Bootstrap Docs > Components > Navbar](https://getbootstrap.com/docs/5.3/components/navbar/), and copy the simplest example.
3. **Apply Utilities:** "I want this navbar to be blue and have a margin at the bottom."
    - Change `bg-light` to `bg-primary`.
    - Add `mb-4` to the `<nav>` tag.

---
## 5. Summary Table for Quick Reference

|**Prefix**|**Category**|**Example**|**Logic**|
|---|---|---|---|
|`col-`|Grid|`col-md-6`|Use 6 out of 12 slots on medium screens.|
|`m-` / `p-`|Spacing|`pt-5`|Padding-Top level 5.|
|`btn-`|Buttons|`btn-outline-danger`|A red "outline" style button.|
|`d-`|Display|`d-none d-lg-block`|Hide on mobile, show on Large screens.|
|`w-` / `h-`|Sizing|`w-100`|Width 100%.|


# Layout
## Breakpoints 
Breakpoints are customizable widths that determine how your responsive layout behaves across device or viewport sizes in Bootstrap.

**Available breakpoints** 
Bootstrap includes six default breakpoints, sometimes referred to as _grid tiers_, for building responsively. These breakpoints can be customized if you’re using our source Sass files.

|Breakpoint|Class infix|Dimensions|
|---|---|---|
|Extra small|_None_|<576px|
|Small|`sm`|≥576px|
|Medium|`md`|≥768px|
|Large|`lg`|≥992px|
|Extra large|`xl`|≥1200px|
|Extra extra large|`xxl`|≥1400px|
Each breakpoint was chosen to comfortably hold containers whose widths are multiples of 12.

## Containers
Containers are a fundamental building block of Bootstrap that contain, pad, and align your content within a given device or viewport.

Bootstrap comes with three different containers:
- `.container`, which sets a `max-width` at each responsive breakpoint
- `.container-{breakpoint}`, which is `width: 100%` until the specified breakpoint
- `.container-fluid`, which is `width: 100%` at all breakpoints

The table below illustrates how each container’s `max-width` compares to the original `.container` and `.container-fluid` across each breakpoint.

|                    | Extra small <576px | Small ≥576px | Medium ≥768px | Large  ≥992px | X-Large ≥1200px | XX-Large  ≥1400px |
| ------------------ | ------------------ | ------------ | ------------- | ------------- | --------------- | ----------------- |
| `.container`       | 100%               | 540px        | 720px         | 960px         | 1140px          | 1320px            |
| `.container-sm`    | 100%               | 540px        | 720px         | 960px         | 1140px          | 1320px            |
| `.container-md`    | 100%               | 100%         | 720px         | 960px         | 1140px          | 1320px            |
| `.container-lg`    | 100%               | 100%         | 100%          | 960px         | 1140px          | 1320px            |
| `.container-xl`    | 100%               | 100%         | 100%          | 100%          | 1140px          | 1320px            |
| `.container-xxl`   | 100%               | 100%         | 100%          | 100%          | 100%            | 1320px            |
| `.container-fluid` | 100%               | 100%         | 100%          | 100%          | 100%            | 100%              |

![[Pasted image 20260219001236.png]]
To understand these three, think of them as **"The Shock Absorbers"** of your webpage. Their only job is to decide how much "white space" (margins) should exist on the left and right sides of your content as the browser window stretches or shrinks.
### 1. `.container` (The Fixed-Width Jumper)

This is the most common choice. It doesn't grow smoothly; it "jumps" to a specific width at certain screen sizes (breakpoints) and then centers itself.
- **How it behaves:** On a small phone, it's 100% wide. As the screen gets bigger (Tablet → Laptop → Desktop), it snaps to a fixed pixel width (e.g., 720px, 960px, 1140px) and stays there until the next "jump."
- **Use Case:** **Standard Websites** (Blogs, E-commerce product pages, Articles). It keeps the text from stretching too wide on a massive monitor, which makes it easier to read.
### 2. `.container-fluid` (The Edge-to-Edge)
This is the "100% width" option. It ignores breakpoints and always touches the left and right edges of the browser, no matter how big the screen is.
- **How it behaves:** It is **always** `width: 100%`. There are no side margins unless you manually add them with padding utilities like `px-5`.
- **Use Case:** **Dashboards and Admin Panels.** When you have a complex data table or a map, you want to use every single pixel of screen real estate available.
### 3. `.container-{breakpoint}` (The Hybrid)
This is a "conditional" container. It acts like `.container-fluid` (100% wide) **until** it hits a specific size, and then it starts behaving like a regular `.container`.
- **Logic:** `100% wide UNTIL [breakpoint], then SNAP to fixed width.`
- **Example: `.container-md`**
    - On a Phone (`sm`): 100% wide.
    - On a Tablet (`md`): Snaps to fixed width and adds margins.
- **Use Case:** **Responsive Landing Pages.** You might want your hero section to be full-width on mobile to save space, but you want it to look "contained" and centered once the user views it on a desktop.
### Quick Comparison Table

|**Class**|**Mobile (<576px)**|**Desktop (>1200px)**|**Vibe**|
|---|---|---|---|
|`.container`|100%|**Fixed** (e.g., 1140px)|Traditional, clean margins.|
|`.container-fluid`|100%|**100%**|Modern, "App-like," expansive.|
|`.container-lg`|100%|**Fixed**|Fluid for mobile/tablet, fixed for PC.|
### Ninja Tip: How to Choose?
If you are building your **Portfolio Website**, start with `.container`. It provides those nice professional gutters on the side so your text isn't touching the very edge of the screen on a laptop.

## Grid
## Columns
## Gutters
## Z-index
##  CSS Grid
