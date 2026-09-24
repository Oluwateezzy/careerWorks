# Phase 1 — Re-analyse Endpoint Fix

> **Priority**: 🔴 High (Quick fix, unblocks user workflow)
> **Estimated Effort**: 0.5–1 day
> **Dependencies**: None

---

## Background

The "Auto-Classify All" button in the Tokens section triggers the `reclassify-all` endpoint to re-classify every design token into the correct tier (Brand / Semantic / Component). Users report this **"is not working with respect to current implementation."**

### Call Chain (traced)

```
[TokensSection.tsx]  "Auto-Classify All" button click
  ↓ handleBulkPreview() → projectService.reclassifyAllVariables(projectId, "preview", …)
  ↓
[project.service.ts]  fetch(`/api/project/${queryId}/variables/reclassify-all`, { method: "POST", … })
  ↓
[app/api/project/[id]/variables/reclassify-all/route.ts]  BFF proxy
  ↓ fetchWithTimeout(`${backendApiUrl}/projects/${queryId}/variables/reclassify-all`, …)
  ↓
[frontendContractController.js]  reclassifyAllVariables(req, res)
  ↓ classifyAllTokens(store.variables, { … })
  ↓ Returns { status: true, data: { mode, reclassifiedCount, changes, … } }
```

### Identified Failure Points

| ID | Issue | Severity |
|----|-------|----------|
| FP-1 | `project.service.ts:570` reads `data.data` — if backend wraps differently or returns `{}` on error, this silently returns `undefined` | **High** |
| FP-2 | BFF `route.ts:25` uses 15s timeout — bulk reclassify on 200+ tokens may timeout | **Medium** |
| FP-3 | `TokensSection.tsx:424` calls `notifySuccessFxn` with `res.reclassifiedCount` which may be `undefined` on silent failure | **Medium** |
| FP-4 | No error toast shown to user when `bulkError` is set — state updates but UI doesn't render it | **Medium** |
| FP-5 | After "auto" mode apply, `syncWithProjectData` re-fetches — if this throws, the success toast still shows | **Low** |

---

## Tasks

### RV-100 — Add error visibility to reclassify-all flow

**Files**: [`TokensSection.tsx`](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-fe/components/sections/project/TokensSection.tsx)

**Changes**:
- Add visible error banner/toast when `bulkError` is non-null (currently state is set but never rendered)
- Add loading spinner to "Auto-Classify All" button during `previewing` and `applying` states
- Handle `res.reclassifiedCount === 0` case with a "No changes needed" info toast instead of success
- Wrap `syncWithProjectData` in try/catch to prevent success toast on re-fetch failure

**Validation Criteria**:
- [ ] When reclassify fails, user sees a red error toast with the error message
- [ ] "Auto-Classify All" button shows spinner while processing
- [ ] When 0 tokens are reclassified, user sees "No changes needed" info message
- [ ] If `syncWithProjectData` throws, no success toast is shown

---

### RV-101 — Harden project.service.ts reclassify response parsing

**Files**: [`project.service.ts`](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-fe/services/project.service.ts#L536-L574)

**Changes**:
- Add response body logging on failure: `console.error("reclassify-all response:", response.status, data)`
- Guard `data.data` — if undefined, try `data` directly (backend returns `{ status: true, data: {...} }`)
- Throw with the actual backend error message instead of generic "Failed to bulk reclassify tokens"

**Validation Criteria**:
- [ ] When backend returns `{ status: true, data: { reclassifiedCount: 3, … } }`, service returns `data.data` correctly
- [ ] When backend returns `{ error: "..." }`, service throws with the backend error message
- [ ] When backend returns `{}` (404 fallback), service throws "Failed to bulk reclassify" rather than returning `undefined`
- [ ] Console logs include the full status code and body for debugging

---

### RV-102 — Increase BFF timeout and add debug logging

**Files**: [`app/api/project/[id]/variables/reclassify-all/route.ts`](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-fe/app/api/project/%5Bid%5D/variables/reclassify-all/route.ts)

**Changes**:
- Increase `fetchWithTimeout` from 15000ms to 30000ms (bulk reclassify on large projects takes longer)
- Add `console.error` with full response body when `!response.ok`
- Forward the backend's `status` field in success responses

**Validation Criteria**:
- [ ] BFF timeout is 30 seconds
- [ ] Error responses include the backend error message in the log
- [ ] Success response includes `status: true` from backend

---

### RV-103 — End-to-end manual verification

**Scope**: No code changes — testing only

**Steps**:
1. Open a project with 30+ tokens in the Token section
2. Click "Auto-Classify All" and verify preview mode works
3. Verify preview shows changed tokens with previous → new layer
4. Click "Apply" and verify tokens are updated
5. Reload the page and verify changes persisted
6. Test with a project that has 0 changes expected — verify info toast

**Validation Criteria**:
- [ ] Preview mode returns results within 10 seconds
- [ ] Apply mode persists results and page reload confirms
- [ ] Edge case: 0 changes shows appropriate message
- [ ] Edge case: network error shows error toast

---

### RV-104 — Fix build type error in DesignEditor.tsx (if still present)

**Files**: [`DesignEditor.tsx:1554`](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-fe/components/sections/settings/DesignEditor.tsx#L1554)

**Changes**:
- Original error: `Argument of type 'unknown' is not assignable to parameter of type 'string'`
- This was at line 1554 where `item.property` was passed to `isLayoutProperty()` without type narrowing
- **Status**: Appears to already be fixed (line 1554 now has proper type narrowing)
- Verify `yarn build` passes cleanly

**Validation Criteria**:
- [ ] `yarn build` completes with exit code 0
- [ ] No TypeScript errors in `DesignEditor.tsx`
