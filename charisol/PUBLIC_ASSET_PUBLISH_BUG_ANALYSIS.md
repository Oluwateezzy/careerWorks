# Public Assets & `strata.json` Synchronization Fix Plan

## Executive Summary & Problem Description

Users reported two critical issues with public asset generation (`strata.json`, `variables.css`, `tokens.json`, etc.) at `http://localhost:3000/api/public/projects/:projectId/assets/strata.json`:

1. **Scenario 1 (Branch Merge):** Merging AI-generated (or manual) branch data into `main` succeeds and says "successful". However, publishing or viewing `strata.json` shows empty/stale design system data.
2. **Scenario 2 (Token Updates on Main):** Creating a project and publishing reflects tokens. But updating tokens on `main` via V2 REST APIs (`updateVariableV2`, `patchComponentNode`, etc.) does NOT update `strata.json`, even after clicking publish.

---

## Technical Root Cause Analysis

Tracing the end-to-end data flow between DynamoDB, S3 draft storage, and CDN asset publishing revealed **three major gaps in state synchronization**:

```
 ┌────────────────────────┐      ┌─────────────────────────┐
 │   REST V2 API / Merge  │      │     Draft S3 Object     │
 │ (updateVariable, etc.) │      │ (projects/:id/draft.json│
 └───────────┬────────────┘      └────────────┬────────────┘
             │                                │
             ▼                                ▼
┌─────────────────────────┐      ┌─────────────────────────┐
│ DynamoDB designSystem   │      │ publishProjectVersion   │
│ (Legacy V1 Array store) │      │ (Reads ONLY from S3)    │
└────────────┬────────────┘      └────────────┬────────────┘
             │                                │
             ▼                                ▼
┌─────────────────────────┐      ┌─────────────────────────┐
│  publishProjectSnapshot │      │  CDN Asset strata.json  │
│  (Stale/Out-of-Sync)    │      │  (Empty or Old Tokens)  │
└─────────────────────────┘      └─────────────────────────┘
```

### 1. `mergeBranch` and `resolveConflicts` Never Trigger Asset Snapshot Publishing
- **File:** [mergeController.js](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-be/controllers/mergeController.js#L132)
- **Issue:** When a branch is merged, `mergeBranch` writes the merged `variables` and `components` to `project.draftData.s3Key` and updates `project.sequenceNumber` in DynamoDB.
- **Root Cause:** Neither `mergeBranch` nor `resolveConflicts` invokes `publishProjectSnapshot()` or updates `project.designSystem` in DynamoDB.
- **Impact:** The main draft in S3 is updated, but CDN public assets (`strata.json`, `variables.css`) are not re-published upon merge, leaving public assets pointing to the pre-merge snapshot.

### 2. V2 Token Edits Update DynamoDB `designSystem` but Skip `project.draftData` in S3
- **File:** [frontendContractController.js](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-be/controllers/frontendContractController.js#L234)
- **Issue:** V2 REST API endpoints (`updateVariableV2`, `createVariableV2`, `deleteVariableV2`, `updateComponentV2`, etc.) call `saveProjectDesignSystem(project, store)`.
- **Root Cause:** `saveProjectDesignSystem` only updates `project.designSystem` (DynamoDB V1 store). It **does not update `project.draftData.s3Key` in S3**.
- **Impact:** 
  1. `publishProjectSnapshot` is called with `store` (from DynamoDB), which re-publishes CDN assets immediately.
  2. **However**, when the user subsequently clicks the **"Publish" button** on the frontend (`POST /project/:queryId/publish`), `publishProjectVersion` in [projectController.js](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-be/controllers/projectController.js#L1717) fetches project data from `project.draftData.s3Key`.
  3. Since `project.draftData.s3Key` was never updated during V2 token edits, `publishProjectVersion` overwrites `strata.json` on S3/CDN with the **old, un-updated draft data from S3**!

### 3. Dual Storage Architecture Disconnect (DynamoDB `designSystem` vs S3 `draftData`)
- **Issue:** The codebase maintains two parallel sources of truth for project tokens:
  - `project.designSystem` (DynamoDB JSON array: `{ variables: [], components: [] }`)
  - `project.draftData.s3Key` (S3 object map: `{ variables: { "--primary": {} }, components: {} }`)
- **Root Cause:** Some controllers (e.g. `frontendContractController`) operate strictly on DynamoDB `designSystem`, while others (`projectController`, `mergeController`) operate on S3 `draftData`.
- **Impact:** Any operation that updates one store without syncing the other causes data drift. Publishing from S3 wipes out edits made in DynamoDB, and vice-versa.

---

## User Review Required

> [!IMPORTANT]
> **Single Source of Truth Strategy**: `saveProjectDesignSystem` must synchronize BOTH `project.designSystem` in DynamoDB AND `project.draftData.s3Key` in S3 whenever collaboration/V2 S3 draft data exists for a project.

---

## Proposed Changes

### Component 1: Centralized Store & S3 Draft Synchronization

#### [MODIFY] [frontendContractController.js](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-be/controllers/frontendContractController.js)
Update `saveProjectDesignSystem` to:
1. Update `project.designSystem` in DynamoDB.
2. If `project.draftData?.s3Key` exists (or project has S3 storage enabled), format and upload `store` to `project.draftData.s3Key` in S3 as well.
3. Update `draftData.hash` and `draftData.updatedAt` on the project record.
4. Trigger `publishProjectSnapshot(projectId, store, updatedProject)`.

#### [MODIFY] [designSystemController.js](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-be/controllers/designSystemController.js)
Align `saveProjectDesignSystem` to use the same dual-sync logic as `frontendContractController.js`.

---

### Component 2: Branch Merge Asset Publishing

#### [MODIFY] [mergeController.js](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-be/controllers/mergeController.js)
In `mergeBranch` and `resolveConflicts`:
1. When `merged` data is written to `draftS3Key` in S3:
2. Convert `merged` object structure to array `store` format `{ variables: Object.values(merged.variables), components: Object.values(merged.components), imports: ... }`.
3. Update `project.designSystem` in DynamoDB alongside `project.draftData`.
4. Call `publishProjectSnapshot(projectId, merged, updatedProject)` so CDN assets (`strata.json`, `variables.css`, `tokens.json`, etc.) immediately reflect the merged branch data.

---

### Component 3: Draft Data Normalization in `publishProjectVersion`

#### [MODIFY] [projectController.js](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-be/controllers/projectController.js)
In `publishProjectVersion`:
1. Check if `project.draftData.s3Key` exists in S3. If not, fallback to `project.designSystem` from DynamoDB.
2. When publishing, update `project.designSystem` in DynamoDB to stay identical with the published snapshot data.

---

## Verification Plan

### Automated Tests
- Run existing test suite for public projects and snapshot endpoints:
  ```bash
  npm test tests/integration/public-projects.test.js
  npm test tests/integration/snapshots.test.js
  ```

### Manual Verification Flow
1. **Test Token Update on Main:**
   - Update a token on `main` via V2 REST API (e.g. change color of `--primary`).
   - Verify `GET /api/public/projects/:projectId/assets/strata.json` returns updated value.
   - Click "Publish" (`POST /project/:queryId/publish`).
   - Verify `strata.json` still has the updated token value (not overwritten by old draft).

2. **Test AI Brief / Branch Merge:**
   - Generate AI design system (creates branch `ai-brief-xxx`).
   - Click "Merge Branch".
   - Verify `GET /api/public/projects/:projectId/assets/strata.json` immediately returns all merged tokens.
   - Click "Publish".
   - Verify `strata.json` contains all merged tokens and new version is reflected.
