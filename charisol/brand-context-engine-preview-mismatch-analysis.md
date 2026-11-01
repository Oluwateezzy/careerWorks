# Brand Context Engine: Preview Mismatch & Semantic Token Compliance

## Background

The Brand Context Engine (BCE) is a 5-step wizard that extracts brand DNA (colors, typography, feel) from uploaded assets, merges those extractions by priority, generates a complete token set and component library, and then lets the user approve and save. Two distinct but interrelated bugs have been identified.

---

## Bug 1 — Visual Preview Mismatch (Step 5 vs Component Settings)

### Symptom

| Location | What renders |
|---|---|
| BCE Step 5 "Generated Components" preview (images 1–2) | Rich, fully styled buttons — primary is solid purple, secondary is branded, outline has a visible border, badge variants have color |
| Component Settings "Component Preview" panel (images 3–4) after saving | Plain, unstyled box — Button primary shows as a black rectangle, outline shows almost nothing |

### Root Cause Analysis

#### Two Completely Different Renderers

**BCE Wizard Preview** (`GenerationPreview.tsx` → `renderLiveComponentPreview`)

This renderer is a hardcoded visual mock. It does **NOT** consume the generated component definition at all. It just looks at the `variantName` string and applies hardcoded inline styles with fallbacks:

```tsx
// GenerationPreview.tsx — hardcoded fake renderer
backgroundColor: isPrimary
  ? "var(--color-primary, var(--color-brand-primary, #7C3AED))"  // ALWAYS purple fallback
  : isSecondary
  ? "var(--color-secondary, var(--color-brand-secondary, #4B5563))"
...
```

The CSS variables (`--color-primary`, `--color-brand-primary`) are resolved by the browser using the page-level CSS variables defined in the app's global stylesheet — **not** the generated tokens. The fallback `#7C3AED` (Tailwind violet-600) is always present, so the button always looks purple in the wizard regardless of what the brand color actually is.

**Component Settings Preview** (`TokenPreviewPanel.tsx` → `renderComponent`)

This renderer properly resolves token references through the `variables` map (the project's saved design token store). Its pipeline:

1. Get `node.properties` → for each property, call `resolveVariableValue(prop.value)`.
2. `resolveVariableValue` regex-replaces `var(--xxx)` with the actual token value from `variables[tokenName]`.
3. Pass resolved values as inline `style` to `React.createElement(htmlTag, ...)`.

**The core problem**: after saving, component properties contain values like `var(--color-text-primary)`, `var(--space-4)`, `var(--radius-md)`. For these to resolve, the token **must exist** in the project's `variables` store. If it doesn't, `resolveVariableValue` returns the original unresolved string, which browsers ignore in inline styles → unstyled element.

---

### The Missing Token Chain — Critical

When BCE generates tokens (`generateFromBrandContext`), it creates three layers:

| Layer | Tokens Generated |
|---|---|
| **brand** | `--color-brand-primary`, `--color-brand-secondary`, `--color-brand-accent`, `--color-neutral-N`, `--color-palette-N` |
| **semantic** | `--color-text-primary`, `--color-action-primary-bg`, `--color-bg-canvas`, `--color-bg-surface`, `--color-accent`, `--color-state-*`, `--color-text-link`, `--color-text-secondary`, `--color-action-secondary-*` |
| **typography** | `--font-family-heading/body`, `--font-size-*`, `--font-weight-*` |

The component properties (from `getDefaultComponentProperties`) reference many tokens that are NEVER generated:

| Token Referenced by Components | Generated? | Impact if Missing |
|---|---|---|
| `--space-2` | ❌ MISSING | No gap/padding on buttons, badges, tooltips |
| `--space-3` | ❌ MISSING | No padding on inputs |
| `--space-4` | ❌ MISSING | No padding on cards, alerts |
| `--space-6` | ❌ MISSING | No padding on cards |
| `--radius-md` | ❌ MISSING | No border-radius on buttons, inputs, cards |
| `--radius-lg` | ❌ MISSING | No border-radius on cards |
| `--radius-full` | ❌ MISSING | No border-radius on badges, avatars |
| `--radius-xl` | ❌ MISSING | No border-radius on some components |
| `--shadow-sm` | ❌ MISSING | No shadow on cards |
| `--shadow-md` | ❌ MISSING | No elevated card shadow |
| `--transition-all` | ❌ MISSING | No transition on buttons |
| `--transition-colors` | ❌ MISSING | No transition on inputs |
| `--opacity-disabled` | ❌ MISSING | No disabled opacity on buttons/inputs |
| `--z-modal` | ❌ MISSING | No z-index on modals/tooltips |
| `--motion-duration-normal` | ❌ MISSING | No card transition |
| `--motion-easing-standard` | ❌ MISSING | No card transition easing |
| `--color-border-default` | ❌ MISSING | No border on inputs, secondary buttons, cards |
| `--color-border-subtle` | ❌ MISSING | No subtle border on outlined cards |
| `--color-action-primary-hover` | ❌ MISSING | No hover state on primary button |
| `--color-action-secondary-hover` | ❌ MISSING | No hover state on secondary button |
| `--color-action-ghost-hover` | ❌ MISSING | No hover state on ghost button |

**CRITICAL DUPLICATE**: `--color-text-primary` is generated **twice** in `generateSemanticTokens`:
- Line 1091: references `primaryKey` (brand primary color)
- Line 1112: references darkest neutral

The second always overwrites the first. Since Button primary uses `var(--color-text-primary)` for `background-color`, the button ends up with a dark neutral (near-black) background instead of the brand primary color. This makes the button look black.

---

## Bug 2 — Non-Semantic Token Values in Component Settings (Image 5)

### Symptom

In the Component Settings panel, after BCE apply, the token value fields show plain CSS values (`0`, `center`, `block`) instead of semantic references like `var(--space-2)` or `var(--radius-md)`.

### Root Cause

`getDefaultComponentProperties` correctly produces semantic references:
```js
props["gap"]           = { value: "var(--space-2)", ... };
props["border-radius"] = { value: "var(--radius-md)", ... };
```

But `--space-2` and `--radius-md` are never generated. When the Settings panel calls `resolveVariableValue("var(--space-2)")`, it gets back the unresolvable string. The Settings panel's value display parser then attempts numeric extraction from the token name string, yielding incorrect values like `2 px`.

**Root fix**: Generate all foundation tokens (Fix 1 below).

---

## Impact Summary

| Issue | Severity | User Impact |
|---|---|---|
| BCE wizard preview is a cosmetic fake | High | User approves content that looks branded in wizard but renders incorrectly in production |
| 21 foundation tokens missing after BCE apply | Critical | All component previews are unstyled/broken in Component Settings |
| `--color-text-primary` generated twice (duplicate) | Critical | Button primary renders with near-black background instead of brand color |
| Structural CSS props stored alongside token refs | Medium | Settings editor is cluttered with non-token properties |

---

## Design Decisions

| # | Question | Decision |
|---|---|---|
| Q1 | `--color-text-primary` semantic conflict | Split into `--color-text-primary` (darkest neutral = body text) and `--color-text-on-brand` (lightest neutral = text on brand surfaces). Component backgrounds use `--color-action-primary-bg`. |
| Q2 | Foundation token values: fixed vs. brand-derived | **Dynamically derived** — spacing and radius values scale with brand feel/mood personality. |
| Q3 | BCE apply: merge vs. overwrite | **Always overwrite** — BCE tokens always take precedence, even if the project has manually-set values. |
| Q4 | Structural CSS in component definition | **Keep them** — `display`, `align-items`, `flex-direction` etc. remain in component property definitions. |

### Q2 Detail — Dynamic Feel-Based Token Derivation

Radius and spacing values are derived from `feel.mood` and `feel.personality`:

| Brand Personality Keywords | `--radius-md` | `--radius-full` | `--space-4` (base) |
|---|---|---|---|
| `playful`, `friendly`, `rounded` | `12px` | `9999px` | `16px` |
| `corporate`, `professional`, `formal` | `4px` | `8px` | `16px` |
| `minimal`, `clean`, `sharp` | `2px` | `4px` | `16px` |
| `bold`, `editorial`, `expressive` | `8px` | `9999px` | `20px` |
| *(default / no match)* | `6px` | `9999px` | `16px` |

`generateFoundationTokens(feel)` must accept the `feel` object and apply these rules.

---

## Implementation Plan

### Fix 1 — Generate Complete Foundation Token Set (Backend)

**File**: `charisol-design-system-be/controllers/brandContextEngineController.js`

**Action**: Add `generateFoundationTokens(feel)` function and call it in `generateFromBrandContext`.

```js
function generateFoundationTokens(feel) {
  const tokens = [];
  const mood = Array.isArray(feel?.mood) ? feel.mood.map(m => m.toLowerCase()) : [];
  const personality = (feel?.personality || "").toLowerCase();
  const allFeelWords = [...mood, personality];

  // --- Derive radius and spacing from brand feel ---
  const isPlayful    = allFeelWords.some(w => ["playful","friendly","rounded","bubbly"].includes(w));
  const isCorporate  = allFeelWords.some(w => ["corporate","professional","formal","serious","business"].includes(w));
  const isMinimal    = allFeelWords.some(w => ["minimal","clean","sharp","geometric","stark"].includes(w));
  const isBold       = allFeelWords.some(w => ["bold","editorial","expressive","dynamic","dramatic"].includes(w));

  let radiusSm, radiusMd, radiusLg, radiusXl, radiusFull, baseSpace;
  if (isPlayful) {
    [radiusSm, radiusMd, radiusLg, radiusXl, radiusFull, baseSpace] = [8, 12, 16, 20, 9999, 16];
  } else if (isCorporate) {
    [radiusSm, radiusMd, radiusLg, radiusXl, radiusFull, baseSpace] = [2, 4, 6,  8,     8, 16];
  } else if (isMinimal) {
    [radiusSm, radiusMd, radiusLg, radiusXl, radiusFull, baseSpace] = [0, 2, 4,  6,     4, 16];
  } else if (isBold) {
    [radiusSm, radiusMd, radiusLg, radiusXl, radiusFull, baseSpace] = [4, 8, 12, 16, 9999, 20];
  } else {
    [radiusSm, radiusMd, radiusLg, radiusXl, radiusFull, baseSpace] = [4, 6, 8,  12, 9999, 16];
  }

  // --- Spacing scale (feel-adjusted base) ---
  const spacingScale = [
    ["--space-0", "0"],
    ["--space-1", "4"],
    ["--space-2", String(Math.round(baseSpace / 2))],
    ["--space-3", String(Math.round(baseSpace * 0.75))],
    ["--space-4", String(baseSpace)],
    ["--space-5", String(Math.round(baseSpace * 1.25))],
    ["--space-6", String(Math.round(baseSpace * 1.5))],
    ["--space-8", String(baseSpace * 2)],
    ["--space-10", String(baseSpace * 2.5)],
    ["--space-12", String(baseSpace * 3)],
    ["--space-16", String(baseSpace * 4)],
  ];
  spacingScale.forEach(([key, val]) =>
    tokens.push(makeGeneratedVar(key, key.replace("--", ""), val, "dimension", "px", `Spacing ${val}px`, "brand"))
  );

  // --- Border radius (feel-adjusted) ---
  const radiusScale = [
    ["--radius-none", "0"],
    ["--radius-sm",   String(radiusSm)],
    ["--radius-md",   String(radiusMd)],
    ["--radius-lg",   String(radiusLg)],
    ["--radius-xl",   String(radiusXl)],
    ["--radius-2xl",  String(Math.round(radiusXl * 1.33))],
    ["--radius-full", String(radiusFull)],
  ];
  radiusScale.forEach(([key, val]) =>
    tokens.push(makeGeneratedVar(key, key.replace("--", ""), val, "dimension", "px", `Radius ${val}px`, "brand"))
  );

  // --- Shadows ---
  tokens.push(makeGeneratedVar("--shadow-none", "Shadow None", "none", "shadow", null, "No shadow", "brand"));
  tokens.push(makeGeneratedVar("--shadow-sm",   "Shadow SM",   "0 1px 2px 0 rgba(0,0,0,0.05)",     "shadow", null, "Small shadow",  "brand"));
  tokens.push(makeGeneratedVar("--shadow-md",   "Shadow MD",   "0 4px 6px -1px rgba(0,0,0,0.10)",  "shadow", null, "Medium shadow", "brand"));
  tokens.push(makeGeneratedVar("--shadow-lg",   "Shadow LG",   "0 10px 15px -3px rgba(0,0,0,0.10)","shadow", null, "Large shadow",  "brand"));

  // --- Transitions ---
  tokens.push(makeGeneratedVar("--transition-all",    "Transition All",    "all 0.15s ease",                                   "string", null, "All properties 150ms",   "brand"));
  tokens.push(makeGeneratedVar("--transition-colors", "Transition Colors", "color,background-color,border-color 0.15s ease",   "string", null, "Color properties 150ms", "brand"));

  // --- Opacity ---
  tokens.push(makeGeneratedVar("--opacity-disabled", "Opacity Disabled", "0.4", "number", null, "Disabled state opacity", "brand"));

  // --- Z-Index ---
  tokens.push(makeGeneratedVar("--z-base",    "Z Base",    "0",   "number", null, "Base layer",    "brand"));
  tokens.push(makeGeneratedVar("--z-modal",   "Z Modal",   "50",  "number", null, "Modal layer",   "brand"));
  tokens.push(makeGeneratedVar("--z-tooltip", "Z Tooltip", "100", "number", null, "Tooltip layer", "brand"));

  // --- Motion ---
  tokens.push(makeGeneratedVar("--motion-duration-fast",   "Motion Fast",   "100", "duration", "ms", "Fast animation (100ms)",   "brand"));
  tokens.push(makeGeneratedVar("--motion-duration-normal", "Motion Normal", "200", "duration", "ms", "Normal animation (200ms)", "brand"));
  tokens.push(makeGeneratedVar("--motion-duration-slow",   "Motion Slow",   "300", "duration", "ms", "Slow animation (300ms)",   "brand"));
  tokens.push(makeGeneratedVar("--motion-easing-standard", "Motion Easing", "cubic-bezier(0.4, 0, 0.2, 1)", "string", null, "Standard easing", "brand"));

  return tokens;
}
```

**Updated `generateFromBrandContext` generation order**:
```js
// 1. Brand color tokens
brandTokens = generateBrandTokens(mergedConfig.colors);

// 2. Foundation tokens — feel-aware (NEW, must precede semantic + component gen)
foundationTokens = generateFoundationTokens(mergedConfig.feel || {});

// 3. Semantic tokens (with added border/hover tokens — see Fix 2)
semanticTokens = generateSemanticTokens(brandTokens, mergedConfig.feel || {});

// 4. Typography tokens
typographyTokens = generateTypographyTokens(mergedConfig.typography);
```

### Fix 2 — Fix Duplicate `--color-text-primary` and Add Missing Semantic Tokens (Backend)

**File**: `charisol-design-system-be/controllers/brandContextEngineController.js`

In `generateSemanticTokens` — replace the duplicate block and add missing tokens:

```js
// REMOVE the duplicate at line 1112 entirely.

// REPLACE lines 1091-1094 with two distinct tokens:
tokens.push(makeGeneratedVar(
  "--color-text-primary", "Text Primary",
  `var(${neutralKeys[neutralKeys.length - 1] || "--color-neutral-900"})`,
  "color", null, "Primary body text (darkest neutral)", "semantic"
));
tokens.push(makeGeneratedVar(
  "--color-text-on-brand", "Text On Brand",
  `var(${neutralKeys[0] || "--color-neutral-0"})`,
  "color", null, "Text on brand-colored backgrounds (lightest neutral)", "semantic"
));

// ADD missing border and hover tokens after the existing block:
const neutral200 = neutralKeys.find(k => k.includes("-200"))
  || neutralKeys[Math.floor(neutralKeys.length / 3)]
  || "--color-neutral-0";
const neutral100 = neutralKeys.find(k => k.includes("-100"))
  || neutralKeys[0]
  || "--color-neutral-0";

tokens.push(makeGeneratedVar("--color-border-default",         "Border Default",         `var(${neutral200})`,                            "color", null, "Default border color",            "semantic"));
tokens.push(makeGeneratedVar("--color-border-subtle",          "Border Subtle",          `var(${neutral100})`,                            "color", null, "Subtle border color",             "semantic"));
tokens.push(makeGeneratedVar("--color-action-primary-hover",   "Action Primary Hover",   `var(${primaryKey})`,                            "color", null, "Primary action hover background", "semantic"));
tokens.push(makeGeneratedVar("--color-action-secondary-hover", "Action Secondary Hover", `var(${neutralKeys[0] || "--color-neutral-0"})`, "color", null, "Secondary action hover bg",       "semantic"));
tokens.push(makeGeneratedVar("--color-action-ghost-hover",     "Action Ghost Hover",     `var(${neutralKeys[0] || "--color-neutral-0"})`, "color", null, "Ghost action hover background",   "semantic"));
```

### Fix 3 — Fix Button Primary Background Token Reference (Backend)

**File**: `charisol-design-system-be/controllers/brandContextEngineController.js`

In `getDefaultComponentProperties`, Button `primary` case (line 1332–1334):

```js
// BEFORE:
if (variantName === "primary") {
  props["background-color"] = { value: `var(${colorTextPrimary})`, type: "color", unit: null, comment: "" };
  props["color"]            = { value: `var(${colorBgCanvas})`,    type: "color", unit: null, comment: "" };

// AFTER:
if (variantName === "primary") {
  props["background-color"] = { value: "var(--color-action-primary-bg)", type: "color", unit: null, comment: "" };
  props["color"]            = { value: "var(--color-text-on-brand)",     type: "color", unit: null, comment: "" };
```

### Fix 4 — Replace Fake BCE Wizard Preview with Token-Aware Renderer (Frontend)

**File**: `charisol-design-system-fe/components/brand-context/GenerationPreview.tsx`

**Action**: Replace `renderLiveComponentPreview` with a function that resolves `variantDef.properties` through `cssVarsObj`.

```tsx
// New helper — recursive var() resolver
function resolveTokenValue(value: string, vars: Record<string, string>, depth = 0): string {
  if (depth > 10 || !value) return value;
  return value.replace(/var\((--[\w-]+)\)/g, (match, key) => {
    const resolved = vars[key];
    if (!resolved) return match;
    return resolveTokenValue(resolved, vars, depth + 1);
  });
}

// Replaces renderLiveComponentPreview
function renderVariantPreview(
  variantDef: ComponentNode,
  cssVarsObj: Record<string, string>
): React.CSSProperties {
  if (!variantDef?.properties) return {};
  const styles: React.CSSProperties = {};
  Object.entries(variantDef.properties).forEach(([cssProp, propData]) => {
    const camelProp = cssProp.replace(/-([a-z])/g, (_, l) => l.toUpperCase());
    const rawVal = typeof propData === "object" && propData !== null ? (propData as any).value : propData;
    const unit   = typeof propData === "object" && propData !== null ? (propData as any).unit || "" : "";
    const resolved = resolveTokenValue(String(rawVal || ""), cssVarsObj);
    if (resolved && resolved !== "null") {
      (styles as any)[camelProp] = `${resolved}${unit}`;
    }
  });
  return styles;
}
```

In the component render loop, replace calls to `renderLiveComponentPreview(type, vName, variantDef)` with:

```tsx
const inlineStyle = renderVariantPreview(variantDef, cssVarsObj);
const htmlTag = variantDef.linkedElement?.htmlTag || "button";
return React.createElement(
  htmlTag,
  { style: inlineStyle, key: vName },
  type === "Button" ? vName.charAt(0).toUpperCase() + vName.slice(1) : undefined
);
```

---

## Files to Modify

### Backend

| File | Change |
|---|---|
| `charisol-design-system-be/controllers/brandContextEngineController.js` | Add `generateFoundationTokens(feel)` with dynamic radius/spacing; update generation order in `generateFromBrandContext`; fix `--color-text-primary` duplicate; add `--color-text-on-brand` + border/hover semantic tokens; fix Button primary to use `--color-action-primary-bg` |

### Frontend

| File | Change |
|---|---|
| `charisol-design-system-fe/components/brand-context/GenerationPreview.tsx` | Replace `renderLiveComponentPreview` with token-aware `renderVariantPreview` using `cssVarsObj` |

---

## Verification Plan

### Automated Tests

```bash
# Backend
cd charisol-design-system-be
npm test -- --grep "brandContext|generateFoundation|generateSemantic"
npm test -- --grep "classifyToken|validateToken"
```

### Manual Verification Checklist

1. **BCE Wizard Step 5 Preview**:
   - Run BCE with a "playful" brand → Button primary shows brand primary color; border-radius is large/rounded
   - Run BCE with a "corporate" brand → border-radius is small/sharp
   - Wizard preview matches Component Settings preview after save

2. **After "Approve & Apply" — Tokens section**:
   - Brand layer → `--space-2`, `--space-4`, `--radius-md`, `--radius-full`, `--shadow-sm`, `--opacity-disabled` all present with feel-derived values
   - Semantic layer → `--color-border-default`, `--color-action-primary-hover`, `--color-text-on-brand` present
   - `--color-text-primary` appears **only once**

3. **After apply — Components section**:
   - `Button — primary` → colored button matching brand primary (not black)
   - `Button — outline` → visible border, transparent background
   - `Badge — default` → styled badge with background color
   - `Input — default` → visible border + correct border-radius

4. **Component Settings token editor**:
   - `background-color` on Button primary → displays `var(--color-action-primary-bg)`
   - `padding-left` → displays `var(--space-4)` with correct resolved value
   - `border-radius` → displays `var(--radius-md)` with correct resolved value

---

## Open Questions

**Q1 — Text Primary Semantic Intent**
`--color-text-primary` currently doubles as both "body text" and "interactive element foreground." The fix splits it into `--color-text-primary` (dark neutral for body text) and `--color-text-on-brand` (light for text on brand-colored surfaces). Does this align with the design token taxonomy used in this system?

**Q2 — Foundation Token Values: Fixed vs. Brand-Derived**
Spacing and radius tokens are currently proposed as a fixed 8pt grid (e.g., `--space-2 = 8px`, `--radius-md = 6px`). A "playful" brand might want larger radii; a "corporate" brand smaller ones. Should the BCE's feel/mood output drive spacing and radius values?

**Q3 — Merge vs. Replace on Apply**
The current `applyBrandContextToProject` merges new tokens into existing ones (overwrites by key). If a user has manually set `--space-4: 20px` and runs BCE, it will be overwritten to `16px`. Should BCE respect existing tokens via a `skipIfExists` option?

**Q4 — Structural CSS in Token Store**
Properties like `display: flex`, `align-items: center`, `flex-direction: column` are stored in component property definitions alongside token references. These are not design tokens — they are structural layout rules. Should they be stored separately or filtered out of the token editor display?
