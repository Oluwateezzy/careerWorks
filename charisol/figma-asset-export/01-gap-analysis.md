# 01 — Gap Analysis: Asset Flow Today

## 1. Current Flow

```
Figma plugin (sandbox)                      Plugin UI (iframe)
─────────────────────────                   ──────────────────────────────
walkDocument → nodeIndex                    ExportProgressView
exportAssets():                               collects EXPORT_ASSET msgs
  a. image fills  → assets/{figmaHash}.png    uploadBundle():
  b. root frames  → renders/…, thumbnails/…     check-assets (CAS)  ─┐
  c. ≤50 VECTOR…  → svgs/{id}_{h8}.svg          presign → PUT manifest│
assembleManifest(assetManifest)                 PUT assets (5 conc.)  │
                                                complete-upload       │
                                                                      ▼
Backend
───────
pluginController   → plugin-uploads/{uploadId}/…  (CAS check reads assets/cas/, never written)
ingest worker      → copy to figma-snapshots/{snapshotId}/…, stampAssetRefs on nodes
snapshotLoader     → imageAssetMap = { ref → presigned GET URL (TTL 3600s) }
screenPackage      → per-screen assetManifest = presigned URLs  ──► LLM prompt
iconExtractor      → icons.tsx (first path only, stroke-only) / <img src="/assets/…">
generateScreen     → LLM writes TSX (inline <svg>, presigned URLs, /assets/… paths)
assembler          → download to public/assets/*, on miss write 1×1 transparent PNG / empty SVG
assemblePrototype  → vite build --base=/p/{projectId}/{jobId}/ → S3 preview
```

## 2. Defects by Stage

Severity: **P0** breaks assets in most prototypes · **P1** breaks many assets or hides failures · **P2** degrades quality or efficiency.

### 2.1 Plugin: discovery and export (`charisol-figma-plugin/src/exporter/assetExporter.ts`)

| # | Sev | Defect | Evidence |
|---|---|---|---|
| P-1 | P0 | **Icons are not detected.** Only nodes whose own type is `VECTOR`/`STAR`/`POLYGON`/`ELLIPSE`/`BOOLEAN_OPERATION` are SVG-exported. Real icons are usually `INSTANCE`/`COMPONENT`/`GROUP`/`FRAME` wrappers (e.g. `icon/arrow-right`) holding several vectors, so the icon is never exported as a unit. Its child vectors are exported as fragments, if at all. | `assetExporter.ts:207-209` |
| P-2 | P0 | **Hard cap of 50 vectors, in traversal order.** Every vector after the 50th is dropped silently. The cap counts icon fragments and ellipses used as decoration, so real icons on later screens are lost first. | `assetExporter.ts:209` |
| P-3 | P1 | **No dedupe by component.** 30 instances of the same icon component would be exported 30 times, if they were exported at all. Dedupe should key on `mainComponent.key`. | `assetExporter.ts:207-246` |
| P-4 | P1 | **Every image fill is labelled PNG.** `getBytesAsync()` returns the original bytes (often JPEG, sometimes GIF/WebP), but the key is `assets/{hash}.png` and the `Content-Type` is `image/png`. | `assetExporter.ts:91-98` |
| P-5 | P1 | **Only `fills` are scanned.** Image paints in `strokes` and `backgrounds` are ignored. Crop and filter information (`imageTransform`, `filters`) is dropped, so cropped or adjusted images render as the full original. | `assetExporter.ts:66-73`, `walker/nodeSerializer.ts:61-67` |
| P-6 | P1 | **Designer intent is ignored.** Nodes with Figma `exportSettings` (the designer's own "this is an asset" signal) are not read. | `walker/nodeSerializer.ts` (no `exportSettings`) |
| P-7 | P1 | **Silent SVG failures.** `catch (_) {}` swallows every SVG export error with no warning, unlike image fills. | `assetExporter.ts:245` |
| P-8 | P2 | **No vector geometry is serialized** (`vectorPaths`, `fillGeometry`). The backend fallback `extractPathData()` therefore never finds a path. | `walker/nodeSerializer.ts`, `iconExtractor.js:129-140` |
| P-9 | P2 | **Inconsistent hash identity.** Image fills use the Figma image hash. Renders and SVGs use `computeHash()`, which in the sandbox (no Web Crypto) falls back to a 24-hex FNV hash rather than SHA-256. CAS dedupe across exports is therefore unreliable. | `assetExporter.ts:31-53` |
| P-10 | P2 | **No intrinsic dimensions** (width/height) or human-readable name recorded per asset, so downstream code cannot size `<img>` or produce meaningful names. | `AssetEntry` type |

### 2.2 Plugin: upload (`charisol-figma-plugin/src/uploader/uploadOrchestrator.ts`)

| # | Sev | Defect | Evidence |
|---|---|---|---|
| U-1 | P1 | **Upload failures are silent.** A non-OK `PUT` response is not recorded, the loop retries without logging, and nothing is reported to the user or the backend. The export is shown as "successful" with missing binaries. | `uploadOrchestrator.ts:59-80` |
| U-2 | P1 | **The manifest is uploaded before the assets**, so it cannot record which assets actually made it. `complete-upload` is not told about failures. | `uploadOrchestrator.ts:184-229` |
| U-3 | P2 | **The dedupe result is ignored.** `assetsToUpload` is computed, but `uploadAssetsWithConcurrency(assets, …)` iterates *all* assets. Deduped ones are skipped only because they have no presigned URL, and progress counts are wrong. | `uploadOrchestrator.ts:222` |
| U-4 | P2 | **The CAS check is dead code.** `check-assets` probes `projects/{p}/assets/cas/{hash}`, but nothing in the backend writes to that prefix. If anything ever does, deduped assets would be absent from `plugin-uploads/{uploadId}/` and ingest would lose them. | `pluginController.js:319`, no writer found |

### 2.3 Backend: ingest (`charisol-design-system-be/workers/ingest`)

| # | Sev | Defect | Evidence |
|---|---|---|---|
| I-1 | P1 | **No asset verification.** `copyManifestAssets` counts `missing` but ingest does not fail, warn or persist a per-asset list. | `lib/copier.js:61-110` |
| I-2 | P2 | **No canonical asset index.** Consumers (`snapshotLoader`, `screenPackage`, `iconExtractor`, `assembler`) each re-derive keys with their own fallbacks (`assets/{hash}.png`, `svgs/{id}.svg`, `{nodeId}.png`, …). This is the source of most path mismatches. | `snapshotLoader.js:308-365`, `screenPackage.js:326-360`, `assembler.js:757-812` |

### 2.4 Backend: generation (`charisol-design-system-be/workers/generate`)

| # | Sev | Defect | Evidence |
|---|---|---|---|
| G-1 | **P0** | **Presigned URLs go into the prompt.** `buildImageAssetMap` presigns every ref (`DEFAULT_PRESIGNED_GET_URL_TTL_SECONDS = 3600`). `screenPackage` copies these into the per-screen `assetManifest`, which the prompt tells the model to use as `src`. The generated TSX embeds URLs that **expire in 1 hour**. The validator explicitly allows any `https://` src. | `snapshotLoader.js:353-361`, `screenPackage.js:339`, `prompts/user-template.md:17-20`, `validator.js:197-204` |
| G-2 | **P0** | **Base path mismatch.** Previews are built with `--base=/p/{projectId}/{jobId}/`. Vite does not rewrite string literals, so `src="/assets/x.png"` in JSX and in `icons.tsx` resolves to the domain root, not the preview folder. | `lambda/assemblePrototype.js:180,287`, `iconExtractor.js:42-46,108-112` |
| G-3 | P0 | **The prompt encourages invented icons.** "If an icon is in the node tree, render clean inline `<svg>` elements" / "or render a clean inline `<svg>` matching the design". With almost no real icons in `icons.tsx` (see P-1/P-2/P-8), the model hand-draws approximations. | `prompts/system.md:78`, `prompts/user-template.md:44,56` |
| G-4 | P1 | **`icons.tsx` is geometrically wrong.** Only `vectorPaths[0]` is used (multi-path icons are truncated). Everything is rendered `fill="none" stroke="currentColor"`, so filled icons become outlines. `viewBox` is taken from node bounds rather than the path's coordinate space. | `iconExtractor.js:96-105,129-140` |
| G-5 | P1 | **Icon names are meaningless.** Names come from the node name, which for exported vectors is usually `Vector`, giving `IconVector`, `IconVector2`, … The model cannot map them to design intent. | `iconExtractor.js:38,85-92` |
| G-6 | P1 | **Icon subtrees still go to the LLM.** The model receives raw vector children with no "this subtree is icon X" annotation. This wastes tokens and invites re-drawing. | `screenPackage.js:213` |
| G-7 | P2 | **Format is guessed from the extension.** `isSvg ? "svg" : "png"`, so JPEG and GIF bytes are written as `.png`. | `snapshotLoader.js:288`, `assembler.js:759-760` |
| G-8 | P2 | **Vite output collision.** Figma assets are written to `public/assets/`, the same folder Vite emits hashed JS/CSS bundles into (`dist/assets/`). This is confusing and risks name clashes. | `assembler.js:489` |

### 2.5 Backend: assembly and validation

| # | Sev | Defect | Evidence |
|---|---|---|---|
| A-1 | P1 | **Invisible fallback.** When an asset cannot be fetched, the assembler writes a transparent 1×1 PNG or an empty `<svg>`. The preview looks "fine" and nobody notices the missing asset. Nothing is recorded in `prototype-status.json`. | `assembler.js:824-845` |
| A-2 | P1 | **Unvalidated references.** The validator only warns on unknown `<img src>`. It ignores CSS `background-image: url(...)`, JSX expressions (`src={...}`) and unused-but-expected assets, and it never checks that the files exist in `public/`. | `validator.js:193-207` |
| A-3 | P2 | **No coverage metric.** There is no measure of "assets in the Figma screen" vs "assets referenced by the generated screen", so regressions are invisible. | n/a |

## 3. Impact Ranking

1. **G-1 + G-2**: these break images that *were* exported correctly. They are backend-only fixes with the largest immediate gain.
2. **P-1 / P-2 / P-8 + G-3 / G-4 / G-5**: these cause icons to be missing or invented. Fixing them needs the plugin and generator changes together.
3. **U-1 / I-1 / A-1 / A-2**: these hide failures. Fix them early so later phases can be measured.
4. **P-4 / P-5 / G-7 / P-9 / U-3 / U-4**: correctness and efficiency.
