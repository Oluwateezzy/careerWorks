# Phase 1 Manual Testing & Verification Guide

This guide provides end-to-end instructions for testing each component of **Phase 1: Screen Package Builder & Per-Screen LLM Generation**.

---

## Overview of Phase 1 Deliverables

| Task | Scope | Primary Files | Verification Command |
|---|---|---|---|
| **1.1** | Screen Package Builder | `charisol-design-system-be/workers/generate/screenPackage.js` | `node --test tests/unit/screen-package.test.js` |
| **1.2** | Screen Generator Service | `charisol-design-system-be/workers/generate/screenGenerator.js` | `node --test tests/unit/screen-generator.test.js` |
| **1.3** | Code Validator | `charisol-design-system-be/workers/generate/validator.js` | `node --test tests/unit/code-validator.test.js` |
| **1.4** | Per-Screen Progress Tracking | `charisol-design-system-be/services/prototypeJobService.js`<br>`charisol-design-system-be/controllers/prototypeController.js`<br>`charisol-design-system-be/routes/v1/index.js` | `node --test tests/unit/screen-progress.test.js` |

---

## Task 1.1: Screen Package Builder (`screenPackage.js`)

### Objective
Verify that `buildScreenPackage({ screenId, snapshot, projectId })` gathers all visual metadata for an individual screen, prunes massive trees to prevent token blowout (< 5,000 tokens), associates image assets with presigned URLs, and extracts screen-specific interactions.

### Automated Checks
```bash
cd charisol-design-system-be
node --test tests/unit/screen-package.test.js
```

### Manual Verification Steps
1. **Subtree Pruning Test**:
   - Provide a synthetic or real snapshot containing 500+ deeply nested nodes.
   - Run `pruneNodeTree(rootId, nodesById, { maxDepth: 8, maxChildren: 20, maxRepeated: 3 })`.
   - Verify that:
     - All invisible nodes (`visible === false`) are stripped.
     - Redundant repeated sibling rows are collapsed into summary indicators (`_collapsed: true`).
     - Vector path curves and huge SVG path strings are removed.
     - Floats in coordinates and color numbers are rounded to 2 decimal places.
     - Total node count drops from 500+ to under 200 nodes.
2. **Asset Resolution**:
   - Verify that all `imageRef` hashes within this screen's subtree map to presigned S3 URLs from `snapshot.imageAssetMap`.
3. **Scoped Interactions**:
   - Verify that clicks/hovers originating on nodes inside this frame are included in `interactions`.
   - Interactions originating on other screens are excluded.
4. **Token Budget Verification**:
   - Check that `JSON.stringify(screenPackage.nodeTree)` remains comfortably under 5,000 tokens.

---

## Task 1.2: Screen Generator Service (`screenGenerator.js`)

### Objective
Verify that `generateScreen({ screenPackage, modelId, client })` renders prompts, calls Amazon Bedrock Claude with the screen package and screenshot image, extracts clean TSX and CSS blocks, validates code, and retries once if output fails validation.

### Automated Checks
```bash
cd charisol-design-system-be
node --test tests/unit/screen-generator.test.js
```

### Manual Verification Steps
1. **Mock Generation Flow**:
   - Pass a mock Bedrock client returning valid markers:
     ```tsx
     ---TSX---
     import React from 'react';
     import styles from './Hero.module.css';
     export default function Hero() { return <div className={styles.root}>Hero</div>; }
     ---CSS---
     .root { display: flex; }
     ---END---
     ```
   - Verify that `generateScreen` returns `{ componentName: "Hero", tsx: "...", css: "...", metrics: { ... } }`.
2. **Auto-Retry on Malformed Output**:
   - Simulate a model returning malformed TSX on the first attempt (missing export or CSS marker).
   - Verify that `generateScreen` automatically re-invokes Bedrock with an error feedback prompt and succeeds on the second attempt.
3. **Live Bedrock Invocation (with active AWS credentials)**:
   - Call `generateScreen` with a sample screen package.
   - Confirm receipt of high-quality TypeScript code importing CSS Modules and wiring interactions via `useNavigate()`.

---

## Task 1.3: Code Validator (`validator.js`)

### Objective
Verify that generated TSX and CSS are quickly validated for syntax correctness, safe imports, semantic tags, and valid component exports without executing a slow full build.

### Automated Checks
```bash
cd charisol-design-system-be
node --test tests/unit/code-validator.test.js
```

### Validation Rule Checklist
- [x] **Default Export**: Validates that component has `export default function ComponentName` or `export default ComponentName`.
- [x] **Safe Imports**: Allows `react`, `react-router-dom`, `./ComponentName.module.css`, and local relative paths (`../components/`, `../mock/`). Rejects arbitrary npm packages (e.g. `@mui/material`, `styled-components`).
- [x] **No Tailwind**: Rejects `@apply` or Tailwind utility class imports.
- [x] **Bracket Balancing**: Confirms matching curly braces `{}` in both TSX and CSS.
- [x] **CSS Module Classes**: Ensures CSS file defines at least one valid CSS class.
- [x] **PascalCase Name**: Validates component name structure.

---

## Task 1.4: Per-Screen Progress Tracking

### Objective
Verify that job records track individual screen generation statuses (`pending`, `generating`, `succeeded`, `failed`) and calculate proportional generation progress.

### Automated Checks
```bash
cd charisol-design-system-be
node --test tests/unit/screen-progress.test.js
```

### Manual API Test
1. **Trigger Screen Status Updates**:
   - Update screen status via `prototypeJobService.updateScreenStatus({ projectId, jobId, screenId, status: "succeeded", latencyMs: 3200 })`.
2. **Query the API**:
   - Send `GET /v1/projects/:projectId/prototype/jobs/:jobId/screens`.
   - Verify response schema:
     ```json
     {
       "status": true,
       "data": {
         "jobId": "job_gen_...",
         "phase": "generating_screens",
         "progress": 55,
         "totalScreens": 4,
         "completedScreens": 2,
         "screens": [
           { "screenId": "1:1", "name": "Home", "status": "succeeded", "latencyMs": 3200 },
           { "screenId": "1:2", "name": "Pricing", "status": "generating", "latencyMs": null },
           { "screenId": "1:3", "name": "About", "status": "pending", "latencyMs": null },
           { "screenId": "1:4", "name": "Contact", "status": "pending", "latencyMs": null }
         ]
       }
     }
     ```

---

## Phase 1 Gate Checklist

Before proceeding to Phase 2 (Assembly & Build), verify each gate condition:

- [ ] **Screen Package Assembly**: Unit test with fixture snapshot passes with all fields present.
- [ ] **Tree Pruning**: 500+ node trees prune to under 200 nodes and < 5,000 tokens.
- [ ] **Bedrock Screen Generation**: Mock & integration tests parse TSX and CSS with latency and token metrics.
- [ ] **Code Validator**: All 6 validation rules tested and passing.
- [ ] **Screen Progress API**: `GET /prototype/jobs/:jobId/screens` returns real-time screen progress.
- [ ] **Regression Suite**: All existing Phase 0 and baseline unit tests continue to pass.
