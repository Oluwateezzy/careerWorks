# AI Brief Merge Bug — Root Cause Analysis & Fix Plan

## Problem Statement

When a user generates design data via the Brand Context engine (AI Brief), the system:
1. ✅ Generates 164 variables + 13 components correctly
2. ✅ Creates a branch with those tokens visible in the UI
3. ✅ Merge preview shows "+177 ADDED" (frontend diff is correct)
4. ✅ Merge API returns `200 OK` with `"Branch merged successfully"`
5. ❌ **But the merged data on `main` contains only the pre-existing `--test` variable — all 177 generated tokens vanish**

Manual branch creation + token creation + merge works correctly.

---

## Root Cause: `baseDataJson === mergedDataJson`

The bug is in [designSystemController.js](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-be/controllers/designSystemController.js) lines **752–780**, inside the `importAiBrief` function's branch-creation path.

### What happens step by step

#### Step 1: Fetch base data (L694–722)

```js
let baseData = { variables: {}, components: {} };
// ... fetches from S3 or DynamoDB ...
```

This correctly fetches the current `main` data (e.g., just `{ variables: { "--test": {...} }, components: {} }`).

#### Step 2: Normalize base data into Maps (L725–741)

```js
const baseVarsMap = new Map();
// ... populates from baseData.variables ...
const baseCompsMap = new Map();
// ... populates from baseData.components ...
```

At this point `baseVarsMap` has 1 entry: `"--test"`.

#### Step 3: Merge AI-generated tokens INTO the base maps (L743–750)

```js
variablesToApply.forEach((v) => {
  const key = v.key || v.name;
  if (key) baseVarsMap.set(key, { ...baseVarsMap.get(key), ...v });
});

componentsToApply.forEach((c) => {
  if (c.name) baseCompsMap.set(c.name, { ...baseCompsMap.get(c.name), ...c });
});
```

Now `baseVarsMap` has 165 entries (1 existing + 164 generated).
Now `baseCompsMap` has 13 entries.

> ⚠️ **This is the mutation that causes the bug.** The generated tokens are merged INTO the `baseVarsMap` and `baseCompsMap` **before** the base snapshot is serialized.

#### Step 4: Serialize BOTH base AND branch from the SAME maps (L752–780)

```js
// BASE snapshot — should be the ORIGINAL main state
const baseDataJson = {
  variables: Object.fromEntries(
    Array.from(baseVarsMap.entries()).map(...)  // ← 165 entries!
  ),
  components: Object.fromEntries(
    Array.from(baseCompsMap.entries()).map(...)  // ← 13 entries!
  ),
};

// BRANCH snapshot — should be main + AI generated tokens
const mergedDataJson = {
  variables: Object.fromEntries(
    Array.from(baseVarsMap.entries()).map(...)  // ← SAME 165 entries!
  ),
  components: Object.fromEntries(
    Array.from(baseCompsMap.entries()).map(...)  // ← SAME 13 entries!
  ),
};
```

**Both `baseDataJson` and `mergedDataJson` are serialized from the exact same mutated maps, so they are identical.**

#### Step 5: Upload to S3 (L786–797)

```js
// Uploads base snapshot — contains all 165 vars + 13 comps
await uploadProjectData({ key: baseS3Key, json: baseDataJson });
// Uploads branch snapshot — contains all 165 vars + 13 comps (IDENTICAL)
await uploadProjectData({ key: branchS3Key, json: mergedDataJson });
```

Both files in S3 are byte-for-byte identical.

#### Step 6: Later, user clicks "Merge" → `mergeBranch()` in mergeController.js

The merge engine calls `threeWayMerge(baseData, oursData, theirsData)`:

- `baseData` = base snapshot from S3 → **165 vars + 13 comps**
- `oursData` = branch snapshot from S3 → **165 vars + 13 comps** (IDENTICAL to base!)
- `theirsData` = current main data → **1 var (`--test`)**

Inside `threeWayMerge`, for each of the 164 generated tokens:
- `hasBase = true` (key exists in base)
- `hasOurs = true` (key exists in branch)
- `hasTheirs = false` (key does NOT exist on current main)

This matches the logic at [threeWayMerge.js L71–92](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-be/utils/threeWayMerge.js#L71-L92):

```js
} else if (hasOurs && !hasTheirs) {
  if (!hasBase) {
    // Added in ours, absent in theirs → KEEP (correct for new additions)
    merged[key] = oursVal;
  } else {
    // Existed in base, deleted in theirs
    const oursModified = !isDeepEqual(oursVal, baseVal);
    if (oursModified) {
      // Modified in ours, deleted in theirs → CONFLICT
    } else {
      // Untouched in ours, deleted in theirs → DELETE  ← THIS PATH!
    }
  }
}
```

Since `oursVal === baseVal` (they're identical — same data) and `!hasTheirs` (main doesn't have these tokens), the merge engine concludes: **"These tokens existed at branch creation time, main deleted them, and the branch didn't modify them → honor the deletion."**

**Result:** All 164 generated tokens are silently dropped. Only `--test` survives because it exists in all three datasets.

---

## Why Manual Branch + Token Creation Works

When you manually create a branch, then manually add a token:

1. **Base snapshot** = main state at branch-creation time (e.g. `{ "--test": {...} }`)
2. **Branch snapshot** = main state + your manually added token (e.g. `{ "--test": {...}, "--new-color": {...} }`)

Now when merge runs:
- `--new-color`: `hasBase = false`, `hasOurs = true`, `hasTheirs = false` → **Added in ours** → KEPT ✅
- The base snapshot correctly reflects what `main` looked like **before** the branch diverged.

---

## The Fix

The fix is simple: serialize `baseDataJson` BEFORE mutating the maps with generated tokens.

### Current (broken) order:
```
1. Populate maps from base data
2. Merge generated tokens INTO the maps      ← base is now polluted
3. Serialize baseDataJson FROM the maps       ← wrong! contains generated tokens
4. Serialize mergedDataJson FROM the maps     ← identical to base
```

### Correct order:
```
1. Populate maps from base data
2. Serialize baseDataJson FROM the maps       ← correct! clean base state
3. Merge generated tokens INTO the maps
4. Serialize mergedDataJson FROM the maps     ← correct! base + generated
```

### Exact code change in [designSystemController.js](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-be/controllers/designSystemController.js)

Move lines **752–765** (the `baseDataJson` serialization) to **before** lines 743–750 (the merge-into-maps loop).

**Before (L724–780):**
```js
// 2. Normalize and merge
const baseVarsMap = new Map();
// ... populate from baseData ...
const baseCompsMap = new Map();
// ... populate from baseData ...

variablesToApply.forEach((v) => { ... baseVarsMap.set(...) });
componentsToApply.forEach((c) => { ... baseCompsMap.set(...) });

const baseDataJson = { /* from baseVarsMap */ };   // ← BUG: maps already mutated
const mergedDataJson = { /* from baseVarsMap */ };  // ← identical to baseDataJson
```

**After (fixed):**
```js
// 2. Normalize base data
const baseVarsMap = new Map();
// ... populate from baseData ...
const baseCompsMap = new Map();
// ... populate from baseData ...

// 3. Serialize base snapshot BEFORE merging generated tokens
const baseDataJson = { /* from baseVarsMap */ };  // ← clean base state ✅

// 4. NOW merge generated tokens into the maps
variablesToApply.forEach((v) => { ... baseVarsMap.set(...) });
componentsToApply.forEach((c) => { ... baseCompsMap.set(...) });

// 5. Serialize branch data with the generated tokens included
const mergedDataJson = { /* from baseVarsMap */ };  // ← base + generated ✅
```

### Also fix in [frontendContractController.js](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-be/controllers/frontendContractController.js)

The same pattern exists in the `frontendContractController.js` AI brief handler (around L1456). The same reordering fix must be applied there.

---

## Additional: Revert previous incorrect s3.js change

The `s3.js` `fetchProjectDataFromS3` function should remain a pure data fetcher. The normalization logic previously added to it was addressing the wrong layer — the generated tokens are never stored in `imports.aiBrief.generated.variables` as full objects (they're intentionally stripped at L836–839 to avoid DynamoDB bloat). The bug is purely a serialization-ordering issue in the branch creation code.

**Status:** Already reverted in this session.

---

## Verification Plan

1. Generate AI design system via Brand Context engine
2. Verify the created branch shows tokens in the UI
3. Click "Compare & Merge"
4. Confirm the merge preview shows correct diff ("+N ADDED")
5. Click "Confirm Merge"
6. Switch to `main` branch
7. Verify all generated tokens are now present on `main`
8. Verify the merge response `mergedData.variables` contains all generated tokens (not just pre-existing ones)

---

## Files to Modify

| File | Change |
|------|--------|
| [designSystemController.js](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-be/controllers/designSystemController.js) | Reorder base snapshot serialization before generated-token merge (L724–780) |
| [frontendContractController.js](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-be/controllers/frontendContractController.js) | Same reorder fix in the parallel AI brief handler |
| [s3.js](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-be/utils/s3.js) | Already reverted — no change needed |
