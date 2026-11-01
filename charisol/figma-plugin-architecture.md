# Figma Plugin Export: Architecture & Implementation Plan

> **Status**: Draft — awaiting review
> **Created**: 2026-09-30
> **Replaces**: REST API–based Figma inventory approach (no fallback)

---

## Table of Contents

1. [Research Validation](#1-research-validation)
2. [Architecture Overview](#2-architecture-overview)
3. [Existing Code Inventory — What Stays, What Goes](#3-existing-code-inventory)
4. [Phase 0 — Plugin Project Scaffolding](#4-phase-0)
5. [Phase 1 — Plugin Core: Node Tree Serialization](#5-phase-1)
6. [Phase 2 — Backend Migration: Presigned Upload & REST Removal](#6-phase-2)
7. [Phase 3 — Frontend Migration: Plugin-Aware UX](#7-phase-3)
8. [Phase 4 — Polish, Testing, Documentation & Deployment](#8-phase-4)
9. [Risk Register](#9-risk-register)
10. [Glossary](#10-glossary)

---

## 1. Research Validation

### 1.1 Why Plugin API Over REST API

The Figma Plugin API runs **inside the Figma editor** on the user's machine, providing direct, unrestricted access to the document model. The REST API runs server-side and returns a JSON snapshot with significant limitations.

| Capability | Plugin API | REST API | Impact |
|:---|:---|:---|:---|
| **Full node tree** | Direct traversal via `figma.root`, `node.children` — full depth, all pages | JSON snapshot with depth limits; page juggling required | Plugin eliminates truncation/depth workarounds |
| **Auto-layout & constraints** | Direct property access (`layoutMode`, `primaryAxisAlignItems`, etc.) | Available but often incomplete at depth | Plugin gets authoritative layout data |
| **Prototype interactions** | `node.reactions` — triggers, actions, navigation, overlays, variable-set | `transitionNodeID` + limited `reactions` array | Plugin captures the full interaction graph |
| **Flow starting points** | `node.flowStartingPoints` on frames | Not directly available | Plugin can map prototype entry points |
| **Variables & modes** | `figma.variables.getLocalVariablesAsync()`, `getLocalVariableCollectionsAsync()` — full read | `GET /files/{key}/variables/local` — read-only, may 403 on some plans | Plugin gets all local variables reliably |
| **Styles** | `figma.getLocalPaintStylesAsync()`, `figma.getLocalTextStylesAsync()`, etc. | `GET /files/{key}/styles` — metadata only, no resolved values | Plugin gets resolved style values |
| **Images & vectors** | `node.exportAsync({format:'PNG'\|'SVG'\|'JPG', ...})` — local, no rate limit | `GET /images/{key}` — rate-limited (429), CDN URLs expire | Plugin exports locally; zero 429 risk |
| **Fonts** | `node.fontName`, `getStyledTextSegments(['fontName','fontSize',...])` per text segment | `style.fontFamily` on text nodes — single value, no segments | Plugin gets per-segment typography |
| **Rate limits** | **None** — all operations are local | ~30 req/min for images, variable per endpoint | Plugin removes the #1 operational bottleneck |
| **Image deduplication** | `imageHash` property on fill paints; export once per hash | Must download each CDN URL, compute hash client-side | Plugin deduplicates at source |

> **IMPORTANT**: The Plugin API has **zero rate limits** for node traversal, property reads, and `exportAsync`. This eliminates the 429 backoff/retry logic that accounts for ~30% of the current `figmaRenderService.js` complexity.

### 1.2 Trade-off: User Must Be in Figma

The one cost is UX friction: users must open Figma and run the plugin (instead of pasting a URL). This is acceptable because:

- The plugin runs once per export; it is not a continuous service
- Figma is already the designer's primary tool — they are already there
- The plugin can remember project association via `figma.clientStorage` (5 MB local KV)
- The plugin UI can show real-time progress, unlike a background REST crawl
- Login is one-time via `figma.clientStorage` auth token persistence

### 1.3 Plugin API Key Constraints

| Constraint | Mitigation |
|:---|:---|
| Cannot run in Prototype/Presentation mode | Export runs in Editor mode — designers always start there |
| Network requests must go through UI iframe | Standard pattern: sandbox → `postMessage` → UI iframe `fetch` → backend |
| `figma.clientStorage` limited to 5 MB | Store only auth token + project mapping; inventory goes to S3 |
| Cannot enumerate linked-library variables by key alone | Walk the local file; library variables resolve to their bound values |
| `documentAccess: "dynamic-page"` required for multi-page | Set in manifest; call `figma.loadAllPagesAsync()` before walk |

---

## 2. Architecture Overview

### 2.1 End-to-End Flow

```
Designer (Figma) → Plugin
  1. Opens plugin, selects project
  2. Walk node tree (figma.root)
  3. Serialize to inventory schema
  4. Export images via exportAsync()
  5. Deduplicate by imageHash

Plugin → Backend API
  6. POST /v1/plugin/presign-upload

Backend API → Plugin
  7. Returns {manifestUrl, assetUrls[]}

Plugin → S3
  8. PUT manifest.json (presigned)
  9. PUT assets/*.png (presigned)

Plugin → Backend API
  10. POST /v1/prototype-jobs {s3ManifestKey}

Backend → Job Worker Pipeline
  11. Enqueue job (existing pipeline)
  12. Read manifest + assets from S3
  13. React/HTML generation
  14. Deploy

Backend → Plugin
  15. Return {jobId, status}
  16. Show progress + preview link
```

### 2.2 Plugin Internal Architecture

```
Plugin Sandbox (code.ts) ──postMessage──▶ Plugin UI (ui.html/ts)
      │                                        │
      ├── Node Walker                          ├── Auth Manager
      ├── Schema Serializer                    ├── Network Layer (fetch)
      └── Asset Exporter                       └── Progress UI
           │                                        │
           ▼                                        ▼
    figma.root                              Backend API / S3
    figma.currentPage                       Presigned Upload
    exportAsync (PNG/SVG)
    figma.variables.*
```

### 2.3 Component Responsibilities

| Component | Responsibility |
|:---|:---|
| **Plugin Sandbox (`code.ts`)** | Walk document tree, serialize nodes to inventory schema, export images, send data to UI via postMessage |
| **Plugin UI (`ui.html/ts`)** | Auth flow, project picker, network requests (fetch), presigned S3 uploads, progress display |
| **Backend `/v1/plugin/*`** | Auth validation, presigned URL generation, job creation from S3 manifest key |
| **Backend existing pipeline** | `prototypeJobService`, `prototypeArtifactService`, `figmaInventorySchema` (reused as-is for validation) |
| **S3** | Receives manifest.json + assets directly from plugin (no backend proxy for large payloads) |
| **Workers** | Unchanged — read inventory from S3, generate React/HTML, deploy |

---

## 3. Existing Code Inventory

### 3.1 Code That Is **Reused** (No Changes or Minimal Adaptation)

| File | What's Reused |
|:---|:---|
| `services/figmaInventorySchema.js` | `formatNormalizedInventory()`, `validateNormalizedInventory()`, `generateStableIdentifiers()`, `SCHEMA_VERSION` — the plugin serializes directly to this schema |
| `services/prototypeJobService.js` | Entire service — job creation, state transitions, cancellation, idempotency |
| `services/prototypeArtifactService.js` | Entire service — artifact upload, metadata, presigned URL generation |
| `constants/prototypeContracts.js` | JOB_STATES, JOB_PHASES, ERROR_CODES, state machine — unchanged |
| `controllers/prototypeController.js` | `getJobStatus`, `cancelJob`, `retryJob`, `getArtifactMetadata`, `startDeployment`, `promoteDeployment`, preview token CRUD, config — all unchanged |
| `workers/prototypeWorkerSkeleton.js` | Worker orchestration skeleton — unchanged (render phase now receives assets from S3 instead of calling Figma API) |
| All DynamoDB models | `PrototypeJob`, `PrototypeArtifact`, `FigmaInventory`, `DeploymentRecord`, `PreviewToken`, `AuditEvent` — unchanged |
| `utils/s3.js` | S3 upload/download/presign utilities — add `generatePresignedPutUrl` |
| `utils/auditLogger.js` | Audit logging — unchanged |

### 3.2 Code That Is **Removed** (REST Figma Approach — No Fallback)

| File | What's Removed | Reason |
|:---|:---|:---|
| `services/figmaInventoryService.js` | **Entire file** — `executeFigmaInventory()`, `buildFigmaInventory()`, `traverseNodeTree()`, `resolveFigmaCredential()`, all Figma REST API calls | Plugin replaces REST-based inventory |
| `services/figmaRenderService.js` | **Entire file** — `executeFigmaNodeRendering()`, `renderSelectedNodes()`, all Figma Image API calls | Plugin exports images locally via `exportAsync()` |
| `workers/figmaInventoryWorker.js` | **Entire file** — CLI/Fargate entry point for REST inventory | No longer needed; inventory comes from plugin upload |
| `utils/figmaExtraction.js` | **Entire file** — `fetchFigmaVariables()`, `fetchFigmaStyles()` via REST API | Plugin reads variables/styles locally |
| `utils/figmaUrlParser.js` | **Entire file** — parses `figma.com/file/...` URLs | No URLs to parse; data comes from plugin |
| `utils/figmaHelpers.js` | **Entire file** — `resolveFigmaTokenFromProfile()` | No server-side Figma token resolution needed |
| `controllers/figmaController.js` | `connectFigma`, `extractFigmaPreview`, `syncFigmaTokens` — prototype-related endpoints | Replace with plugin-upload endpoint |
| `prototypeController.js` → `startInventory` | The REST-based inventory trigger | Replace with plugin-upload-initiated job |
| `prototypeController.js` → `startRender` | The REST-based render trigger | Renders come bundled with plugin upload |
| `prototypeWorkerSkeleton.js` → inventory branch | Calls `executeFigmaInventory` | Inventory already uploaded by plugin |
| `prototypeWorkerSkeleton.js` → render branch | Calls `executeFigmaNodeRendering` | Renders already uploaded by plugin |

### 3.3 Code That Is **Modified**

| File | Change |
|:---|:---|
| `utils/s3.js` | Add `generatePresignedPutUrl()` for plugin direct-to-S3 uploads |
| `routes/v1/index.js` | Add `/v1/plugin/*` routes; remove REST-based inventory/render routes |
| `controllers/prototypeController.js` | Replace `startInventory`/`startRender` with `startPluginJob`; add `presignPluginUpload` |
| Frontend prototype pages | Update UX to guide user to install/run plugin instead of pasting Figma URL |

---

## 4. Phase 0 — Plugin Project Scaffolding

**Phase Goal**: Create the Figma plugin project with TypeScript, manifest, build tooling, and the communication architecture between sandbox and UI.

### Task 0.1: Initialize Plugin Repository

**Goal**: Create a standalone Figma plugin project with TypeScript, bundler, and the correct manifest.

**Implementation Steps**:

1. Create a new directory `charisol-figma-plugin/` at the same level as the existing repos (or as a separate repo)
2. Initialize with `npm init`
3. Install dependencies:
   - `typescript`, `@figma/plugin-typings`
   - `esbuild` or `webpack` for bundling
   - `@figma/create-figma-plugin` utilities (optional — or use raw setup)
4. Create `manifest.json`:
   ```json
   {
     "name": "Strata Design System",
     "id": "<figma-plugin-id>",
     "api": "1.0.0",
     "main": "dist/code.js",
     "ui": "dist/ui.html",
     "editorType": ["figma"],
     "documentAccess": "dynamic-page",
     "networkAccess": {
       "allowedDomains": [
         "https://api.strata.design",
         "https://*.s3.amazonaws.com",
         "https://*.s3.*.amazonaws.com"
       ]
     }
   }
   ```
5. Create `tsconfig.json` with strict mode and Figma plugin typings
6. Create build scripts that produce `dist/code.js` (sandbox) and `dist/ui.html` (UI bundle)
7. Create `.gitignore`, `README.md`, `LICENSE`

**Validation Criteria**:
- [ ] `npm run build` produces `dist/code.js` and `dist/ui.html`
- [ ] Plugin loads in Figma desktop app (Developer → Import plugin from manifest)
- [ ] Plugin shows a blank UI panel when run
- [ ] TypeScript compiles with zero errors using `@figma/plugin-typings`

**Completion Criteria**: A developer can load the plugin in Figma and see a blank panel.

---

### Task 0.2: Plugin Sandbox/UI Communication Layer

**Goal**: Establish the `postMessage`-based communication protocol between the plugin sandbox (`code.ts`) and the UI iframe (`ui.ts`).

**Implementation Steps**:

1. Create `src/code.ts` — the plugin sandbox entry point:
   ```typescript
   figma.showUI(__html__, { width: 420, height: 640, themeColors: true });
   
   figma.ui.onmessage = async (msg: PluginMessage) => {
     switch (msg.type) {
       case 'START_EXPORT':
         // Will be implemented in Phase 1
         break;
       case 'CANCEL':
         figma.closePlugin();
         break;
     }
   };
   ```

2. Create `src/ui.tsx` (or `src/ui.ts` + `ui.html`) — the UI entry point:
   ```typescript
   window.onmessage = (event) => {
     const msg = event.data.pluginMessage;
     // Handle messages from sandbox
   };
   
   function sendToSandbox(msg: PluginMessage) {
     parent.postMessage({ pluginMessage: msg }, '*');
   }
   ```

3. Define shared message types in `src/types/messages.ts`:
   ```typescript
   type PluginMessage =
     | { type: 'START_EXPORT'; projectId: string; scope: ExportScope }
     | { type: 'EXPORT_PROGRESS'; phase: string; current: number; total: number }
     | { type: 'EXPORT_NODE_DATA'; nodes: SerializedNode[] }
     | { type: 'EXPORT_ASSET'; hash: string; data: Uint8Array; format: string }
     | { type: 'EXPORT_COMPLETE'; manifest: InventoryManifest }
     | { type: 'EXPORT_ERROR'; error: string; phase: string }
     | { type: 'AUTH_TOKEN'; token: string }
     | { type: 'CANCEL' }
   ```

4. Implement batched transfer for large node trees (chunk nodes into arrays of ~100 to avoid `postMessage` payload limits)

**Validation Criteria**:
- [ ] Sandbox can send a message to UI and receive a response
- [ ] UI can send a message to sandbox and receive a response
- [ ] Large payloads (>1MB simulated) transfer without loss via chunked protocol
- [ ] Message type exhaustiveness is enforced by TypeScript

**Completion Criteria**: Bidirectional communication works reliably with typed messages.

---

### Task 0.3: Auth & Project Picker UI

**Goal**: Implement one-time auth flow and project selection within the plugin UI.

**Implementation Steps**:

1. Create login screen in UI:
   - User enters Strata email/password or uses existing auth token
   - UI calls `POST /v1/auth/login` (or validates existing token)
   - On success, store auth token in `figma.clientStorage.setAsync('strata_auth_token', token)`
   - On subsequent launches, read from `figma.clientStorage.getAsync('strata_auth_token')` and validate

2. Create project picker:
   - Fetch user's projects via `GET /v1/projects` with auth token
   - Display project list with search/filter
   - Store selected project ID in `figma.clientStorage.setAsync('strata_selected_project', projectId)`

3. Create auto-reconnect logic:
   - On plugin open, check stored token validity
   - If valid, skip login and go directly to project picker or last-used project
   - If invalid/expired, show login screen

4. Handle token refresh/expiry gracefully

**Validation Criteria**:
- [ ] First-time user can log in and select a project
- [ ] Subsequent launches auto-reconnect without re-login
- [ ] Expired token triggers re-login flow
- [ ] Project list loads and filters correctly
- [ ] Auth token is never sent to `console.log` or exposed in plugin code variables

**Completion Criteria**: User can authenticate and select a project in <3 clicks on subsequent launches.

---

### Phase 0 Exit Criteria

- [ ] Plugin repo exists with TypeScript, build tooling, and Figma typings
- [ ] Plugin loads in Figma and shows UI
- [ ] Sandbox <-> UI communication works bidirectionally with typed messages
- [ ] Auth token persists in `figma.clientStorage`
- [ ] Project picker shows real projects from the backend API
- [ ] No secrets in `console.log`, source code, or manifest

---

## 5. Phase 1 — Plugin Core: Node Tree Serialization

**Phase Goal**: Walk the entire Figma document, serialize nodes to the existing inventory schema, export images locally, and prepare the upload bundle.

### Task 1.1: Document Tree Walker

**Goal**: Traverse the full document (or selection) and capture every node with its properties, respecting the existing `figmaInventorySchema.js` shape.

**Implementation Steps**:

1. Create `src/walker/treeWalker.ts`:
   ```typescript
   async function walkDocument(scope: ExportScope): Promise<WalkResult> {
     await figma.loadAllPagesAsync(); // Required for dynamic-page access
     
     const pages = scope.pageIds?.length
       ? figma.root.children.filter(p => scope.pageIds.includes(p.id))
       : figma.root.children;
     
     const nodeIndex = new Map<string, SerializedNode>();
     const parentMap = new Map<string, string | null>();
     const rootNodeIds: string[] = [];
     const warnings: Warning[] = [];
     let totalNodes = 0;
     
     for (const page of pages) {
       walkNode(page, null, true, 0, {
         nodeIndex, parentMap, rootNodeIds, warnings,
         state: { totalNodes: 0 },
         maxNodes: scope.maxNodes || 10000,
       });
     }
     
     return { nodeIndex, parentMap, rootNodeIds, warnings, totalNodes };
   }
   ```

2. Create `src/walker/nodeSerializer.ts` — maps each Figma node type to the inventory schema:
   - **Bounds**: `node.absoluteBoundingBox` → `{ x, y, width, height }`
   - **Render bounds**: `node.absoluteRenderBounds`
   - **Layout**: `layoutMode`, `primaryAxisAlignItems`, `counterAxisAlignItems`, `itemSpacing`, padding, `clipsContent`, `constraints`, `layoutSizingHorizontal/Vertical`, `layoutGrow`, `layoutAlign`
   - **Fills/Strokes**: compact paint arrays with color, gradient stops, image refs
   - **Effects**: drop shadows, inner shadows, blurs
   - **Text**: `characters`, `fontName`, `fontSize`, `lineHeight`, `letterSpacing`, `textAlignHorizontal` — plus `getStyledTextSegments()` for mixed-style text
   - **Component references**: `componentId` (for instances), `componentProperties`, `componentPropertyDefinitions`
   - **Prototype interactions**: `node.reactions` → triggers, actions, navigation destinations
   - **Flow starting points**: `node.flowStartingPoints` on frames
   - **Visibility**: `node.visible`
   - **Corner radius**: `cornerRadius`, `topLeftRadius`, etc.
   - **Opacity/blendMode**: `node.opacity`, `node.blendMode`

3. Ensure every serialized node includes `stableId`, `componentKey`, `routeKey` using the same `generateStableIdentifiers()` logic from `figmaInventorySchema.js`

4. Report progress back to UI via `figma.ui.postMessage()` every ~100 nodes

**Validation Criteria**:
- [ ] Walks all pages when no scope filter is applied
- [ ] Walks only selected pages when `scope.pageIds` is specified
- [ ] Walks only selected nodes when `scope.nodeIds` is specified
- [ ] Captures all node types: FRAME, GROUP, COMPONENT, COMPONENT_SET, INSTANCE, TEXT, VECTOR, RECTANGLE, ELLIPSE, LINE, SECTION, BOOLEAN_OPERATION
- [ ] Parent map is complete — every node has a parent entry or is a root
- [ ] Hidden nodes are captured with `visible: false`
- [ ] Layout properties are present on auto-layout frames
- [ ] Text segments with mixed fonts produce per-segment `fontName` data
- [ ] Progress messages arrive in UI during walk
- [ ] Node count >= expected for a known test file

**Completion Criteria**: Given any Figma file, the walker produces a node index that passes `validateNormalizedInventory()`.

---

### Task 1.2: Variables & Styles Extraction

**Goal**: Read all local variables, variable collections, and styles from the document and serialize them to the inventory schema.

**Implementation Steps**:

1. Create `src/walker/variableExtractor.ts`:
   ```typescript
   async function extractVariables(): Promise<ExtractedVariable[]> {
     const collections = await figma.variables.getLocalVariableCollectionsAsync();
     const variables: ExtractedVariable[] = [];
     
     for (const collection of collections) {
       for (const varId of collection.variableIds) {
         const variable = await figma.variables.getVariableByIdAsync(varId);
         if (!variable) continue;
         
         for (const [modeId, value] of Object.entries(variable.valuesByMode)) {
           const modeName = collection.modes.find(m => m.modeId === modeId)?.name || modeId;
           variables.push({
             key: sanitizeTokenKey(variable.name),
             value: resolveVariableValue(value, variable.resolvedType),
             category: resolveCategory(variable.resolvedType),
             figmaVariableId: variable.id,
             figmaVariableName: variable.name,
             figmaCollectionName: collection.name,
             figmaMode: modeName,
             figmaType: variable.resolvedType,
           });
         }
       }
     }
     return variables;
   }
   ```

2. Create `src/walker/styleExtractor.ts`:
   ```typescript
   async function extractStyles(): Promise<ExtractedStyle[]> {
     const [paintStyles, textStyles, effectStyles, gridStyles] = await Promise.all([
       figma.getLocalPaintStylesAsync(),
       figma.getLocalTextStylesAsync(),
       figma.getLocalEffectStylesAsync(),
       figma.getLocalGridStylesAsync(),
     ]);
     
     // Serialize each style type with resolved values
     // Paint styles -> color values extracted from fills
     // Text styles -> fontFamily, fontSize, lineHeight, letterSpacing
     // Effect styles -> shadows, blurs
     // Grid styles -> pattern, sectionSize, gutterSize
   }
   ```

3. Ensure variable/style data matches the shape expected by `formatNormalizedInventory()` — specifically `fileVariables` and `fileStyles` arrays

**Validation Criteria**:
- [ ] All local variable collections are captured
- [ ] Multi-mode variables produce one entry per mode
- [ ] Color variables resolve to hex or RGBA values
- [ ] Float/integer variables resolve to numeric strings
- [ ] All paint, text, effect, and grid styles are captured
- [ ] Style values are resolved (not just metadata)
- [ ] Output matches `figmaInventorySchema.js` expectations

**Completion Criteria**: Variables and styles populate the `variables` and `styles` arrays of the normalized inventory.

---

### Task 1.3: Prototype Interactions & Flow Extraction

**Goal**: Capture all prototype connections, reactions, and flow starting points.

**Implementation Steps**:

1. Create `src/walker/prototypeExtractor.ts`:
   ```typescript
   function extractReactions(node: SceneNode): PrototypeInteractions | null {
     if (!('reactions' in node) || !node.reactions.length) return null;
     
     return {
       reactions: node.reactions.map(reaction => ({
         trigger: { type: reaction.trigger?.type || 'ON_CLICK' },
         action: reaction.actions?.map(action => ({
           type: action.type,
           destinationId: action.type === 'NODE' ? action.destinationId : null,
           navigation: action.navigation,
           transition: action.transition ? {
             type: action.transition.type,
             duration: action.transition.duration,
             easing: action.transition.easing,
           } : null,
         })),
       })),
     };
   }
   ```

2. Extract flow starting points from top-level frames:
   ```typescript
   function extractFlowStartingPoints(page: PageNode): FlowStartingPoint[] {
     const flows: FlowStartingPoint[] = [];
     for (const child of page.children) {
       if ('flowStartingPoints' in child && child.flowStartingPoints.length) {
         flows.push(...child.flowStartingPoints.map(fp => ({
           nodeId: child.id,
           name: fp.name,
           description: fp.description,
         })));
       }
     }
     return flows;
   }
   ```

3. Build a prototype link graph — source -> destination for all navigation actions

**Validation Criteria**:
- [ ] Click-to-navigate interactions are captured with source and destination node IDs
- [ ] Overlay interactions are captured with overlay target
- [ ] Transition properties (duration, easing) are captured
- [ ] Flow starting points are identified on the correct frames
- [ ] Output matches the `prototypeLinks` array shape in the inventory schema
- [ ] Variable-set actions are captured where present

**Completion Criteria**: The prototype interaction graph is complete enough to generate navigation in the React prototype.

---

### Task 1.4: Asset Export & Deduplication

**Goal**: Export all image-bearing nodes as PNG/SVG using `exportAsync()`, deduplicate by `imageHash`, and prepare for upload.

**Implementation Steps**:

1. Create `src/exporter/assetExporter.ts`:
   ```typescript
   async function exportAssets(
     nodeIndex: Map<string, SerializedNode>,
     options: ExportOptions
   ): Promise<AssetManifest> {
     const imageHashes = new Set<string>();
     const assets: AssetEntry[] = [];
     
     // 1. Collect unique image hashes from fills
     for (const [nodeId, node] of nodeIndex) {
       if (node.fills) {
         for (const fill of node.fills) {
           if (fill.type === 'IMAGE' && fill.imageRef) {
             imageHashes.add(fill.imageRef);
           }
         }
       }
     }
     
     // 2. Export unique images
     for (const hash of imageHashes) {
       const image = figma.getImageByHash(hash);
       if (!image) continue;
       
       const bytes = await image.getBytesAsync();
       assets.push({
         hash,
         data: bytes,
         format: 'png',
         sizeBytes: bytes.length,
       });
       
       figma.ui.postMessage({
         type: 'EXPORT_PROGRESS',
         phase: 'assets',
         current: assets.length,
         total: imageHashes.size,
       });
     }
     
     // 3. Export top-level frames as screenshots (for preview)
     for (const [nodeId, node] of nodeIndex) {
       if (node.isRoot && ['FRAME', 'COMPONENT', 'COMPONENT_SET'].includes(node.type)) {
         const figmaNode = figma.getNodeById(nodeId);
         if (!figmaNode || !('exportAsync' in figmaNode)) continue;
         
         const bytes = await figmaNode.exportAsync({
           format: 'PNG',
           constraint: { type: 'SCALE', value: options.scale || 2 },
         });
         
         const frameHash = await computeHash(bytes);
         assets.push({
           hash: frameHash,
           data: bytes,
           format: 'png',
           sizeBytes: bytes.length,
           nodeId,
           isFrameRender: true,
         });
       }
     }
     
     return { assets, totalUniqueImages: imageHashes.size };
   }
   ```

2. Implement SHA-256 hashing in the plugin for content deduplication:
   ```typescript
   async function computeHash(data: Uint8Array): Promise<string> {
     const hashBuffer = await crypto.subtle.digest('SHA-256', data);
     return Array.from(new Uint8Array(hashBuffer))
       .map(b => b.toString(16).padStart(2, '0'))
       .join('');
   }
   ```

3. Map `imageRef` -> asset hash in the inventory so the backend can resolve image references

**Validation Criteria**:
- [ ] Images used in multiple nodes are exported only once (deduplicated by hash)
- [ ] Top-level frames are exported as preview screenshots
- [ ] Export progress is reported to UI
- [ ] Each asset has a content hash for immutable S3 naming
- [ ] Asset manifest maps `imageRef` -> S3 key for downstream resolution
- [ ] Empty/transparent images are skipped
- [ ] Export errors on individual nodes don't halt the entire export

**Completion Criteria**: All images are exported, deduplicated, and ready for presigned S3 upload.

---

### Task 1.5: Manifest Assembly & Schema Validation

**Goal**: Assemble the complete inventory manifest from all extracted data, validate it against the existing schema, and prepare for upload.

**Implementation Steps**:

1. Create `src/assembler/manifestAssembler.ts` — takes all extracted data and produces the canonical inventory object

2. Port `formatNormalizedInventory()` and `validateNormalizedInventory()` logic to the plugin (or share via a published npm package)

3. Run validation before upload — if validation fails, show errors in the plugin UI

4. The manifest should include `pluginSource` metadata:
   ```typescript
   pluginSource: {
     pluginVersion: PLUGIN_VERSION,
     exportedBy: 'figma-plugin',
     exportedAt: new Date().toISOString(),
   }
   ```

**Validation Criteria**:
- [ ] Assembled manifest passes `validateNormalizedInventory()` with zero errors
- [ ] `schemaVersion` is set to `"1.0.0"`
- [ ] `source.fileKey` is populated from file metadata
- [ ] All node IDs in `parentMap` have corresponding entries in `nodes`
- [ ] All node IDs in `rootNodeIds` exist in `nodes`
- [ ] `prototypeLinks` references valid source and destination node IDs
- [ ] Asset manifest hashes are unique
- [ ] Warnings from truncation/errors are included

**Completion Criteria**: The manifest is valid, complete, and structurally identical to what the backend expects.

---

### Task 1.6: Upload Orchestrator

**Goal**: Orchestrate the full upload flow: request presigned URLs, upload manifest + assets to S3, trigger job creation.

**Implementation Steps**:

1. Create `src/uploader/uploadOrchestrator.ts`:
   ```typescript
   async function uploadBundle(
     manifest: NormalizedInventory,
     assets: AssetEntry[],
     projectId: string,
     authToken: string
   ): Promise<UploadResult> {
     // 1. Request presigned URLs from backend
     const fileList = [
       { name: 'manifest.json', contentType: 'application/json', sizeBytes: ... },
       ...assets.map(a => ({ name: `assets/${a.hash}.png`, contentType: 'image/png', sizeBytes: a.sizeBytes })),
     ];
     
     const presignResponse = await fetch(`${API_BASE}/v1/projects/${projectId}/plugin/presign-upload`, {
       method: 'POST',
       headers: { 'Authorization': `Bearer ${authToken}`, 'Content-Type': 'application/json' },
       body: JSON.stringify({ files: fileList }),
     });
     
     const { uploadId, urls } = await presignResponse.json();
     
     // 2. Upload manifest to S3
     const manifestUrl = urls.find(u => u.name === 'manifest.json');
     await fetch(manifestUrl.presignedUrl, {
       method: 'PUT',
       body: JSON.stringify(manifest),
       headers: { 'Content-Type': 'application/json' },
     });
     
     // 3. Upload assets to S3 (parallel, with concurrency limit)
     await uploadAssetsWithConcurrency(assets, urls, 5);
     
     // 4. Complete upload
     const completeResponse = await fetch(`${API_BASE}/v1/projects/${projectId}/plugin/complete-upload`, {
       method: 'POST',
       headers: { 'Authorization': `Bearer ${authToken}`, 'Content-Type': 'application/json' },
       body: JSON.stringify({ uploadId }),
     });
     
     return await completeResponse.json();
   }
   ```

2. Implement parallel upload with concurrency limit (max 5 simultaneous S3 PUTs)

3. Report upload progress to UI

4. Handle upload failures with retry (up to 3 attempts per file)

**Validation Criteria**:
- [ ] Manifest uploads successfully to S3
- [ ] All assets upload successfully to S3
- [ ] Backend `complete-upload` returns a valid job ID
- [ ] Failed uploads retry up to 3 times
- [ ] Progress is reported during upload
- [ ] Concurrency limit prevents overwhelming the network

**Completion Criteria**: The entire bundle uploads to S3 and a job is created in the backend.

---

### Phase 1 Exit Criteria

- [ ] Plugin walks entire Figma document (all pages) or scoped selection
- [ ] Node tree serialization matches `figmaInventorySchema.js` v1.0.0
- [ ] Variables and styles are extracted with resolved values
- [ ] Prototype interactions and flow starting points are captured
- [ ] Images are exported locally and deduplicated by hash
- [ ] Complete manifest passes schema validation
- [ ] Upload to S3 works via presigned URLs
- [ ] Job creation succeeds in the backend
- [ ] Progress is reported to user throughout the export and upload
- [ ] No Figma REST API calls are made anywhere in the plugin

---

## 6. Phase 2 — Backend Migration: Presigned Upload & REST Removal

**Phase Goal**: Add new backend endpoints for plugin presigned uploads, create the plugin-initiated job flow, and remove all REST-based Figma code.

### Task 2.1: Add Presigned PUT URL Generation

**Goal**: Add S3 presigned PUT URL generation to the backend so the plugin can upload directly to S3.

**Files to Modify**:
- `utils/s3.js` — add `generatePresignedPutUrl()`

**Implementation Steps**:

1. Add `generatePresignedPutUrl()` to `s3.js`:
   ```javascript
   async function generatePresignedPutUrl({ bucket, key, contentType, expiresIn = 3600 }) {
     const command = new PutObjectCommand({
       Bucket: bucket,
       Key: key,
       ContentType: contentType,
     });
     return await getSignedUrl(s3Client, command, { expiresIn });
   }
   ```

2. Configure S3 bucket CORS to allow `PUT` from `https://www.figma.com`:
   ```json
   {
     "CORSRules": [{
       "AllowedHeaders": ["*"],
       "AllowedMethods": ["PUT"],
       "AllowedOrigins": ["https://www.figma.com", "https://figma.com"],
       "ExposeHeaders": ["ETag"],
       "MaxAgeSeconds": 3600
     }]
   }
   ```

**Validation Criteria**:
- [ ] `generatePresignedPutUrl()` returns a valid URL
- [ ] URL allows `PUT` with correct content type
- [ ] URL expires after configured time
- [ ] CORS headers are set on S3 bucket

**Completion Criteria**: The plugin can upload files directly to S3 using presigned URLs.

---

### Task 2.2: Plugin Upload Endpoint

**Goal**: Create `POST /v1/projects/:projectId/plugin/presign-upload` — accepts a manifest describing what will be uploaded, returns presigned PUT URLs for each file.

**Files to Create/Modify**:
- New: `controllers/pluginController.js`
- Modify: `routes/v1/index.js`

**Implementation Steps**:

1. Create `controllers/pluginController.js` with two endpoints:

   **`presignPluginUpload`** — generates presigned URLs:
   ```javascript
   async function presignPluginUpload(req, res) {
     const projectId = req.params.projectId;
     const { files } = req.body;
     // files = [{ name: 'manifest.json', contentType: 'application/json', sizeBytes: 12345 }, ...]
     
     // Validate: max 500 files, max 100MB total, valid content types
     // Generate uploadId
     // Generate S3 keys: projects/{projectId}/plugin-uploads/{uploadId}/{filename}
     // Generate presigned PUT URLs for each file
     // Return { uploadId, urls: [{ name, presignedUrl, s3Key }] }
   }
   ```

   **`completePluginUpload`** — called after all files are uploaded:
   ```javascript
   async function completePluginUpload(req, res) {
     const projectId = req.params.projectId;
     const { uploadId } = req.body;
     
     // 1. Verify manifest.json exists in S3 at expected key
     // 2. Read and validate manifest against figmaInventorySchema
     // 3. Create FigmaInventory record
     // 4. Create PrototypeJob with state=READY_FOR_REVIEW (inventory already done)
     // 5. Upload artifact metadata
     // 6. Return { jobId, inventoryId, status }
   }
   ```

2. Register routes:
   ```javascript
   router.post('/projects/:projectId/plugin/presign-upload', auth, authorize('editor'), pluginController.presignPluginUpload);
   router.post('/projects/:projectId/plugin/complete-upload', auth, authorize('editor'), pluginController.completePluginUpload);
   ```

**Validation Criteria**:
- [ ] Endpoint requires authentication and editor role
- [ ] Returns presigned URLs for each requested file
- [ ] Rejects requests exceeding file count/size limits
- [ ] `complete-upload` validates the manifest against schema
- [ ] `complete-upload` creates job and inventory records
- [ ] Invalid manifest returns 422 with detailed errors
- [ ] Audit events are recorded for both endpoints

**Completion Criteria**: Plugin can request upload URLs, upload files to S3, and trigger job creation.

---

### Task 2.3: Plugin-Initiated Job Flow

**Goal**: Replace `startInventory` and `startRender` with a single plugin-initiated job that begins the pipeline from the inventory-already-uploaded state.

**Files to Modify**:
- `controllers/prototypeController.js`
- `controllers/pluginController.js`

**Implementation Steps**:

1. Add `startPluginJob()` to `pluginController.js`:
   ```javascript
   async function startPluginJob(req, res) {
     const projectId = req.params.projectId;
     const { s3ManifestKey, uploadId, idempotencyKey } = req.body;
     
     // 1. Validate s3ManifestKey exists and is a valid inventory
     // 2. Create job starting at READY_FOR_REVIEW phase (inventory + renders already done)
     // 3. When user triggers generation, pipeline starts from generate phase
     // 4. Return { jobId, state, phase }
   }
   ```

2. Modify `prototypeWorkerSkeleton.js` to skip inventory/render phases when manifest is plugin-sourced

3. Ensure the generate/validate/github/deploy phases work unchanged — they read from S3

**Validation Criteria**:
- [ ] Job is created with `READY_FOR_REVIEW` state
- [ ] Job skips inventory and render phases
- [ ] Generation phase reads manifest from S3 correctly
- [ ] Existing generation/deploy pipeline works with plugin-sourced data
- [ ] Idempotency works for plugin uploads

**Completion Criteria**: The full pipeline works from plugin upload through generation, GitHub, and deploy.

---

### Task 2.4: Remove REST Figma Approach

**Goal**: Delete all REST-based Figma inventory, rendering, and credential resolution code. No fallback.

**Files to Delete**:
- `services/figmaInventoryService.js`
- `services/figmaRenderService.js`
- `workers/figmaInventoryWorker.js`
- `utils/figmaExtraction.js`
- `utils/figmaUrlParser.js`
- `utils/figmaHelpers.js`

**Files to Modify**:
- `controllers/prototypeController.js` — remove `startInventory`, `startRender`
- `controllers/figmaController.js` — remove prototype-related endpoints (keep token-import endpoints if still needed for design system sync)
- `routes/v1/index.js` — remove routes for REST inventory/render
- `workers/prototypeWorkerSkeleton.js` — remove inventory/render branches

**Implementation Steps**:

1. Delete the files listed above
2. Remove all `require()` references to deleted files
3. Remove REST-based inventory/render routes from `routes/v1/index.js`
4. Remove `startInventory` and `startRender` from `prototypeController.js` exports
5. Remove inventory/render branches from `prototypeWorkerSkeleton.js`
6. Update `package.json` — remove `axios` if no longer needed elsewhere
7. Run `grep -r "figmaInventoryService\|figmaRenderService\|figmaExtraction\|figmaUrlParser\|figmaHelpers\|resolveFigmaCredential\|buildFigmaInventory\|executeFigmaInventory\|executeFigmaNodeRendering" --include='*.js'` to find any remaining references
8. Fix any broken references

**NOTE**: **Preserve** `figmaController.js` token-import endpoints (`connectFigma`, `extractFigmaPreview`, `applyFigmaImport`, `syncFigmaTokens`) **only if** the design-system token sync feature still needs them. If the plugin will also replace token sync, mark those for Phase 4 removal.

**Validation Criteria**:
- [ ] All deleted files are gone
- [ ] No `require()` references to deleted files remain
- [ ] `grep` for deleted function names returns zero results
- [ ] Backend starts without errors
- [ ] All non-deleted tests pass
- [ ] OpenAPI spec is updated (removed endpoints removed from spec)
- [ ] No REST Figma API calls remain in the codebase for prototype flow

**Completion Criteria**: The backend has zero REST-based Figma code for the prototype workflow. The only way to create an inventory is through the plugin upload path.

---

### Task 2.5: Update OpenAPI & API Documentation

**Goal**: Update the OpenAPI spec to reflect new plugin endpoints and removed REST endpoints.

**Files to Modify**:
- `api-doc.js` or `docs/openapi.js`

**Implementation Steps**:

1. Add OpenAPI operations for:
   - `POST /v1/projects/{projectId}/plugin/presign-upload`
   - `POST /v1/projects/{projectId}/plugin/complete-upload`
   - `POST /v1/projects/{projectId}/plugin/jobs`

2. Remove OpenAPI operations for:
   - `POST /v1/projects/{projectId}/prototype/inventory`
   - `POST /v1/projects/{projectId}/prototype/render`

3. Update request/response schemas for plugin endpoints

**Validation Criteria**:
- [ ] `yarn docs:validate` passes
- [ ] New endpoints have request bodies, response schemas, auth requirements, and error responses documented
- [ ] Removed endpoints no longer appear in spec

**Completion Criteria**: API documentation is accurate and complete.

---

### Phase 2 Exit Criteria

- [ ] Presigned PUT URL generation works for S3
- [ ] Plugin can upload manifest + assets directly to S3
- [ ] Plugin-initiated job starts from READY_FOR_REVIEW (skipping inventory/render)
- [ ] Full pipeline works: plugin upload → generation → GitHub → deploy
- [ ] All REST Figma code is deleted (no fallback)
- [ ] OpenAPI spec is updated
- [ ] Backend tests pass
- [ ] No Figma API credentials are stored or transmitted by the backend

---

## 7. Phase 3 — Frontend Migration: Plugin-Aware UX

**Phase Goal**: Update the frontend to guide users through the plugin-based workflow instead of URL-pasting.

### Task 3.1: Plugin Installation Guide Component

**Goal**: Create a component that guides users to install the Figma plugin.

**Implementation Steps**:

1. Create `components/sections/project/PrototypePluginGuide.tsx`:
   - Show plugin installation link (Figma Community URL)
   - Step-by-step visual guide: Install → Open in Figma → Run Plugin → Select Project → Export
   - Detect if plugin has already exported (check for existing inventory) and show "Re-export" flow

2. Replace the current "Paste Figma URL" input with the plugin guide

3. Add a status indicator showing the latest plugin export timestamp and inventory stats

**Validation Criteria**:
- [ ] Plugin guide shows clear installation and usage instructions
- [ ] Link to Figma Community plugin page is correct
- [ ] Guide is only shown when no inventory exists (first-time flow)
- [ ] Re-export option is shown when an inventory already exists

**Completion Criteria**: Users understand how to install and use the plugin from the frontend.

---

### Task 3.2: Update Prototype Tab State Machine

**Goal**: Update the frontend prototype workflow to handle the plugin-sourced flow.

**Implementation Steps**:

1. Update prototype state machine to handle new states:
   - `AWAITING_PLUGIN_EXPORT` — no inventory yet; show plugin guide
   - `PLUGIN_EXPORT_RECEIVED` — inventory uploaded by plugin; show review UI
   - `GENERATING` → `DEPLOYING` → `DEPLOYED` — unchanged

2. Remove the "Start Inventory" button that triggers REST-based inventory
3. Remove the "Start Render" button that triggers REST-based rendering
4. Update job polling to handle the new `READY_FOR_REVIEW` initial state

**Validation Criteria**:
- [ ] Prototype tab shows plugin guide when no inventory exists
- [ ] Tab shows inventory review when plugin export is received
- [ ] No UI elements trigger REST-based inventory or render
- [ ] Job polling works correctly with plugin-sourced jobs

**Completion Criteria**: The frontend prototype flow is fully plugin-driven.

---

### Task 3.3: Update BFF Routes

**Goal**: Add Next.js BFF routes for the new plugin endpoints.

**Implementation Steps**:

1. Create `app/api/plugin/presign-upload/route.ts` — proxy to backend
2. Create `app/api/plugin/complete-upload/route.ts` — proxy to backend
3. Create `app/api/plugin/jobs/route.ts` — proxy to backend
4. Remove BFF routes for REST-based inventory/render if they exist

**Validation Criteria**:
- [ ] BFF routes proxy correctly to backend with auth cookies
- [ ] Error responses are properly forwarded
- [ ] No sensitive data is logged in BFF

**Completion Criteria**: Frontend can communicate with new backend endpoints through BFF.

---

### Phase 3 Exit Criteria

- [ ] Frontend guides users to install and use the Figma plugin
- [ ] No UI elements trigger REST-based Figma operations
- [ ] Prototype tab state machine handles plugin-sourced flow
- [ ] BFF routes work correctly for plugin endpoints
- [ ] Frontend builds and passes lint/typecheck

---

## 8. Phase 4 — Polish, Testing, Documentation & Deployment

**Phase Goal**: End-to-end testing, plugin publishing preparation, documentation, and deployment.

### Task 4.1: End-to-End Integration Tests

**Goal**: Verify the complete flow from plugin export through generation and deployment.

**Implementation Steps**:

1. Create fixture Figma files (or use Figma test files) with known node counts, styles, variables, and interactions
2. Create integration tests that:
   - Simulate plugin manifest upload to S3
   - Trigger job creation via API
   - Verify inventory is valid
   - Verify generation produces expected output
   - Verify deploy works

3. Create plugin-side tests:
   - Unit tests for node serializer
   - Unit tests for variable/style extractor
   - Unit tests for manifest assembler
   - Integration test for full walk → serialize → validate cycle

**Validation Criteria**:
- [ ] Known fixture file produces expected node count
- [ ] Generated prototype has expected routes and components
- [ ] Deployed prototype loads and navigates correctly
- [ ] Error cases are handled gracefully (missing nodes, invalid data)

**Completion Criteria**: The complete flow works end-to-end with automated tests.

---

### Task 4.2: Plugin Publishing Preparation

**Goal**: Prepare the Figma plugin for publishing to the Figma Community.

**Implementation Steps**:

1. Create plugin icon (128x128 PNG)
2. Create plugin cover image (1920x960 PNG)
3. Write plugin description and usage instructions
4. Set up plugin review submission
5. Create privacy policy page (required by Figma)
6. Create terms of service page
7. Configure production `networkAccess` domains in manifest
8. Add analytics/telemetry (optional, with consent)

**Validation Criteria**:
- [ ] Plugin passes Figma's automated review checks
- [ ] Plugin icon and cover are correct dimensions
- [ ] Privacy policy is accessible
- [ ] `networkAccess` domains are production URLs

**Completion Criteria**: Plugin is ready for Figma Community submission.

---

### Task 4.3: Documentation

**Goal**: Create user-facing and developer-facing documentation.

**Implementation Steps**:

1. **User guide**: How to install, authenticate, select scope, export, and view results in Strata
2. **Developer guide**: Plugin architecture, how to build/test, how to add new node types
3. **API migration guide**: What changed from REST to plugin approach
4. **Backend API reference**: New plugin endpoints
5. Update existing `README.md` files in both repos

**Validation Criteria**:
- [ ] User guide covers the complete happy path
- [ ] Developer guide covers building, testing, and extending
- [ ] API migration guide lists all removed and added endpoints

**Completion Criteria**: Documentation is complete and accurate.

---

### Task 4.4: Backend Cleanup & Performance

**Goal**: Final cleanup of dead code references and performance optimization.

**Implementation Steps**:

1. Remove any remaining `require()` to deleted files
2. Remove unused npm dependencies (`axios` if only used for Figma)
3. Verify all tests pass after removal
4. Add performance monitoring for presigned URL generation latency
5. Add CloudWatch alarms for plugin upload failures
6. Add S3 lifecycle rules for plugin upload cleanup (delete incomplete uploads after 24h)

**Validation Criteria**:
- [ ] `npm audit` shows no new vulnerabilities
- [ ] No dead imports or references
- [ ] All tests pass
- [ ] S3 lifecycle rules are configured
- [ ] CloudWatch alarms are set up

**Completion Criteria**: Codebase is clean, monitored, and production-ready.

---

### Phase 4 Exit Criteria

- [ ] End-to-end tests pass for the complete flow
- [ ] Plugin is ready for Figma Community submission
- [ ] Documentation is complete
- [ ] Backend is clean with no dead code
- [ ] Performance monitoring is in place
- [ ] S3 bucket has appropriate lifecycle and CORS policies

---

## 9. Risk Register

| # | Risk | Likelihood | Impact | Mitigation |
|:--|:-----|:-----------|:-------|:-----------|
| R1 | Large Figma files (>10k nodes) may slow plugin export | Medium | Medium | Progress UI, chunked processing, configurable max-node limit |
| R2 | `exportAsync` may fail on certain node types | Low | Low | Wrap in try/catch per node; log warning; continue with remaining |
| R3 | S3 presigned URL expiry during slow upload | Low | Medium | Set generous expiry (1h); retry with fresh URL on 403 |
| R4 | CORS misconfiguration blocks S3 upload | Medium | High | Test CORS config in staging before plugin release |
| R5 | Plugin review rejection by Figma | Low | Medium | Follow Figma guidelines strictly; submit for review early |
| R6 | Schema drift between plugin and backend | Medium | High | Share schema via npm package or validate on backend receipt |
| R7 | `figma.clientStorage` cleared by user | Low | Low | Re-authenticate gracefully; no data loss |
| R8 | Designers forget to re-export after changes | Medium | Low | Show last-export timestamp; suggest re-export in UI |

---

## 10. Glossary

| Term | Definition |
|:-----|:-----------|
| **Plugin Sandbox** | The main thread of the Figma plugin (`code.ts`) — has access to the document model but no network |
| **Plugin UI** | The iframe-based UI (`ui.html`) — has network access but no document model access |
| **Inventory Schema** | The `figmaInventorySchema.js` v1.0.0 format — the canonical representation of a Figma document |
| **Presigned URL** | A time-limited S3 URL that allows the plugin to upload directly without backend proxying |
| **postMessage** | The browser API used for communication between the plugin sandbox and UI iframe |
| **figma.clientStorage** | A 5MB local key-value store in the Figma plugin runtime — persists between plugin sessions |
| **exportAsync** | The Figma Plugin API method to export a node as PNG, SVG, JPG, or PDF locally |
| **imageHash** | Figma's internal content hash for images — used for deduplication |
| **Flow Starting Point** | A Figma prototype entry point defined on a top-level frame |
| **node.reactions** | The array of prototype interactions (triggers + actions) on a Figma node |
