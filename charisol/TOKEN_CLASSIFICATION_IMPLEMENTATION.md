# Token Classification — Implementation Plan

> **Version:** 1.0  
> **Date:** 2026-07-29  
> **Architecture Reference:** [TOKEN_CLASSIFICATION_ARCHITECTURE.md](file:///Users/oluwatobiloba/Desktop/charisol/design_system/docsplan/TOKEN_CLASSIFICATION_ARCHITECTURE.md)  
> **Status:** 🟡 Pending Approval

---

## Implementation Overview

This plan is divided into **3 phases** with **14 tasks**. Each task has a unique ID, clear description, affected files, and validation criteria.

### Phase Summary

| Phase | Name | Tasks | Dependencies | Goal |
|-------|------|-------|-------------|------|
| Phase 1 | Backend Classification Engine | TC-001 → TC-006 | None | Backend can classify tokens and store layer metadata |
| Phase 2 | API Endpoints & Backfill | TC-007 → TC-010 | Phase 1 | All existing/new tokens get classified via API |
| Phase 3 | Frontend Cutover | TC-011 → TC-014 | Phase 2 | Frontend reads authoritative layer from backend |

---

## Phase 1: Backend Classification Engine

### TC-001 — Create Token Classification Engine Module

**Task ID:** `TC-001`  
**Priority:** 🔴 Critical  
**Estimated Effort:** 3-4 hours  
**Depends On:** None

#### Description

Create a new utility module `tokenClassifier.js` that implements the deterministic classification engine. This module must be a **pure function** with no database or network side effects — it takes token data in and returns classification metadata out.

The engine applies rules in strict priority order:

1. **EXPLICIT_LAYER** — User-provided layer override
2. **COMPONENT_PREFIX** — Token key starts with a recognized component name
3. **VARIABLE_REFERENCE** — Token value contains `var(--...)`
4. **LITERAL_VALUE** — Token value is a raw CSS literal (hex, px, rem, etc.)
5. **PURPOSE_KEYWORD** — Token key contains purpose-oriented naming (primary, error, etc.)
6. **SCALE_KEYWORD** — Token key contains scale-oriented naming (500, lg, bold, etc.)
7. **FALLBACK** — Default to `semantic` with `autoClassified: true`

#### Files to Create

| File | Purpose |
|------|---------|
| [tokenClassifier.js](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-be/utils/tokenClassifier.js) | `[NEW]` Classification engine — exports `classifyToken()`, `classifyAllTokens()`, `BUILT_IN_COMPONENT_PREFIXES` |

#### Functions to Implement

```javascript
/**
 * @param {string} key - Token key (e.g., "--color-blue-500")
 * @param {string} value - Token value (e.g., "#3B82F6")
 * @param {Object} options
 * @param {string|null} options.explicitLayer - User-provided layer override
 * @param {string[]} options.registeredComponents - Project's component names
 * @returns {{ layer: "brand"|"semantic"|"component", confidence: "explicit"|"high"|"medium"|"low", rule: string, autoClassified: boolean, componentRef?: string }}
 */
function classifyToken(key, value, options = {}) { ... }

/**
 * Classify all tokens in a project's variable set.
 * @param {Object|Array} variables - Token map or array
 * @param {Object} options
 * @param {boolean} options.overrideExplicit - Re-classify tokens with existing explicit layer
 * @param {string[]} options.registeredComponents - Project's component names
 * @returns {{ classified: Object, summary: { brand: number, semantic: number, component: number }, warnings: Array }}
 */
function classifyAllTokens(variables, options = {}) { ... }

/**
 * Helper functions
 */
function isLiteralValue(value) { ... }
function isVariableReference(value) { ... }
function hasPurposeKeyword(key) { ... }
function hasScaleKeyword(key) { ... }
function normalizeLayerName(layer) { ... }
```

#### Validation Criteria

| # | Criterion | How to Validate |
|---|-----------|----------------|
| 1 | `classifyToken("--color-blue-500", "#3B82F6")` returns `{ layer: "brand", rule: "LITERAL_VALUE" }` | Unit test |
| 2 | `classifyToken("--color-primary", "var(--color-blue-500)")` returns `{ layer: "semantic", rule: "VARIABLE_REFERENCE" }` | Unit test |
| 3 | `classifyToken("--button-primary-bg", "var(--color-primary)")` returns `{ layer: "component", rule: "COMPONENT_PREFIX" }` | Unit test |
| 4 | `classifyToken("--spacing-16", "1rem")` returns `{ layer: "brand", rule: "LITERAL_VALUE" }` | Unit test |
| 5 | `classifyToken("--unknown-token", "something")` returns `{ layer: "semantic", rule: "FALLBACK", autoClassified: true }` | Unit test |
| 6 | `classifyToken("--anything", "val", { explicitLayer: "brand" })` returns `{ layer: "brand", rule: "EXPLICIT_LAYER" }` | Unit test |
| 7 | Module has zero external dependencies (no DB, no AWS, no HTTP) | Code review |
| 8 | Handles edge cases: empty key, null value, undefined options, non-string inputs | Unit tests |
| 9 | `classifyAllTokens()` correctly processes both array format and object-map format variables | Unit test |
| 10 | `classifyAllTokens()` returns accurate `summary` counts matching the actual classification results | Unit test |

---

### TC-002 — Create Token Classification Validation Module

**Task ID:** `TC-002`  
**Priority:** 🔴 Critical  
**Estimated Effort:** 2-3 hours  
**Depends On:** TC-001

#### Description

Create a validation module that enforces the classification constraints (hard and soft). This module checks whether a token's layer assignment is structurally valid and returns errors or warnings.

#### Files to Create

| File | Purpose |
|------|---------|
| [tokenClassificationValidator.js](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-be/utils/tokenClassificationValidator.js) | `[NEW]` Validation rules for token layer assignments |

#### Validation Rules to Implement

| Rule Code | Layer | Severity | Condition | Message |
|-----------|-------|----------|-----------|---------|
| `BRAND_CANNOT_REFERENCE` | Brand | **Error** | Brand token value contains `var(--...)` | Brand tokens must store raw values, not references. Use `semantic` layer for alias tokens. |
| `SCOPED_MISSING_COMPONENT` | Scoped | **Error** | Token key doesn't start with a recognized component prefix | Scoped tokens must be prefixed with a component name (e.g., button-, card-). |
| `CIRCULAR_REFERENCE` | Any | **Error** | Token references itself directly or through a chain | Token creates a circular reference chain: {chain}. |
| `SEMANTIC_LITERAL_VALUE` | Semantic | **Warning** | Semantic token has a literal value (no `var(--...)` reference) | This semantic token has a literal value. Consider creating a brand token and referencing it. |
| `SCOPED_DIRECT_BRAND_REF` | Scoped | **Warning** | Scoped token references a brand token directly (skipping semantic) | Scoped token references a brand token directly. Consider using a semantic alias for better maintainability. |
| `LOW_CONFIDENCE_CLASSIFICATION` | Any | **Info** | Auto-classification confidence is "low" | This token was auto-classified with low confidence. Consider setting the layer explicitly. |

#### Validation Criteria

| # | Criterion | How to Validate |
|---|-----------|----------------|
| 1 | Brand token with `var(--...)` value returns hard error `BRAND_CANNOT_REFERENCE` | Unit test |
| 2 | Scoped token without component prefix returns hard error `SCOPED_MISSING_COMPONENT` | Unit test |
| 3 | Circular reference detection works for direct self-reference | Unit test |
| 4 | Circular reference detection works for transitive chains (A→B→C→A) | Unit test |
| 5 | Semantic token with literal value returns warning (not error) | Unit test |
| 6 | Valid tokens return empty errors array | Unit test |
| 7 | Validator works with `resolveValue()` from existing tokenResolver.js | Integration test |

---

### TC-003 — Integrate Classification into Variable Create (V2 API)

**Task ID:** `TC-003`  
**Priority:** 🔴 Critical  
**Estimated Effort:** 2-3 hours  
**Depends On:** TC-001, TC-002

#### Description

Modify the `createVariableV2()` function in [`frontendContractController.js`](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-be/controllers/frontendContractController.js#L751-L788) to call the classification engine when a new token is created. The endpoint should:

1. Accept an optional `layer` field in the request body
2. Call `classifyToken()` with the token data
3. Call `validateTokenClassification()` to check constraints
4. Return hard errors as 400 responses
5. Include soft warnings in the success response
6. Store the `layer` and `classification` metadata on the token

#### Files to Modify

| File | Change |
|------|--------|
| [frontendContractController.js](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-be/controllers/frontendContractController.js) | `[MODIFY]` `createVariableV2()` — add classification call |

#### Code Change Summary

```javascript
// In createVariableV2():
// AFTER creating the token object, BEFORE storing:

const { classifyToken } = require("../utils/tokenClassifier");
const { validateTokenClassification } = require("../utils/tokenClassificationValidator");

// Get project's registered component names for better classification
const componentNames = (store.components || []).map(c => c.name || c.componentId).filter(Boolean);

const classification = classifyToken(req.body.key, req.body.value, {
  explicitLayer: req.body.layer || null,
  registeredComponents: componentNames,
});

const validation = validateTokenClassification({
  key: req.body.key,
  value: req.body.value,
  layer: classification.layer,
  allVariables: store.variables,
});

if (validation.errors.length > 0) {
  return apiError(res, 400, "CLASSIFICATION_ERROR", validation.errors[0].message, {
    errors: validation.errors,
  });
}

// Add classification to the created token object
created.layer = classification.layer;
created.classification = {
  confidence: classification.confidence,
  rule: classification.rule,
  autoClassified: classification.autoClassified || false,
};
```

#### Validation Criteria

| # | Criterion | How to Validate |
|---|-----------|----------------|
| 1 | Creating a token with `value: "#3B82F6"` stores `layer: "brand"` | API test |
| 2 | Creating a token with `value: "var(--color-blue-500)"` stores `layer: "semantic"` | API test |
| 3 | Creating a token with `key: "--button-bg"` stores `layer: "component"` | API test |
| 4 | Creating a token with `layer: "brand"` in body uses explicit layer | API test |
| 5 | Creating a brand token with `var(--...)` value returns 400 error | API test |
| 6 | Response includes `warnings` array (even if empty) | API test |
| 7 | Existing token creation without `layer` still works (backward compatible) | API test |
| 8 | The stored token in DynamoDB/S3 contains `layer` and `classification` fields | DB inspection |

---

### TC-004 — Integrate Classification into Variable Update (V2 API)

**Task ID:** `TC-004`  
**Priority:** 🔴 Critical  
**Estimated Effort:** 1-2 hours  
**Depends On:** TC-001, TC-002, TC-003

#### Description

Modify the `updateVariableV2()` function in [`frontendContractController.js`](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-be/controllers/frontendContractController.js#L790-L821) to re-classify a token when its value or key changes. When a user changes a token's value from a literal to a `var()` reference, the layer should auto-update unless the user explicitly set it.

#### Files to Modify

| File | Change |
|------|--------|
| [frontendContractController.js](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-be/controllers/frontendContractController.js) | `[MODIFY]` `updateVariableV2()` — re-classify on value change |

#### Validation Criteria

| # | Criterion | How to Validate |
|---|-----------|----------------|
| 1 | Updating a token's value from `#3B82F6` to `var(--color-blue-500)` changes layer from `brand` to `semantic` | API test |
| 2 | Updating a token with explicit `layer` preserves the explicit layer even when value changes | API test |
| 3 | Re-classification does NOT fire when only `comment` or `name` changes | API test |
| 4 | Validation constraints still apply on update | API test |

---

### TC-005 — Integrate Classification into V1 API Endpoints

**Task ID:** `TC-005`  
**Priority:** 🟡 Medium  
**Estimated Effort:** 2-3 hours  
**Depends On:** TC-001, TC-002

#### Description

Apply the same classification logic to the V1 API endpoints in [`designSystemController.js`](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-be/controllers/designSystemController.js).

#### Files to Modify

| File | Change |
|------|--------|
| [designSystemController.js](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-be/controllers/designSystemController.js) | `[MODIFY]` `createVariable()`, `updateVariable()` — add classification |

#### Validation Criteria

| # | Criterion | How to Validate |
|---|-----------|----------------|
| 1 | V1 token creation stores `layer` and `classification` | API test |
| 2 | V1 token update re-classifies on value change | API test |
| 3 | V1 API remains backward compatible (no breaking changes to request/response shape) | API test |

---

### TC-006 — Integrate Classification into Import Pipelines

**Task ID:** `TC-006`  
**Priority:** 🔴 Critical  
**Estimated Effort:** 3-4 hours  
**Depends On:** TC-001

#### Description

Ensure tokens created through all import pipelines (AI Brief, Context Engine, Style Dictionary, Figma, Baseline Preset) are classified. This is critical because the majority of tokens enter the system through imports, not manual creation.

#### Files to Modify

| File | Function(s) | Change |
|------|-------------|--------|
| [frontendContractController.js](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-be/controllers/frontendContractController.js) | `importAiBriefV2()`, `generateDesignSystemEngine()` | Classify generated/imported tokens |
| [designSystemController.js](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-be/controllers/designSystemController.js) | `importStyleDictionary()`, `importAiBrief()` | Classify imported tokens |
| [projectController.js](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-be/controllers/projectController.js) | `syncProjectVersionToRemote()`, `patchProjectVersionToRemote()` | Classify on sync/patch |
| [aiBriefGenerator.js](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-be/utils/aiBriefGenerator.js) | `buildAiGeneratedStarter()` | Pre-classify generated tokens |
| [baselinePreset.js](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-be/constants/baselinePreset.js) | `buildBaselinePreset()` | Pre-classify baseline tokens |

#### Approach

For bulk operations (imports, AI generation), use `classifyAllTokens()` rather than calling `classifyToken()` per-token, for efficiency and to generate a summary.

#### Validation Criteria

| # | Criterion | How to Validate |
|---|-----------|----------------|
| 1 | Tokens from AI Brief import have `layer` and `classification` set | API test |
| 2 | Tokens from Style Dictionary import are classified | API test |
| 3 | Tokens from Context Engine generation are classified | API test |
| 4 | Tokens from baseline preset have correct pre-assigned layers | Unit test |
| 5 | Bulk classification summary is included in import response | API test |
| 6 | Import with `targetLayer: "brand"` forces all imported tokens to brand | API test |
| 7 | Performance: classifying 1000 tokens takes < 50ms | Performance test |

---

## Phase 2: API Endpoints & Backfill

### TC-007 — Add Reclassification Endpoint

**Task ID:** `TC-007`  
**Priority:** 🟡 Medium  
**Estimated Effort:** 2-3 hours  
**Depends On:** TC-001, TC-003

#### Description

Add a new endpoint to reclassify a single token's layer. This allows users to manually override the auto-classification. The endpoint should:

1. Validate the new layer against constraints
2. Store `reclassifiedAt` and `reclassifiedBy` metadata
3. Return the old and new layer for user confirmation

#### API Specification

```
PATCH /v2/projects/:projectId/variables/:variableId/reclassify
```

**Request Body:**
```json
{
  "layer": "brand" | "semantic" | "component"
}
```

**Response:**
```json
{
  "data": {
    "id": "var_...",
    "key": "--token-key",
    "previousLayer": "semantic",
    "layer": "brand",
    "classification": {
      "confidence": "explicit",
      "rule": "EXPLICIT_LAYER",
      "autoClassified": false,
      "reclassifiedAt": "2026-07-29T13:00:00Z",
      "reclassifiedBy": "user_abc"
    },
    "warnings": []
  }
}
```

#### Files to Modify

| File | Change |
|------|--------|
| [frontendContractController.js](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-be/controllers/frontendContractController.js) | `[ADD]` `reclassifyVariable()` function |
| [routes/v1/](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-be/routes/v1) | `[MODIFY]` Add route `PATCH /v2/projects/:projectId/variables/:variableId/reclassify` |

#### Validation Criteria

| # | Criterion | How to Validate |
|---|-----------|----------------|
| 1 | Reclassifying a token updates `layer` in storage | API test |
| 2 | Response includes `previousLayer` for confirmation | API test |
| 3 | Reclassification validates constraints (brand token can't have var() value) | API test |
| 4 | `reclassifiedAt` and `reclassifiedBy` are stored | DB inspection |
| 5 | Non-existent variable returns 404 | API test |
| 6 | Invalid `layer` value returns 400 | API test |

---

### TC-008 — Add Bulk Reclassification Endpoint

**Task ID:** `TC-008`  
**Priority:** 🟡 Medium  
**Estimated Effort:** 3-4 hours  
**Depends On:** TC-001, TC-007

#### Description

Add a bulk reclassification endpoint that runs the classification engine across ALL tokens in a project. Supports two modes:

- **`auto`** — Actually reclassifies tokens and saves results
- **`preview`** — Dry-run that returns what WOULD change without saving

This is the primary tool for migrating existing projects from the broken frontend classification to the correct backend classification.

#### API Specification

```
POST /v2/projects/:projectId/variables/reclassify-all
```

**Request Body:**
```json
{
  "mode": "auto" | "preview",
  "overrideExplicit": false
}
```

#### Files to Modify

| File | Change |
|------|--------|
| [frontendContractController.js](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-be/controllers/frontendContractController.js) | `[ADD]` `reclassifyAllVariables()` function |
| Routes | `[MODIFY]` Add route for the bulk endpoint |

#### Validation Criteria

| # | Criterion | How to Validate |
|---|-----------|----------------|
| 1 | `preview` mode returns changes without modifying storage | API test + DB check |
| 2 | `auto` mode reclassifies tokens and saves to storage | API test + DB check |
| 3 | Summary counts (brand/semantic/component) are accurate | API test |
| 4 | `overrideExplicit: false` skips tokens with `classification.confidence: "explicit"` | API test |
| 5 | `overrideExplicit: true` re-classifies ALL tokens including explicit ones | API test |
| 6 | Response includes `reclassified` count and `unchanged` count | API test |
| 7 | Large projects (1000+ tokens) complete within 5 seconds | Performance test |
| 8 | The operation is idempotent (running twice produces same result) | API test |

---

### TC-009 — Add Layer Filter to List Variables Endpoint

**Task ID:** `TC-009`  
**Priority:** 🟡 Medium  
**Estimated Effort:** 1-2 hours  
**Depends On:** TC-003

#### Description

Extend the `listVariablesV2()` endpoint to support filtering by `layer` and to return `layerCounts` in the response metadata. This allows the frontend to efficiently query "show me all brand tokens" without client-side filtering.

#### API Specification Change

```
GET /v2/projects/:projectId/variables?layer=brand&type=color&page=1&pageSize=50
```

**New query parameters:**
- `layer` — Filter by token layer: `brand`, `semantic`, `component`
- `confidence` — Filter by classification confidence: `explicit`, `high`, `medium`, `low`

**New response field:**
```json
{
  "data": [...],
  "pagination": { ... },
  "layerCounts": {
    "brand": 340,
    "semantic": 165,
    "component": 108,
    "total": 613
  }
}
```

#### Files to Modify

| File | Change |
|------|--------|
| [frontendContractController.js](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-be/controllers/frontendContractController.js) | `[MODIFY]` `listVariablesV2()` — add layer filter and counts |

#### Validation Criteria

| # | Criterion | How to Validate |
|---|-----------|----------------|
| 1 | `?layer=brand` returns only brand tokens | API test |
| 2 | `?layer=semantic` returns only semantic tokens | API test |
| 3 | `?layer=component` returns only scoped tokens | API test |
| 4 | Omitting `layer` returns all tokens (backward compatible) | API test |
| 5 | `layerCounts` in response shows correct counts for all three layers | API test |
| 6 | Layer filter works with pagination | API test |
| 7 | Layer filter works with `type` filter together | API test |

---

### TC-010 — Include Classification in Token Export Formats

**Task ID:** `TC-010`  
**Priority:** 🟢 Low  
**Estimated Effort:** 2-3 hours  
**Depends On:** TC-003

#### Description

Update the token export formatters in [`tokenFormatters.js`](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-be/utils/tokenFormatters.js) to include layer metadata and group tokens by layer in exports.

#### Files to Modify

| File | Change |
|------|--------|
| [tokenFormatters.js](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-be/utils/tokenFormatters.js) | `[MODIFY]` `buildCss()` — group variables by layer in CSS comments |
| [tokenFormatters.js](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-be/utils/tokenFormatters.js) | `[MODIFY]` `buildTokensJson()` — include `$extensions.layer` per DTCG spec |
| [tokenFormatters.js](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-be/utils/tokenFormatters.js) | `[MODIFY]` `buildScss()` — group SCSS variables by layer |
| [publishSnapshot.js](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-be/utils/publishSnapshot.js) | `[MODIFY]` Include `layer` in published snapshot metadata |

#### Export Format Examples

**CSS (grouped by layer):**
```css
:root {
  /* ── BRAND TOKENS ── */
  --color-blue-500: #3B82F6;
  --font-size-lg: 18px;
  
  /* ── SEMANTIC TOKENS ── */
  --color-primary: var(--color-blue-500);
  --text-error: var(--color-red-600);
  
  /* ── COMPONENT TOKENS ── */
  --button-primary-bg: var(--color-primary);
  --input-border-focus: var(--color-interactive);
}
```

**DTCG tokens.json:**
```json
{
  "color-blue-500": {
    "$value": "#3B82F6",
    "$type": "color",
    "$extensions": { "com.strata.layer": "brand" }
  }
}
```

#### Validation Criteria

| # | Criterion | How to Validate |
|---|-----------|----------------|
| 1 | CSS export groups tokens by layer with comments | Snapshot test |
| 2 | DTCG JSON export includes `$extensions.layer` | Snapshot test |
| 3 | SCSS export groups variables by layer | Snapshot test |
| 4 | Published CDN snapshot includes layer metadata | Manual verification |
| 5 | Exports without layer data still work (backward compatible) | Snapshot test |

---

## Phase 3: Frontend Cutover

### TC-011 — Simplify Frontend Token Classification

**Task ID:** `TC-011`  
**Priority:** 🔴 Critical  
**Estimated Effort:** 2-3 hours  
**Depends On:** TC-003, TC-008 (backfill must be available)

#### Description

Replace the complex frontend heuristic in [`token.utils.ts`](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-fe/util/token.utils.ts#L92-L124) with a simple property read. The backend is now the source of truth for classification.

#### Files to Modify

| File | Change |
|------|--------|
| [token.utils.ts](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-fe/util/token.utils.ts) | `[MODIFY]` `getSubTab()` — simplify to `return token?.layer ?? "semantic"` |
| [mock.types.ts](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-fe/data/mock.types.ts) | `[MODIFY]` Update `DesignToken` type to include `classification` field |
| [TokensSection.tsx](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-fe/components/sections/project/TokensSection.tsx) | `[MODIFY]` Update `getSubTab()` import usage |

#### Code Change

```typescript
// token.utils.ts — BEFORE (30+ lines of heuristic)
static getSubTab(key: string, token?: DesignToken): "brand" | "semantic" | "component" {
  if (token?.layer) { ... }
  if (token?.category) { ... }
  const lower = key.toLowerCase().replace(/^--/, "");
  // ... 30+ lines of prefix matching ...
  return "semantic";
}

// token.utils.ts — AFTER (1 line)
static getSubTab(key: string, token?: DesignToken): "brand" | "semantic" | "component" {
  return token?.layer ?? "semantic";
}
```

#### Validation Criteria

| # | Criterion | How to Validate |
|---|-----------|----------------|
| 1 | Tokens with `layer: "brand"` appear in Brand tab | Manual UI test |
| 2 | Tokens with `layer: "semantic"` appear in Semantic tab | Manual UI test |
| 3 | Tokens with `layer: "component"` appear in Scoped tab | Manual UI test |
| 4 | Tokens without `layer` field fall back to "semantic" | Manual UI test |
| 5 | Token counts per layer match backend `layerCounts` | Manual UI test |
| 6 | No regression in existing functionality (edit, delete, duplicate, search) | Manual UI test |

---

### TC-012 — Add Reclassification UI to Edit Token Modal

**Task ID:** `TC-012`  
**Priority:** 🟡 Medium  
**Estimated Effort:** 2-3 hours  
**Depends On:** TC-007, TC-011

#### Description

Update the [`EditTokenModal.tsx`](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-fe/components/modals/project/EditTokenModal.tsx) to allow users to manually reclassify a token's layer. When the user changes the layer:

1. Call the reclassification API endpoint
2. Show any validation warnings returned by the backend
3. Update the local state with the new layer

#### Files to Modify

| File | Change |
|------|--------|
| [EditTokenModal.tsx](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-fe/components/modals/project/EditTokenModal.tsx) | `[MODIFY]` Layer selector calls reclassify API instead of local state change |
| [project.service.ts](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-fe/services/project.service.ts) | `[MODIFY]` Add `reclassifyVariable()` method |

#### Validation Criteria

| # | Criterion | How to Validate |
|---|-----------|----------------|
| 1 | Layer selector shows current backend layer | Manual UI test |
| 2 | Changing layer calls reclassify API | Network inspector |
| 3 | Validation warnings display in modal | Manual UI test |
| 4 | Hard validation errors prevent reclassification | Manual UI test |
| 5 | Token moves to correct tab after reclassification | Manual UI test |

---

### TC-013 — Add Classification Badges to Token List

**Task ID:** `TC-013`  
**Priority:** 🟢 Low  
**Estimated Effort:** 1-2 hours  
**Depends On:** TC-011

#### Description

Add visual indicators to the token list UI showing the classification confidence. This helps users understand which tokens were auto-classified and might need review.

#### Badge Design

| Confidence | Badge | Color | Tooltip |
|-----------|-------|-------|---------|
| `explicit` | ✓ | Green | "Layer set manually" |
| `high` | ⚡ | Blue | "Auto-classified: {rule}" |
| `medium` | ~ | Amber | "Auto-classified: {rule} — review recommended" |
| `low` | ? | Red | "Low confidence — consider setting layer manually" |

#### Files to Modify

| File | Change |
|------|--------|
| [TokensSection.tsx](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-fe/components/sections/project/TokensSection.tsx) | `[MODIFY]` Add classification confidence badge to token rows |

#### Validation Criteria

| # | Criterion | How to Validate |
|---|-----------|----------------|
| 1 | Explicit tokens show green checkmark badge | Manual UI test |
| 2 | Auto-classified tokens show appropriate badge color | Manual UI test |
| 3 | Hovering badge shows tooltip with classification rule | Manual UI test |
| 4 | Tokens without classification metadata show no badge | Manual UI test |

---

### TC-014 — Add Bulk Reclassification UI

**Task ID:** `TC-014`  
**Priority:** 🟢 Low  
**Estimated Effort:** 2-3 hours  
**Depends On:** TC-008, TC-011

#### Description

Add a UI action (button in the Tokens section header) that triggers bulk reclassification. This allows project owners to migrate all their tokens to the correct classification in one action.

#### UX Flow

1. User clicks "Auto-Classify All" button
2. System calls preview API first → shows summary of what would change
3. User reviews and confirms
4. System calls auto API → applies changes
5. Token counts update across all tabs

#### Files to Modify

| File | Change |
|------|--------|
| [TokensSection.tsx](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-fe/components/sections/project/TokensSection.tsx) | `[MODIFY]` Add "Auto-Classify" button + confirmation dialog |
| [project.service.ts](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-fe/services/project.service.ts) | `[MODIFY]` Add `reclassifyAllVariables()` method |

#### Validation Criteria

| # | Criterion | How to Validate |
|---|-----------|----------------|
| 1 | Preview shows correct counts before applying | Manual UI test |
| 2 | Applying updates token counts in all tabs | Manual UI test |
| 3 | Button is only visible to project owners | Manual UI test |
| 4 | Confirmation dialog prevents accidental bulk changes | Manual UI test |
| 5 | Progress indicator during bulk operation | Manual UI test |

---

## Testing Strategy

### Unit Tests (Required for Phase 1)

| Test File | Coverage |
|-----------|----------|
| `tokenClassifier.test.js` | All classification rules, edge cases, bulk classification |
| `tokenClassificationValidator.test.js` | All validation rules, circular reference detection |

### API Tests (Required for Phase 1-2)

| Test | Coverage |
|------|----------|
| Create variable with auto-classification | TC-003 |
| Create variable with explicit layer | TC-003 |
| Update variable triggers reclassification | TC-004 |
| Reclassify endpoint | TC-007 |
| Bulk reclassify endpoint (preview + auto) | TC-008 |
| List variables with layer filter | TC-009 |

### Manual UI Tests (Required for Phase 3)

| Test | Coverage |
|------|----------|
| Token tabs show correct counts | TC-011 |
| Edit modal reclassification flow | TC-012 |
| Classification badges display correctly | TC-013 |
| Bulk reclassification flow | TC-014 |

---

## Risk Assessment

| Risk | Impact | Mitigation |
|------|--------|-----------|
| **Existing tokens lose classification during backfill** | High | Preview mode first; idempotent operations; rollback via S3 versioning |
| **Performance degradation on large token sets** | Medium | Bulk classification is O(n); avoid per-token DB writes during backfill |
| **Frontend/backend classification mismatch during rollout** | Medium | Phase 1 adds backend classification; frontend reads `token.layer` with fallback to heuristic |
| **Breaking change to API response format** | Low | `layer` and `classification` are additive fields; no existing fields removed |
| **Component prefix list too aggressive** | Medium | Only classify by prefix for REGISTERED components; `BUILT_IN_COMPONENT_PREFIXES` is conservative |
