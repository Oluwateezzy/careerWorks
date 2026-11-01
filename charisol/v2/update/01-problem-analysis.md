# Problem Analysis: Why Generated Prototypes Are Poor

## Executive Summary

The current prototype generation pipeline produces **visually broken, non-runnable output**. The downloaded `.zip` does not contain a working application. The preview shows overlapping elements, missing images, broken layouts, and text collisions. This document identifies the root causes and explains why the current approach **cannot be fixed incrementally** — it requires an architectural shift.

---

## 1. What Users See

From the screenshots of a generated prototype for a skincare brand website (27 screens selected):

| Symptom | Visual Evidence |
|---|---|
| **Overlapping elements** | Text labels ("Testimonial", "SERUM", "NEWS") stack on top of each other |
| **No images** | All product images, hero images, and backgrounds are missing |
| **Broken layout** | Absolute positioning causes elements to pile up in the top-left corner |
| **No typography** | All text renders in the same default size/weight/color |
| **No responsive behavior** | The page renders as a fixed-width block with overflow |
| **Not runnable from zip** | Downloaded source.zip does not produce a working app when extracted |

---

## 2. Root Causes

### 2.1 The Compiler Is a Mechanical CSS/JSX Translator — Not a Page Builder

The current compiler ([compile.js](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-be/workers/generate/compiler/compile.js)) walks the Figma node tree and emits:
- A `<div>` for every node
- CSS class with absolute positions from `node.bounds`
- Background color from the first solid fill

**What it does NOT do:**
- Render images (no `<img>` tags are ever emitted)
- Apply typography (font-family, font-size, font-weight, letter-spacing, line-height from the actual node are ignored; hardcoded to `fontSize: 16, lineHeight: 24px`)
- Handle gradients, shadows, borders, border-radius, or effects
- Apply opacity or blend modes
- Handle auto-layout properly (gap, padding, alignment, grow/shrink)
- Emit semantic HTML (`<nav>`, `<header>`, `<section>`, `<button>`, `<a>`)
- Create responsive layouts
- Handle component variants with visual differences

### 2.2 The Style Module Is Severely Limited

[style.js](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-be/workers/generate/compiler/style.js) — the `styleForNode()` function — extracts only:

```
✅ display: flex (if layoutMode is set)
✅ flex-direction
✅ gap (from itemSpacing)
✅ padding (from layout padding)
✅ position: absolute/relative
✅ width, height (from bounds)
✅ left, top (from bounds, relative to parent)
✅ backgroundColor (from first solid fill OR token variable)
✅ fontSize: 16 (HARDCODED)
✅ lineHeight: 24px (HARDCODED)
✅ color: rgba(17,17,20,1) (HARDCODED)

❌ font-family (Figma nodes carry this)
❌ font-weight
❌ letter-spacing
❌ text-align
❌ text-decoration
❌ border, border-radius
❌ box-shadow
❌ opacity
❌ overflow
❌ background-image / gradient
❌ images (fills of type IMAGE)
❌ strokes
❌ effects (shadow, blur)
❌ constraints / responsive sizing
❌ min/max width/height
```

This means **~80% of the visual properties** that make a design look like a design are discarded.

### 2.3 Images Are Completely Absent

The emit module ([emit.js](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-be/workers/generate/compiler/emit.js)) renders every node as a `<div>`. There is **no code path** that emits `<img>`, `<svg>`, or `background-image`. The Figma plugin exports asset renders as PNGs to S3, but the compiler never references them.

### 2.4 Layout Strategy Causes Visual Collision

The positioning logic in `styleForNode`:
- **Root nodes**: `position: relative` with fixed `width` + `height`
- **In-flex children**: Fixed `width` + `height` (no grow/shrink)
- **All other children**: `position: absolute` with `left` + `top`

When the parent is not a flex container (which is the majority of Figma frames that use "auto layout" or constraints), all children are absolutely positioned relative to the frame's original Figma coordinates. Without proper container sizing, alignment, or overflow, elements pile up.

### 2.5 All 27 Screens Are Generated Simultaneously

The compiler processes all selected screens in one pass. Each becomes a route. But the screens share no meaningful structure — they are independently designed Figma frames with different layouts, typography, and imagery. The compiler treats them identically: walk nodes, emit divs, write CSS.

### 2.6 The Node Data Itself Is Incomplete

The snapshot loader ([snapshotLoader.js](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-be/workers/generate/snapshotLoader.js)) strips nodes down to only:
```javascript
const NODE_FIELDS = ["id", "name", "type", "visible", "layoutProperties", "fills", "textContent"];
```

Missing from the normalized node:
- `strokes`, `effects`, `opacity`, `cornerRadius`
- `constraints`, `layoutAlign`, `layoutGrow`
- `style` (Figma's text style properties)
- `backgroundImageRef`, `imageRef`
- `exportSettings`, `absoluteBoundingBox`

---

## 3. Why This Cannot Be Fixed Incrementally

Adding more CSS properties to `styleForNode()` would improve things marginally, but the fundamental problem is:

> **A deterministic node-to-div translator cannot produce production-quality code.**

Figma designs contain:
- **Visual intent** (this hero section should have a full-bleed background image with overlaid text)
- **Semantic meaning** (this group of links is a navigation bar)
- **Responsive behavior** (this layout should reflow on mobile)
- **Interactive patterns** (this card should be clickable and navigate to a detail page)

The current compiler extracts **none of this**. It sees rectangles with coordinates and colors.

Even with perfect CSS extraction, the output would be:
- Thousands of nested `<div>` elements with pixel-perfect absolute positions
- No semantic HTML
- No responsive behavior
- No maintainable CSS (one class per node)
- No reusable components (promoted components have a flat `{label}` prop and nothing else)

**This is fundamentally the wrong approach for generating usable prototypes.**

---

## 4. The User's Core Insight

> "At the point of extraction we need to build that page or component maybe using another worker and LLM"

This is correct. The key insight is:

1. **Export per-screen or per-component** — not all 27 screens at once
2. **Use an LLM to generate code from the design** — the LLM understands visual intent, semantic meaning, and modern CSS patterns
3. **Generate real components** — with proper HTML, CSS, responsive layout, and image references
4. **Assemble into a working prototype** — combine the generated screens into a Vite+React app with routing

This transforms the pipeline from:

```
Figma JSON → Mechanical div translator → Broken output
```

To:

```
Figma Screen → Visual snapshot + structured data → LLM code generation → Quality component → Assembly into working prototype
```

---

## 5. Key Metrics for Success

| Metric | Current | Target |
|---|---|---|
| Downloaded zip is runnable | ❌ No | ✅ `npm install && npm run dev` works |
| Images render | ❌ None | ✅ All referenced images display |
| Typography matches design | ❌ Hardcoded 16px | ✅ Font family, size, weight, spacing match |
| Layout is correct | ❌ Overlapping divs | ✅ Proper flex/grid, no visual collisions |
| Responsive | ❌ Fixed width | ✅ Adapts to viewport (if design has variants) |
| Semantic HTML | ❌ All divs | ✅ nav, header, section, button, img, a |
| Code quality | ❌ Unmaintainable | ✅ Clean, readable, component-based |
