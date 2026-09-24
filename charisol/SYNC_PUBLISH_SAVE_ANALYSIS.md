# Syncing, Publishing & Saving — Deep Analysis & Implementation Plan

> **Created**: 2026-07-30  
> **Status**: Draft — Pending Review  
> **Scope**: Frontend (Next.js BFF + React context) ↔ Backend (Express + DynamoDB + S3)

---

## Table of Contents

1. [Problem Statement](#1-problem-statement)
2. [Current Architecture Map](#2-current-architecture-map)
3. [Root Cause Analysis — "Merge Not Showing on Main"](#3-root-cause-analysis)
4. [Full Discrepancy Inventory](#4-full-discrepancy-inventory)
5. [Separation of Concerns — Current vs Proposed](#5-separation-of-concerns)
6. [Implementation Plan](#6-implementation-plan)
7. [Validation Criteria](#7-validation-criteria)

---

## 1. Problem Statement

**Primary Bug**: After merging a branch into `main`, the merged data is **not visible** when viewing the `main` branch in the UI.

**Secondary Issues** identified during analysis:
- Tangled responsibilities between `DesignSystemContext`, `projectService`, and BFF API routes
- Inconsistent data fetching paths for `main` vs non-main branches
- `hasUnpublishedChanges` detection relies on fragile hash comparison that can miss merge-injected changes
- `syncWithProjectData()` after merge re-fetches from an endpoint that may return stale cached data
- Patch queue (`queuePatch`) does not distinguish between main draft and branch S3 targets cleanly
- `uploadToServer` (POST) and `flushPending` (PATCH) use different endpoints but have overlapping intent

---

## 2. Current Architecture Map

### 2.1 Data Flow Layers

```
┌─────────────────────────────────────────────────────────────────┐
│  LAYER 1: UI Components                                        │
│  ProjectTopBar, BranchSelector, MergeConfirmDialog,            │
│  TokensSection, ComponentsSection                              │
│  ► Call context methods: publish(), mergeBranch(), updateVar()  │
└───────────┬─────────────────────────────────────────────────────┘
            │ useDesignSystem()
┌───────────▼─────────────────────────────────────────────────────┐
│  LAYER 2: DesignSystemContext (State Management)                │
│  ► useReducer for local state (variables, components, etc.)     │
│  ► syncWithProjectData(id, branch) — PULL from server           │
│  ► uploadToServer(id) — PUSH full state                         │
│  ► updateVariable/Component — dispatch + projectService.queue   │
│  ► publish() → projectService.publishProject()                  │
│  ► mergeBranch() → projectService.mergeBranch()                 │
│  ► localStorage cache layer for offline fallback                │
└───────────┬─────────────────────────────────────────────────────┘
            │
┌───────────▼─────────────────────────────────────────────────────┐
│  LAYER 3: ProjectService (API Client + Patch Queue)             │
│  ► Singleton class with patch queue (pendingOps map)            │
│  ► queuePatch() → batches ops → flushPending() after 10s       │
│  ► flushImmediate() → sends PATCH to BFF route                 │
│  ► publishProject() → flushImmediate + POST /publish            │
│  ► mergeBranch() → flushImmediate + POST /branches/:name/merge  │
│  ► Sequence number tracking (optimistic locking)                │
└───────────┬─────────────────────────────────────────────────────┘
            │ fetch()
┌───────────▼─────────────────────────────────────────────────────┐
│  LAYER 4: BFF (Next.js API Routes — /app/api/)                  │
│  ► Proxy layer: forwards to backend with cookie auth            │
│  ► /api/project/[id]/data — GET main draft/published data       │
│  ► /api/project/[id]/upload — PATCH (ops) / POST (full)         │
│  ► /api/project/[id]/publish — POST publish to CDN              │
│  ► /api/project/[id]/branches/[name] — GET/DELETE branch        │
│  ► /api/project/[id]/branches/[name]/merge — POST merge         │
│  ► /api/project/[id]/branches/[name]/upload — PATCH branch ops  │
│  ► /api/project/[id]/diff — GET draft vs published diff         │
└───────────┬─────────────────────────────────────────────────────┘
            │ fetch(backendApiUrl/v1/...)
┌───────────▼─────────────────────────────────────────────────────┐
│  LAYER 5: Backend (Express API)                                 │
│  ► projectController.patchProjectVersionToRemote — patch main   │
│  ► projectController.getProjectVersion — fetch main data        │
│  ► projectController.publishProjectVersion — publish to CDN     │
│  ► branchController.patchBranchData — patch branch S3           │
│  ► branchController.getBranchData — fetch branch S3             │
│  ► mergeController.mergeBranch — 3-way merge → write main draft │
│  ► mergeController.resolveConflicts — apply resolutions + merge │
└───────────┬─────────────────────────────────────────────────────┘
            │
┌───────────▼─────────────────────────────────────────────────────┐
│  LAYER 6: Storage                                               │
│  ► DynamoDB: Project (metadata, draftData, projectData, seq#)   │
│  ►          ProjectBranch (per-branch metadata, s3Key, hash)    │
│  ► S3: projects/{id}/draft.json (main working copy)             │
│  ►    projects/{id}/{version}.json (published snapshots)        │
│  ►    projects/{id}/branches/{name}.json (branch working copy)  │
│  ►    projects/{id}/branches/{name}.base.json (branch ancestor) │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 Key Data Concepts

| Concept | Storage | Description |
|---------|---------|-------------|
| **Draft** | `project.draftData.s3Key` → S3 | Working copy on `main`. Patch ops apply here. |
| **Published Version** | `project.projectData[N].s3Key` → S3 | Immutable snapshot at publish time. |
| **Branch Data** | `branch.s3Key` → S3 | Working copy on branch. |
| **Branch Base** | `branch.baseS3Key` → S3 | Snapshot of main at branch creation time. Used for 3-way merge. |
| **Sequence Number** | DynamoDB (`project.sequenceNumber`, `branch.sequenceNumber`) | Optimistic concurrency control counter. |

---

## 3. Root Cause Analysis

### 3.1 The Merge Flow (What Should Happen)

1. User clicks "Confirm Merge" → `MergeConfirmDialog.handleSubmit()`
2. Context calls `mergeBranch(message)` (DesignSystemContext L1328)
3. Context calls `projectService.mergeBranch(projectId, currentBranch, message)` (project.service.ts L1609)
4. ProjectService first calls `flushImmediate(projectId, branchName)` — flushes pending ops
5. ProjectService POSTs to `/api/project/{id}/branches/{name}/merge`
6. BFF proxies to backend `mergeController.mergeBranch()`
7. Backend performs 3-way merge: `threeWayMerge(base, ours, theirs)`
8. Backend uploads merged result to **main's draft S3 key** (`projects/{id}/draft.json`)
9. Backend updates `project.draftData.s3Key` + increments `sequenceNumber`
10. Backend marks branch as `status: "merged"`
11. Backend returns `{ success: true }`
12. **Frontend receives success**, then:
    - Dispatches `SET_CURRENT_BRANCH` → `"main"` 
    - Calls `syncWithProjectData(projectId, "main")`

### 3.2 Where It Breaks — The Discrepancy Chain

**BUG #1: `syncWithProjectData` reads from `/api/project/{id}/data` for main, NOT from the project hydration endpoint**

In `DesignSystemContext.tsx` L637-643:
```typescript
if (branchName === "main") {
  response = await fetch(`/api/project/${projectId}/data`, { ... });
}
```

This BFF route (`/api/project/[id]/data/route.ts`) proxies to `projectController.getProjectVersion()` which:
- Checks `project.draftData.s3Key` first (✅ correct — merge writes here)
- Falls back to `project.projectData[defaultVersion].s3Key`

**However**, the problem is in the **response shape parsing** in the context (L675-679):

```typescript
const projectPayload =
  serverData.data?.projectData?.data ||
  serverData.projectData?.data ||
  (serverData.data?.variables || serverData.data?.components ? serverData.data : null) ||
  serverData;
```

The `getProjectVersion` backend returns:
```json
{
  "status": true,
  "message": "...",
  "data": {
    "projectData": { "<version>": { "s3Key": "...", "hash": "..." } },
    "sequenceNumber": N,
    "versions": N,
    ...serialized project fields
  }
}
```

The **actual design system data** (variables, components) is embedded in the `data` field directly when fetched from S3, but the serializer only includes metadata fields in `.data`. The critical point: **`getProjectVersion` returns the raw S3 data under a `data` key containing `{ variables, components }` mixed with project metadata.**

**BUG #2: Stale sequence number after merge**

After merge, the backend increments `project.sequenceNumber`. But the frontend's `syncWithProjectData` stores the sequence number from `serverData.data.sequenceNumber` (L713-715). If the response parsing above fails to extract the right fields, the sequence number may also be stale, causing subsequent patches to fail with 409 conflicts.

**BUG #3: `hasUnpublishedChanges` detection is unreliable after merge**

Lines 717-729 compare `draftData.hash` vs `publishedVersion.hash`. After a merge, `draftData.hash` is updated by the backend but the frontend's response may not contain these fields in the expected shape, so `different` may be incorrectly set to `false`.

**BUG #4: No forced cache invalidation after merge**

The `syncWithProjectData` call after merge (L1340) may get a cached response from the BFF or the browser's HTTP cache. The fetch doesn't include any cache-busting mechanism (no `Cache-Control: no-cache`, no random query param, no timestamp).

**BUG #5: `clearPendingOps` queue key mismatch**

In `syncWithProjectData` (L621):
```typescript
projectService.clearPendingOps(projectId, branchName);
```

But `clearPendingOps` constructs the queue key as:
```typescript
const queueKey = branchName ? `${projectId}:${branchName}` : projectId;
```

When called with `branchName = "main"`, this generates key `"projectId:main"` but the main branch queues under just `"projectId"` (L132):
```typescript
const queueKey = branch === "main" ? projectId : `${projectId}:${branch}`;
```

This means **`clearPendingOps(projectId, "main")` clears the WRONG key** — it clears `projectId:main` instead of `projectId`, leaving stale main-branch pending ops potentially corrupting the next flush.

---

## 4. Full Discrepancy Inventory

### DISC-001: Response Shape Inconsistency — Main vs Branch Fetch
| Attribute | Value |
|-----------|-------|
| **ID** | DISC-001 |
| **Severity** | Critical |
| **Location** | `DesignSystemContext.tsx` L637-679 |
| **Problem** | `syncWithProjectData` uses different endpoints for main (`/data`) vs branch (`/branches/{name}`). The response shapes differ: main returns serialized project metadata wrapped around S3 data; branch returns raw S3 data directly. The "waterfall" parser at L675 attempts to accommodate both but is fragile and can fail on edge cases. |
| **Impact** | After merge, re-fetching main may parse the wrong nested object, displaying empty or stale data. |

### DISC-002: clearPendingOps Queue Key Mismatch
| Attribute | Value |
|-----------|-------|
| **ID** | DISC-002 |
| **Severity** | High |
| **Location** | `project.service.ts` L132 vs L146 |
| **Problem** | `queuePatch` uses key `projectId` for main but `clearPendingOps` constructs key `projectId:main` when branchName = "main" is passed. |
| **Impact** | Stale patch ops for main are never cleared when switching branches, potentially double-applying or corrupting data. |

### DISC-003: No Cache Invalidation on Post-Merge Sync
| Attribute | Value |
|-----------|-------|
| **ID** | DISC-003 |
| **Severity** | High |
| **Location** | `DesignSystemContext.tsx` L1340 |
| **Problem** | After successful merge, `syncWithProjectData(projectId, "main")` issues a plain GET without cache-busting headers. Browser or CDN may serve stale pre-merge data. |
| **Impact** | User sees pre-merge data on main immediately after merge succeeds. |

### DISC-004: hasUnpublishedChanges Fragile Detection
| Attribute | Value |
|-----------|-------|
| **ID** | DISC-004 |
| **Severity** | Medium |
| **Location** | `DesignSystemContext.tsx` L717-729 |
| **Problem** | Relies on response containing `draftData.hash` and `projectData[defaultVersion].hash` which may not be present in the `/data` endpoint response shape. |
| **Impact** | Publish button shows wrong state (greyed out when changes exist, or highlighted when nothing changed). |

### DISC-005: Dual Save Pathways (uploadToServer vs flushPending)
| Attribute | Value |
|-----------|-------|
| **ID** | DISC-005 |
| **Severity** | Medium |
| **Location** | `DesignSystemContext.tsx` L742-791 vs `project.service.ts` L221-262 |
| **Problem** | `uploadToServer()` sends a full POST with the entire design system state. `flushPending()` sends PATCH ops. Both write to the same endpoint. There's no coordination — calling uploadToServer while pending ops exist could overwrite queued changes with stale local state. |
| **Impact** | Data loss when user manually triggers sync while patch queue has pending ops. |

### DISC-006: localStorage Cache Not Branch-Aware
| Attribute | Value |
|-----------|-------|
| **ID** | DISC-006 |
| **Severity** | Medium |
| **Location** | `DesignSystemContext.tsx` L487, L526-541 |
| **Problem** | Cache key is `designSystem_{projectId}` — does not include branch name. Switching branches loads the cache from the wrong branch. |
| **Impact** | Brief flash of wrong branch data on switch; potential data corruption if the cached (wrong branch) data triggers a save. |

### DISC-007: Undo/Redo Bypasses Patch Queue
| Attribute | Value |
|-----------|-------|
| **ID** | DISC-007 |
| **Severity** | Low |
| **Location** | `DesignSystemContext.tsx` L1237-1287 |
| **Problem** | Undo/redo calls `projectService.updateProjectDesignSystem()` which does a full project PATCH (not the patch queue). This means: (a) pending queued ops are cleared first, potentially losing batched changes; (b) the full design system replaces the draft rather than applying a targeted undo. |
| **Impact** | Undoing a single variable change overwrites the entire draft with potentially outdated local state. |

### DISC-008: Publish Does Not Confirm Draft Freshness
| Attribute | Value |
|-----------|-------|
| **ID** | DISC-008 |
| **Severity** | Low |
| **Location** | `projectController.js` L1735-1814 |
| **Problem** | `publishProjectVersion` reads from `project.draftData.s3Key` and creates a new version from it. It does not verify that the draft hasn't changed since the user last viewed it (no sequence number check on publish). |
| **Impact** | Two concurrent users could publish different states. |

---

## 5. Separation of Concerns — Current vs Proposed

### 5.1 Current (Tangled) Architecture

```
DesignSystemContext.tsx (1425 lines)
├── State management (useReducer)
├── Data fetching (syncWithAPI, syncWithProjectData)
├── Data pushing (uploadToServer)
├── CRUD operations (updateVariable, updateComponent, etc.)
├── Branch operations (setCurrentBranch, mergeBranch, resolveConflicts)
├── Publishing (publish, getDiff)
├── History management (undo, redo)
├── localStorage caching
├── Online/offline detection
├── Linked element management
└── Component attribute operations

project.service.ts (1737 lines)
├── Project CRUD
├── Design system CRUD
├── Patch queue management
├── Branch management
├── Collaborator management
├── Brand asset management
├── Notifications
├── Rate limiting
├── Sync token management
└── Logo upload
```

### 5.2 Proposed Separation

```
hooks/
├── useDesignSystemState.ts     — Pure state: reducer + selectors
├── useDesignSystemSync.ts      — Pull data from server, cache management  
├── useDesignSystemMutations.ts — Push changes (variable/component CRUD)
├── useBranchOperations.ts      — Branch create, switch, merge, rebase
├── usePublish.ts               — Publish workflow + diff preview
└── useDesignSystemHistory.ts   — Undo/redo with proper patch integration

services/
├── project-api.service.ts      — HTTP client + project CRUD
├── design-data.service.ts      — Design data fetch/push (patch queue)
├── branch.service.ts           — Branch-specific API calls
├── publish.service.ts          — Publish + diff API calls
├── collaboration.service.ts    — Members, invitations, notifications
└── asset.service.ts            — Brand assets, logo, file uploads

context/
└── DesignSystemContext.tsx      — Thin orchestration layer composing hooks
```

---

## 6. Implementation Plan

### Phase 1: Fix the Critical Merge Bug (Immediate)

#### TASK-SP-001: Fix clearPendingOps Queue Key for Main Branch
| Attribute | Value |
|-----------|-------|
| **ID** | TASK-SP-001 |
| **File** | `services/project.service.ts` |
| **Lines** | 145-154 |
| **Change** | Update `clearPendingOps` to normalize the queue key the same way `queuePatch` does: use bare `projectId` when `branchName` is `"main"` or undefined. |
| **Validation** | Unit test: `clearPendingOps(id, "main")` clears the same key that `queuePatch(id, op)` writes to. |

```typescript
// BEFORE (L146):
const queueKey = branchName ? `${projectId}:${branchName}` : projectId;

// AFTER:
const branch = branchName || this.currentBranch;
const queueKey = (!branch || branch === "main") ? projectId : `${projectId}:${branch}`;
```

#### TASK-SP-002: Add Cache-Busting to Post-Merge Sync
| Attribute | Value |
|-----------|-------|
| **ID** | TASK-SP-002 |
| **File** | `context/DesignSystemContext.tsx` |
| **Lines** | 636-651 |
| **Change** | Append a cache-busting timestamp query parameter when fetching main data after merge. Add `Cache-Control: no-cache` header. |
| **Validation** | Network inspector shows unique URL per sync call; no 304 responses after merge. |

```typescript
// Add timestamp to bust cache
const cacheBuster = `_t=${Date.now()}`;
if (branchName === "main") {
  response = await fetch(`/api/project/${projectId}/data?${cacheBuster}`, {
    method: "GET",
    headers: {
      "Content-Type": "application/json",
      "Cache-Control": "no-cache",
    },
  });
}
```

#### TASK-SP-003: Normalize Response Shape Parsing for Main Data
| Attribute | Value |
|-----------|-------|
| **ID** | TASK-SP-003 |
| **File** | `context/DesignSystemContext.tsx` |
| **Lines** | 670-679 |
| **Change** | Create a dedicated `extractDesignSystemData(serverResponse)` function that handles all known backend response shapes. Log a warning if the fallback path is used. |
| **Validation** | Test with mock responses from all backend endpoints (getProjectVersion, getBranchData). All return correct `{ variables, components }` shape. |

```typescript
function extractDesignSystemData(serverData: any): {
  variables: Record<string, any>;
  components: Record<string, any>;
  custom?: any;
} {
  // Shape 1: getProjectVersion returns data at top level with project metadata mixed in
  // The S3 data is returned directly: { status, data: { variables, components, ... project fields } }
  // Shape 2: getBranchData returns { status, data: { variables, components, sequenceNumber } }
  // Shape 3: Nested projectData wrapper { data: { projectData: { data: { variables, components } } } }
  
  const d = serverData?.data;
  
  // Try nested projectData wrapper first
  if (d?.projectData?.data?.variables || d?.projectData?.data?.components) {
    return d.projectData.data;
  }
  
  // Try direct data with variables/components
  if (d?.variables !== undefined || d?.components !== undefined) {
    return { variables: d.variables || {}, components: d.components || {}, custom: d.custom };
  }
  
  // Bare response
  if (serverData?.variables !== undefined || serverData?.components !== undefined) {
    return { variables: serverData.variables || {}, components: serverData.components || {} };
  }
  
  console.warn("[syncWithProjectData] Could not extract design system data from response:", serverData);
  return { variables: {}, components: {} };
}
```

#### TASK-SP-004: Fix hasUnpublishedChanges After Merge
| Attribute | Value |
|-----------|-------|
| **ID** | TASK-SP-004 |
| **File** | `context/DesignSystemContext.tsx` |
| **Lines** | 717-729 |
| **Change** | After a merge, the backend increments the sequence number and updates `draftData`. The frontend should always set `hasUnpublishedChanges = true` after a merge since the draft now differs from the last published version. Move the detection to use the backend's version info from the `/data` response more reliably, or ask the backend for the diff hash. |
| **Validation** | After merge, the "Publish changes" button is correctly highlighted. |

---

### Phase 2: Fix Data Integrity Issues (Short-term)

#### TASK-SP-005: Make localStorage Cache Branch-Aware
| Attribute | Value |
|-----------|-------|
| **ID** | TASK-SP-005 |
| **File** | `context/DesignSystemContext.tsx` |
| **Lines** | 487, 506-541 |
| **Change** | Update `getStorageKey` to include branch name: `designSystem_{projectId}_{branchName}`. Clear cache on branch switch. |
| **Validation** | Switch branches 5x rapidly. Each switch shows correct branch data. No cross-contamination in localStorage. |

#### TASK-SP-006: Coordinate uploadToServer with Patch Queue
| Attribute | Value |
|-----------|-------|
| **ID** | TASK-SP-006 |
| **File** | `context/DesignSystemContext.tsx` |
| **Lines** | 742-791 |
| **Change** | Before `uploadToServer` sends a full POST, first `await projectService.flushImmediate(projectId)` to drain the patch queue. Log a warning if there were pending ops. Consider deprecating `uploadToServer` entirely in favour of always using the patch queue. |
| **Validation** | Manual test: make a change → immediately click sync → verify no data loss. |

#### TASK-SP-007: Fix Undo/Redo to Use Patch Queue
| Attribute | Value |
|-----------|-------|
| **ID** | TASK-SP-007 |
| **File** | `context/DesignSystemContext.tsx` |
| **Lines** | 1237-1287 |
| **Change** | Instead of calling `updateProjectDesignSystem()` (full replace), compute the diff between current and previous state and issue targeted patch ops. Or at minimum, flush pending ops first and then do the full replace. |
| **Validation** | Make 3 variable changes → undo → redo → verify all values match expectations. |

---

### Phase 3: Separation of Concerns Refactor (Medium-term)

#### TASK-SP-010: Extract State Management to useDesignSystemState
| Attribute | Value |
|-----------|-------|
| **ID** | TASK-SP-010 |
| **File** | `hooks/useDesignSystemState.ts` (NEW) |
| **Change** | Move the reducer, initial state, and action types out of `DesignSystemContext.tsx` into a standalone hook. Export selectors for derived state (canUndo, canRedo, hasChanges). |
| **Validation** | All existing tests pass. Context file reduced by ~400 lines. |

#### TASK-SP-011: Extract Sync Logic to useDesignSystemSync
| Attribute | Value |
|-----------|-------|
| **ID** | TASK-SP-011 |
| **File** | `hooks/useDesignSystemSync.ts` (NEW) |
| **Change** | Move `syncWithAPI`, `syncWithProjectData`, `uploadToServer`, online/offline detection, and localStorage caching into a dedicated sync hook. |
| **Validation** | Context file reduced by ~200 more lines. Sync behavior identical. |

#### TASK-SP-012: Extract Branch Operations to useBranchOperations
| Attribute | Value |
|-----------|-------|
| **ID** | TASK-SP-012 |
| **File** | `hooks/useBranchOperations.ts` (NEW) |
| **Change** | Move `setCurrentBranch`, `mergeBranch`, `resolveConflicts` into a dedicated branch hook. Include branch-aware cache clearing. |
| **Validation** | Branch create, switch, merge, delete all work. Merge shows data on main. |

#### TASK-SP-013: Split ProjectService Into Domain Services
| Attribute | Value |
|-----------|-------|
| **ID** | TASK-SP-013 |
| **Files** | `services/project-api.service.ts` (NEW), `services/design-data.service.ts` (NEW), `services/branch.service.ts` (NEW), `services/publish.service.ts` (NEW) |
| **Change** | Break the 1737-line `project.service.ts` into focused modules. `design-data.service.ts` owns the patch queue and data sync. `branch.service.ts` owns branch CRUD and merge. `publish.service.ts` owns publish and diff. Each service ≤ 400 lines. |
| **Validation** | All features work. Import paths updated. No circular dependencies. |

#### TASK-SP-014: Extract Mutation Logic to useDesignSystemMutations
| Attribute | Value |
|-----------|-------|
| **ID** | TASK-SP-014 |
| **File** | `hooks/useDesignSystemMutations.ts` (NEW) |
| **Change** | Move `updateVariable`, `updateComponent`, `deleteVariable`, `deleteComponent`, `linkElement`, `unlinkElement`, `updateComponentAttributes` into a mutations hook. |
| **Validation** | All CRUD operations work. Context file is < 200 lines (thin orchestration). |

---

### Phase 4: Backend Improvements (Medium-term)

#### TASK-SP-020: Add Sequence Number Check to Publish
| Attribute | Value |
|-----------|-------|
| **ID** | TASK-SP-020 |
| **File** | `controllers/projectController.js` (BE) |
| **Lines** | L1735-1814 |
| **Change** | Accept optional `sequenceNumber` in publish request body. If provided and < current project sequence number, return 409 with current sequence number. |
| **Validation** | Two simultaneous publish attempts: second one gets 409. |

#### TASK-SP-021: Return Merged Data in Merge Response
| Attribute | Value |
|-----------|-------|
| **ID** | TASK-SP-021 |
| **File** | `controllers/mergeController.js` (BE) |
| **Lines** | L241-248 |
| **Change** | Include the merged design system data in the merge response so the frontend can hydrate immediately without a second fetch. |
| **Validation** | Frontend can skip the `syncWithProjectData` call after merge and use the inline response data. Eliminates the race condition entirely. |

#### TASK-SP-022: Normalize Data Endpoint Response Shapes
| Attribute | Value |
|-----------|-------|
| **ID** | TASK-SP-022 |
| **Files** | `controllers/projectController.js` (BE), `controllers/branchController.js` (BE) |
| **Change** | Ensure `getProjectVersion` and `getBranchData` return an identical response shape: `{ status, data: { variables, components, custom, sequenceNumber, draftHash, publishedHash } }`. This eliminates the need for waterfall response parsing on the frontend. |
| **Validation** | Frontend `extractDesignSystemData()` handles both endpoints with the same code path. |

---

## 7. Validation Criteria

### 7.1 Critical Path — Merge to Main Visibility

| # | Test Scenario | Expected Result | Task IDs |
|---|--------------|-----------------|----------|
| V-001 | Create branch, add 3 variables, merge to main | All 3 variables visible on main immediately after merge | SP-001, SP-002, SP-003 |
| V-002 | Create branch, modify 2 existing variables, merge | Modified values reflected on main | SP-003 |
| V-003 | Create branch, delete 1 variable, merge | Variable removed from main | SP-003 |
| V-004 | Create branch, add component, merge | Component visible on main | SP-003 |
| V-005 | Merge, then click Publish | "Publish changes" button is highlighted; publish succeeds | SP-004 |
| V-006 | Merge, navigate away, return to project | Main shows merged data (not stale cache) | SP-002, SP-005 |

### 7.2 Data Integrity

| # | Test Scenario | Expected Result | Task IDs |
|---|--------------|-----------------|----------|
| V-010 | Edit variable on main → switch to branch → switch back to main | Main shows the edit, not branch data | SP-005 |
| V-011 | Make rapid edits (10 variables in 5 seconds) → wait for auto-save → refresh | All 10 changes persisted | SP-001, SP-006 |
| V-012 | Make change → immediately click Sync button | Change persisted, no duplicates | SP-006 |
| V-013 | Make 3 changes → Undo → Redo → Undo | Correct state at each step; server reflects final state | SP-007 |
| V-014 | Two browser tabs, edit same variable differently | Second save gets conflict toast; no silent data overwrite | SP-020 |

### 7.3 Separation of Concerns

| # | Validation | Target | Task IDs |
|---|-----------|--------|----------|
| V-020 | `DesignSystemContext.tsx` < 200 lines | Thin orchestration only | SP-010..SP-014 |
| V-021 | `project.service.ts` replaced by ≤5 files, each < 400 lines | Domain separation | SP-013 |
| V-022 | No circular imports between hooks or services | Clean dependency graph | SP-010..SP-014 |
| V-023 | All existing functionality preserved | Full regression pass | ALL |

---

## Appendix A: File Reference

| File | Layer | Lines | Role |
|------|-------|-------|------|
| `context/DesignSystemContext.tsx` | State/Sync | 1425 | State management + sync orchestration |
| `services/project.service.ts` | API Client | 1737 | All API calls + patch queue |
| `app/api/project/[id]/data/route.ts` | BFF | 79 | Fetch main data |
| `app/api/project/[id]/upload/route.ts` | BFF | 140 | Upload/patch main data |
| `app/api/project/[id]/publish/route.ts` | BFF | 71 | Publish to CDN |
| `app/api/project/[id]/branches/route.ts` | BFF | 90 | List/create branches |
| `app/api/project/[id]/branches/[branchName]/route.ts` | BFF | 90 | Get/delete branch |
| `app/api/project/[id]/branches/[branchName]/merge/route.ts` | BFF | 53 | Merge branch |
| `app/api/project/[id]/branches/[branchName]/merge/resolve/route.ts` | BFF | 49 | Resolve conflicts |
| `app/api/project/[id]/diff/route.ts` | BFF | 68 | Draft vs published diff |
| `controllers/projectController.js` | Backend | 1916 | Project CRUD + data + publish |
| `controllers/branchController.js` | Backend | 520 | Branch CRUD + patch |
| `controllers/mergeController.js` | Backend | 539 | 3-way merge + conflict resolution |
| `lib/normalize-sync-response.ts` | Util | 64 | Response shape normalization |
| `lib/api-config.ts` | Config | 33 | Backend URL resolution |
| `components/sections/project/ProjectTopBar.tsx` | UI | 324 | Sync/publish/merge buttons |
| `components/collaboration/BranchSelector.tsx` | UI | 192 | Branch picker dropdown |
| `components/collaboration/MergeConfirmDialog.tsx` | UI | 193 | Merge confirmation modal |
| `components/collaboration/ConflictResolutionPanel.tsx` | UI | ~350 | Conflict resolution UI |

---

## Appendix B: Priority Matrix

| Priority | Tasks | Effort | Impact |
|----------|-------|--------|--------|
| 🔴 P0 — Ship this week | SP-001, SP-002, SP-003, SP-004 | 1-2 days | Fixes the merge bug |
| 🟡 P1 — Next sprint | SP-005, SP-006, SP-007 | 2-3 days | Prevents data integrity issues |
| 🟢 P2 — Next quarter | SP-010 → SP-014 | 1-2 weeks | Clean architecture |
| 🔵 P3 — Backlog | SP-020, SP-021, SP-022 | 1 week | Backend hardening |
