# File Change Manifest — Figma Plugin Migration

> Every file that is created, modified, or deleted, organized by phase and repository.

---

## Legend

| Symbol | Meaning |
|:-------|:--------|
| ✳️ **CREATE** | New file |
| ✏️ **MODIFY** | Existing file changed |
| 🗑️ **DELETE** | File removed entirely |
| 🔒 **KEEP** | File unchanged — listed for awareness |

---

## New Repository: `charisol-figma-plugin/`

All files below are ✳️ **CREATE**.

### Phase 0 — Scaffolding

```
charisol-figma-plugin/
├── manifest.json                          # Figma plugin manifest (editorType, networkAccess, documentAccess)
├── package.json                           # npm project with build/dev/test scripts
├── tsconfig.json                          # TypeScript strict config with @figma/plugin-typings
├── .gitignore
├── .eslintrc.json
├── README.md
├── LICENSE
│
├── src/
│   ├── code.ts                            # Plugin sandbox entry — figma.showUI, message handler
│   ├── ui.html                            # Plugin UI shell (loads bundled JS)
│   ├── ui.ts                              # Plugin UI logic — auth, network, progress
│   │
│   ├── types/
│   │   ├── messages.ts                    # PluginMessage union type (sandbox <-> UI)
│   │   ├── inventory.ts                   # Inventory schema TypeScript types (mirrors figmaInventorySchema.js)
│   │   └── api.ts                         # Backend API request/response types
│   │
│   ├── auth/
│   │   ├── authManager.ts                 # Login, token storage (figma.clientStorage), refresh
│   │   └── projectPicker.ts               # Fetch projects, select, persist choice
│   │
│   ├── walker/                            # Phase 1
│   │   ├── treeWalker.ts                  # Recursive document traversal (figma.root → all pages)
│   │   ├── nodeSerializer.ts              # Map Figma node → inventory schema node
│   │   ├── variableExtractor.ts           # figma.variables.getLocalVariablesAsync() → schema
│   │   ├── styleExtractor.ts              # figma.getLocalPaintStylesAsync() etc. → schema
│   │   └── prototypeExtractor.ts          # node.reactions, flowStartingPoints → schema
│   │
│   ├── exporter/                          # Phase 1
│   │   ├── assetExporter.ts               # exportAsync() per unique imageHash
│   │   └── hashUtils.ts                   # SHA-256 via crypto.subtle
│   │
│   ├── assembler/                         # Phase 1
│   │   ├── manifestAssembler.ts           # Combine all data → NormalizedInventory
│   │   └── schemaValidator.ts             # Port of validateNormalizedInventory()
│   │
│   ├── uploader/                          # Phase 1
│   │   └── uploadOrchestrator.ts          # Presign → S3 PUT → complete-upload
│   │
│   └── ui-components/                     # Phase 0-1 (UI rendering)
│       ├── LoginScreen.ts                 # Email/password form
│       ├── ProjectPicker.ts               # Project list with search
│       ├── ExportScreen.ts                # Scope selection, export button, progress bars
│       └── ResultScreen.ts                # Success/error, link to Strata dashboard
│
├── build/
│   └── esbuild.config.js                  # Build config: code.ts → dist/code.js, ui.ts → dist/ui.html
│
├── dist/                                  # Build output (gitignored)
│   ├── code.js
│   └── ui.html
│
└── tests/
    ├── walker/
    │   ├── treeWalker.test.ts
    │   ├── nodeSerializer.test.ts
    │   ├── variableExtractor.test.ts
    │   └── prototypeExtractor.test.ts
    ├── exporter/
    │   └── assetExporter.test.ts
    ├── assembler/
    │   ├── manifestAssembler.test.ts
    │   └── schemaValidator.test.ts
    └── uploader/
        └── uploadOrchestrator.test.ts
```

---

## Backend: `charisol-design-system-be/`

### Phase 2 — New Files (✳️ CREATE)

| File | Purpose |
|:-----|:--------|
| `controllers/pluginController.js` | `presignPluginUpload`, `completePluginUpload`, `startPluginJob` — new plugin endpoints |
| `tests/controllers/pluginController.test.js` | Unit tests for plugin controller |
| `tests/integration/pluginUpload.test.js` | Integration test for presign → upload → job flow |

### Phase 2 — Modified Files (✏️ MODIFY)

| File | Change | Lines Affected |
|:-----|:-------|:---------------|
| `utils/s3.js` | Add `generatePresignedPutUrl()` export | +15 lines |
| `routes/v1/index.js` | Add plugin routes, remove REST inventory/render routes | +10 / -8 lines |
| `controllers/prototypeController.js` | Remove `startInventory`, `startRender` from exports; keep all other exports | -70 lines (functions), -2 lines (exports) |
| `workers/prototypeWorkerSkeleton.js` | Remove `phase === "inventory"` and `phase === "render"` branches | -20 lines |
| `controllers/figmaController.js` | Evaluate: keep token-import endpoints if design-system sync still needs them | Conditional |
| `api-doc.js` or `docs/openapi.js` | Add plugin endpoints, remove REST inventory/render endpoints | +60 / -40 lines |
| `package.json` | Remove `axios` if only used for Figma REST | Conditional |

### Phase 2 — Deleted Files (🗑️ DELETE)

| File | Size | What It Contains |
|:-----|:-----|:-----------------|
| `services/figmaInventoryService.js` | 797 lines / 23.8 KB | `executeFigmaInventory()`, `buildFigmaInventory()`, `traverseNodeTree()`, REST API calls |
| `services/figmaRenderService.js` | 536 lines / 16.8 KB | `executeFigmaNodeRendering()`, `renderSelectedNodes()`, Figma Image API calls |
| `workers/figmaInventoryWorker.js` | 58 lines / 1.6 KB | CLI entry for REST inventory worker |
| `utils/figmaExtraction.js` | 227 lines / 7.6 KB | `fetchFigmaVariables()`, `fetchFigmaStyles()` via REST |
| `utils/figmaUrlParser.js` | 52 lines / 1.6 KB | `parseFigmaUrl()` — Figma URL regex parser |
| `utils/figmaHelpers.js` | ~60 lines / 2.2 KB | `resolveFigmaTokenFromProfile()` |
| **Total deleted** | **~1,730 lines / ~53.6 KB** | |

### Phase 2 — Unchanged Files (🔒 KEEP)

| File | Why Unchanged |
|:-----|:--------------|
| `services/figmaInventorySchema.js` | Schema validation reused by backend when receiving plugin uploads |
| `services/prototypeJobService.js` | Job creation/state machine — unchanged |
| `services/prototypeArtifactService.js` | Artifact upload/metadata — unchanged |
| `services/stepFunctionsService.js` | SFN orchestration — unchanged |
| `services/ownershipService.js` | Ownership tracking — unchanged |
| `services/githubAppService.js` | GitHub operations — unchanged |
| `constants/prototypeContracts.js` | State machine, error codes — unchanged |
| `models/PrototypeJob.js` | DynamoDB model — unchanged |
| `models/PrototypeArtifact.js` | DynamoDB model — unchanged |
| `models/FigmaInventory.js` | DynamoDB model — unchanged |
| `models/DeploymentRecord.js` | DynamoDB model — unchanged |
| `models/PreviewToken.js` | DynamoDB model — unchanged |
| `models/AuditEvent.js` | DynamoDB model — unchanged |
| `utils/auditLogger.js` | Audit events — unchanged |
| `utils/crypto.js` | Encryption — unchanged |
| `utils/s3.js` | Modified (add presign PUT), not deleted |

---

## Frontend: `charisol-design-system-fe/`

### Phase 3 — New Files (✳️ CREATE)

| File | Purpose |
|:-----|:--------|
| `components/sections/project/PrototypePluginGuide.tsx` | Plugin installation and usage guide component |
| `services/plugin.service.ts` | API client for plugin endpoints (presign, complete, jobs) |
| `app/api/plugin/presign-upload/route.ts` | BFF proxy for presign-upload |
| `app/api/plugin/complete-upload/route.ts` | BFF proxy for complete-upload |
| `app/api/plugin/jobs/route.ts` | BFF proxy for plugin jobs |

### Phase 3 — Modified Files (✏️ MODIFY)

| File | Change |
|:-----|:-------|
| Prototype tab component(s) | Replace URL-paste input with plugin guide; update state machine |
| `services/project.service.ts` | Remove REST inventory/render methods if they exist |

### Phase 3 — Potentially Removed (Conditional)

| File | Condition |
|:-----|:----------|
| `app/api/figma/connect/route.ts` | Remove only if token-import is also migrated to plugin |
| `app/api/figma/extract-preview/route.ts` | Remove only if token-import is also migrated to plugin |

---

## Infrastructure / CDK

### Phase 2 — Modified (✏️ MODIFY)

| File | Change |
|:-----|:-------|
| `cdk/lib/charisol-design-system-stack.ts` | Add S3 CORS configuration for Figma domain; potentially remove Figma-specific IAM policies if no REST calls remain |

---

## Dependency Impact Summary

### Backend `package.json` Dependencies to Evaluate

| Package | Current Use | After Migration |
|:--------|:------------|:----------------|
| `axios` | Figma REST API calls in `figmaInventoryService`, `figmaRenderService`, `figmaExtraction`, `figmaController` | Check if used elsewhere (e.g., GitHub, Vercel); remove if Figma was sole consumer |
| `@aws-sdk/client-s3` | S3 operations | Keep — add `@aws-sdk/s3-request-presigner` if not present |
| `@aws-sdk/s3-request-presigner` | May not be installed yet | Add for `generatePresignedPutUrl()` |

### Plugin `package.json` Dependencies

| Package | Purpose |
|:--------|:--------|
| `typescript` | Build |
| `@figma/plugin-typings` | Figma API type definitions |
| `esbuild` | Bundle code.ts → code.js, ui.ts → ui.html |
| `@figma/create-figma-plugin` | Optional CLI utilities |

---

## `require()` Reference Cleanup Checklist

After deleting files in Phase 2 Task 2.4, run this grep to find broken references:

```bash
grep -rn \
  "figmaInventoryService\|figmaRenderService\|figmaExtraction\|figmaUrlParser\|figmaHelpers\|resolveFigmaCredential\|buildFigmaInventory\|executeFigmaInventory\|executeFigmaNodeRendering\|fetchFigmaVariables\|fetchFigmaStyles\|parseFigmaUrl\|resolveFigmaTokenFromProfile" \
  --include='*.js' --include='*.ts' --include='*.tsx' \
  charisol-design-system-be/ charisol-design-system-fe/
```

**Expected result**: Zero matches (excluding this plan document and test fixtures).

---

## Net Code Change Summary

| Category | Lines Removed | Lines Added | Net |
|:---------|:-------------|:------------|:----|
| Backend deleted files | ~1,730 | 0 | -1,730 |
| Backend modifications | ~100 | ~200 | +100 |
| Plugin new repo | 0 | ~2,500 (est.) | +2,500 |
| Frontend changes | ~50 | ~300 | +250 |
| **Total** | **~1,880** | **~3,000** | **+1,120** |

The migration adds ~1,120 net lines but replaces fragile REST+rate-limit code with robust local Plugin API code. The backend shrinks by ~1,630 lines.
