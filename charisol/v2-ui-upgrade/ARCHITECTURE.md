# V2 UI Upgrade — Architecture Document

> **Date:** 2026-09-09  
> **Branch Context:** `new-create-project-flow` (PR #250) introduced the new UI patterns. This document analyses the delta between the old architecture and the new tree-based UI, and defines the target state.

---

## 1. Executive Summary

PR #250 introduced three major UI rebuilds:
1. **Token management** — from a flat filtered table to a hierarchical `TokenTree` with a detail side panel
2. **Component management** — from a sidebar component browser to an inline `ComponentsTree` with taxonomy-based categorization
3. **Project creation** — from a single-modal form to a multi-step `NewProjectFlow` wizard with `BuildFromScratchWizard`

The old architecture enforced a **3-tier token layer model** (Brand → Semantic → Scoped/Component) deeply throughout the UI. The new UI still references this model in data structures but the visual UI has moved toward a **role-based grouping** (Color by role, Typography by font-family vs. type-scale, everything else by category). The "semantic" and "scoped" concepts are becoming vestigial — adding complexity without user value.

This document proposes **deprecating the semantic/scoped layer enforcement** in the frontend, simplifying token management to a flat hierarchy grouped by CSS property category, and upgrading the three core pages to the new UI paradigm.

---

## 2. Current State Analysis

### 2.1 Token Layer Model (OLD — To Be Deprecated)

The current architecture defines three token layers:

| Layer | ID | Description | Where Enforced |
|---|---|---|---|
| **Brand** | `brand` / `foundation` | Raw primitive values (colors, font families) | `TokenUtils.getSubTab()`, `EditTokenModal`, `AddVariableModal`, `TokenImportModal`, `ComponentsSection` filter |
| **Semantic** | `semantic` | Purpose-mapped aliases referencing brand tokens | `validateComponentPropertyValue()`, `findMatchingSemanticTokens()`, `filteredTokens` in ComponentsSection |
| **Scoped** | `component` | Component-specific overrides | `getSubTab()` prefix matching, `TokenTree.LAYER_TIERS` rendering |

**Files enforcing the 3-tier model (24 files total):**

| File | Usage |
|---|---|
| `data/mock.types.ts` | `DesignToken.layer` type union includes `"semantic" \| "component"` |
| `util/token.utils.ts` | `getSubTab()` — classifies tokens into brand/semantic/component via prefix heuristics |
| `util/token.utils.ts` | `findMatchingSemanticTokens()` — filters by `layer === "semantic"` |
| `util/token.utils.ts` | `validateComponentPropertyValue()` — rejects literal values and brand-token references, enforces semantic-only rule |
| `components/sections/project/TokensSection.tsx` | `SubTab` type, `getSubTab()` re-export, `computeTokenGroupCounts()` with brand/semantic/component counts |
| `components/sections/project/TokenTree.tsx` | `LAYER_TIERS` array, `renderTieredCategory()` — renders brand/semantic/scoped sub-folders |
| `components/modals/project/EditTokenModal.tsx` | Layer selector UI, reclassify API call |
| `components/modals/project/AddVariableModal.tsx` | Layer selector for new tokens |
| `components/modals/import/TokenImportModal.tsx` | Auto-layer assignment on import |
| `components/sections/project/ComponentsSection.tsx` | `filteredTokens` only allows `layer !== "semantic"` → forces semantic-only token picker |
| `services/project.service.ts` | `reclassifyVariable()`, `reclassifyAllVariables()` API methods |
| `app/api/project/[id]/variables/reclassify-all/route.ts` | BFF route for bulk reclassify |
| `app/api/project/[id]/variables/[variableId]/reclassify/route.ts` | BFF route for single reclassify |
| Multiple test files | `util.token-utils.test.ts`, `util.tag-safety.test.ts`, etc. |

### 2.2 Token Tab (OLD vs. NEW)

**OLD (TokensSection.tsx inline rendering):**
- Flat table with columns: Name, Value, Type, Visual Preview
- Sub-tab pills: Brand / Semantic / Component (with counts)
- Group tabs: Color, Font Family, Font Size, etc. (13 groups in `VARIABLE_GROUPS`)
- Inline edit on double-click
- Desktop table + mobile cards layout

**NEW (TokenTree.tsx):**
- Hierarchical tree with collapsible folders
- Top categories: Color, Typography, Spacing, Sizing, Border, Shadow, Motion, Layout, Flexbox, Lists
- Color is sub-grouped by **role**: Primary, Secondary, Accent, Neutral
- Typography is sub-grouped by: Font Family, Type Scale
- Spacing uses the old layer tiers (Brand/Semantic/Scoped) — **this is a leftover**
- Detail side panel on token selection showing: Resolves To, Used By (reference chain)
- Global search with expand/collapse all
- Column headers: Token, Value, Preview

### 2.3 Component Tab (OLD vs. NEW)

**OLD (ComponentsSection.tsx + sidebar):**
- Sidebar listed components grouped by HTML tag type (Button, Input/Form, Container/Layout, etc.)
- Click a component → DesignEditor opens inline
- Token picker restricted to semantic-layer tokens only

**NEW (ComponentsTree.tsx):**
- Full-page hierarchical tree with 6 top-level categories:
  - Actions & Triggers (Buttons, Dropdowns, Tooltips)
  - Forms & Inputs (Text Inputs, Selection Controls, Advanced Selectors)
  - Layout & Containers (Accordions, Cards, Tabs, Modals, Fragments)
  - Data Display & Visualization (Grids, Badges, Charts, Images)
  - Navigation (Nav Bars, Breadcrumbs, Pagination, Links)
  - Other (Uncategorized)
- Each component row shows a live preview cell
- Click → opens existing DesignEditor

### 2.4 Create Project Flow (OLD vs. NEW)

**OLD (CreateProjectForm.tsx):**
- Single modal with name/description/color/visibility fields
- Direct API call to create project

**NEW (NewProjectFlow.tsx + BuildFromScratchWizard.tsx):**
- 2-step flow: Name → Setup
- Setup step offers: Start from Scratch, Import, or Skip
- BuildFromScratchWizard: 5-step wizard (Basics → Colors → Type → Logo → Voice → Review)
  - Industry + Vibe selection → suggests palette and type pairing
  - 4 preset palettes + custom colors
  - 4 type pairings + custom fonts (with live Google Fonts rendering)
  - Logo: generate wordmark or upload
  - Voice tags selection
  - Review summary before applying

---

## 3. Target Architecture

### 3.1 Token Layer Simplification

**Decision: Deprecate the 3-tier enforcement.**

Tokens will have an optional `layer` field for metadata/export purposes only. The UI will NOT:
- Filter token pickers by layer
- Show brand/semantic/scoped sub-folders in the tree
- Validate that component properties must reference "semantic" tokens
- Offer reclassify functionality

**New token grouping model (all UI):**

```
├── Color (grouped by role: Primary, Secondary, Accent, Neutral)
├── Typography
│   ├── Font Family
│   └── Type Scale (font-size, font-weight, line-height)
├── Spacing (padding, margin, gap)
├── Sizing (width, height, min/max)
├── Border (border, border-radius)
├── Shadow
├── Motion (transition, duration)
├── Layout (display, position, overflow)
└── Other
```

### 3.2 Data Model Changes

```typescript
// DesignToken.layer remains for backward compat and export
// but is no longer used for UI filtering/validation
type DesignToken = {
  name: string;
  value: string;
  type: TokenType;
  unit: string | null;
  comment: string;
  key?: string;
  category?: string;       // e.g. "color", "typography", "spacing"
  deprecated?: boolean;
  layer?: string;           // kept for export/metadata only — not enforced in UI

  // classification field — kept for backend but not surfaced in UI
  classification?: { ... };

  // reference tracking — useful for Resolves To / Used By panel
  references?: { ... };
};
```

### 3.3 Component Architecture

```
components/
├── sections/
│   ├── dashboard/
│   │   ├── Dashboard.tsx          — main dashboard layout
│   │   ├── NewProjectFlow.tsx     — multi-step project creation
│   │   ├── NoProject.tsx          — empty state
│   │   └── CreateProject.tsx      — legacy (to be replaced by NewProjectFlow)
│   └── project/
│       ├── TokensSection.tsx      — container: header + actions + delegates to TokenTree
│       ├── TokenTree.tsx          — tree view with detail panel (remove layer tiers)
│       ├── ComponentsSection.tsx  — container: topbar + delegates to ComponentsTree or DesignEditor
│       ├── ComponentsTree.tsx     — taxonomy tree with live previews
│       ├── ProjectTopBar.tsx      — workspace tabs + branch selector + share
│       ├── ProjectSidebar.tsx     — collapsible navigation
│       └── ProjectDetailsScreen.tsx — tab router
├── modals/
│   └── project/
│       ├── BuildFromScratchWizard.tsx  — 5-step brand setup
│       ├── CreateComponentModal.tsx    — new component creation
│       ├── EditTokenModal.tsx          — token editing (remove layer selector)
│       ├── AddVariableModal.tsx        — add token (remove layer selector)
│       └── CreateProjectForm.tsx       — legacy (to be removed)
├── layout/
│   └── QuickCreateMenu.tsx        — shared action launcher
└── ui/
    └── AvatarMenu.tsx             — profile + theme toggle
```

---

## 4. What Gets Removed

### 4.1 Dead Code & Features to Remove

| Item | File(s) | Reason |
|---|---|---|
| `SubTab` type & sub-tab pills | `TokensSection.tsx` | Replaced by tree hierarchy |
| `VARIABLE_GROUPS` + group tabs | `TokensSection.tsx` | Replaced by `TOP_CATEGORIES` in TokenTree |
| `computeTokenGroupCounts()` brand/semantic/component sub-counts | `TokensSection.tsx` | No longer grouped by layer |
| `getSubTab()` in `TokensSection.tsx` | `TokensSection.tsx` | Wrapper around `TokenUtils.getSubTab()` — both unused after layer removal |
| `LAYER_TIERS` + `renderTieredCategory()` | `TokenTree.tsx` | Remove layer sub-folders from tree |
| Layer selector UI in `EditTokenModal` | `EditTokenModal.tsx` | No layer UI |
| Layer selector UI in `AddVariableModal` | `AddVariableModal.tsx` | No layer UI |
| Reclassify button + modal in `TokensSection` | `TokensSection.tsx` | Reclassify feature removed |
| `handleBulkPreview` / `handleBulkApply` / bulk states | `TokensSection.tsx` | Reclassify feature removed |
| `reclassifyVariable()` / `reclassifyAllVariables()` | `project.service.ts` | API methods for reclassify |
| BFF routes for reclassify | `app/api/.../reclassify*/route.ts` | Backend proxy routes |
| `validateComponentPropertyValue()` | `token.utils.ts` | Enforced semantic-only rule |
| `findMatchingSemanticTokens()` | `token.utils.ts` | Used by validation |
| `findMatchingBrandToken()` | `token.utils.ts` | Used by validation |
| `filteredTokens` semantic-only filter | `ComponentsSection.tsx` | Remove layer filter from token picker |
| `semanticMode` / `brandTokenName` state | `ComponentsSection.tsx` | Removed along with layer logic |
| `DownstreamImpactPanel` references | `TokensSection.tsx` | Possibly keep for value-change tracking but decouple from layers |
| `CreateProjectForm.tsx` | `modals/project/` | Replaced by `NewProjectFlow.tsx` |
| `CreateProject.tsx` (old modal trigger) | `sections/dashboard/` | Replaced by `NewProjectFlow.tsx` flow |

### 4.2 Functions to Simplify in `token.utils.ts`

| Function | Action |
|---|---|
| `getSubTab()` | Remove entirely or convert to a simple metadata getter (not used for UI) |
| `findMatchingSemanticTokens()` | Remove (no longer needed for validation) |
| `findMatchingBrandToken()` | Remove (no longer needed for validation) |
| `validateComponentPropertyValue()` | Remove entirely — was enforcing semantic-only property rule |

---

## 5. Key Design Decisions

### 5.1 Token Picker in Components

**Old:** Only semantic tokens could be picked for component properties.  
**New:** All tokens can be picked. The token picker shows a flat searchable list, optionally grouped by category. No layer enforcement.

### 5.2 Token Tree Hierarchy

Color tokens are grouped by **role** (primary/secondary/accent/neutral) via name heuristics. All other tokens are grouped purely by **CSS category** (spacing, sizing, border, etc.) — no layer sub-folders.

### 5.3 Token Detail Panel

The "Resolves To" / "Used By" side panel from `TokenTree.tsx` is retained as-is. It provides genuine value by showing reference chains without requiring layer concepts.

### 5.4 Create Project Flow

`NewProjectFlow.tsx` fully replaces the old `CreateProjectForm.tsx` modal. The `BuildFromScratchWizard` stores brand setup decisions and creates the project with a primary color. Typography, logo, and voice data are currently captured but only `primaryColor` is used in the create API call — future work will wire these through.

---

## 6. Migration Risks

| Risk | Impact | Mitigation |
|---|---|---|
| Backend still expects `layer` field | Medium | Keep `layer` on `DesignToken` type; stop sending it from new-creation flows; let backend default |
| Reclassify API routes become orphaned | Low | Remove BFF routes; backend endpoints remain but are unreachable from frontend |
| Tests reference `getSubTab` and layer logic | Medium | Update/remove affected test cases in the same phase |
| Token import flows set `layer` | Low | Simplify `TokenImportModal` to not set explicit layers |
| Existing projects have `layer` data | None | Data stays in DB; frontend simply doesn't render or filter by it |
