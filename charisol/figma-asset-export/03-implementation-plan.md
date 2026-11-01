# 03 — Implementation Plan

> **Update (22 Oct 2026):** the "Hi-fi" incident (`04-incident-hifi-landing-page.md`) showed that asset fixes alone cannot fix wrong output. The screen-fidelity Phases S0–S5 in doc 04 come **first**. Note also that `e261594` widened G-1: it presigns svg, render and thumbnail refs into the prompt. Phase 1.1 below must undo that, and must exclude `renderRef`/`thumbRef` from the per-screen manifest.

## Roadmap

```
┌──────────────────────────────────────────────────────────────────────────┐
│ Phase 0  Baseline & instrumentation                 (BE only)            │
│ Phase 1  Backend quick wins: no URLs in prompt,     (BE only, ships first)│
│          base-path-safe registry, loud failures                          │
│ Phase 2  Plugin export fidelity: classify, dedupe,  (plugin release)     │
│          real formats, SHA-256, upload reporting                         │
│ Phase 3  Ingest: verification, asset-index.json, CAS (BE)                │
│ Phase 4  Generation: real-SVG icons.tsx, annotated  (BE)                 │
│          node tree, prompt rules                                         │
│ Phase 5  Validation, repair, status & UI surfacing  (BE + FE)            │
│ Phase 6  End-to-end verification & rollout                               │
└──────────────────────────────────────────────────────────────────────────┘
```

Phases 0–1 fix G-1/G-2/A-1 for **existing** snapshots without a plugin release. Phases 2–4 must ship together behind the flag `ASSET_PIPELINE_V2`, with a legacy path kept for old snapshots.

---

## Phase 0: Baseline & Instrumentation

**Goal:** measure asset loss before changing anything.

### 0.1 `scripts/asset-audit.js` (new, BE)
For a given `snapshotId` (or a fixture in `fixtures/figma/F1…F6`):
- Count the image fills, icon-like nodes (using the classification rules in `02 §2`) and manifest assets.
- Count the objects actually present in S3 under the snapshot prefix.
- For a generated job: count `<img>`/`url()`/`<Icon…>` usages in `src/screens/*.tsx`, presigned URLs found, and fallback files written (transparent PNG or empty SVG, detected by byte equality).
- Output a JSON and markdown table. Record the baseline in `docs/asset-baseline.md`.

### 0.2 Assembler counters
`workers/generate/assembler.js:757-845`: count `fetched`, `fallback`, and `failedKeys[]`, and log `generation_assets_written` as a structured event. Do not change behaviour yet.

**Exit:** a baseline table for F1–F6 and at least two real projects.

---

## Phase 1: Backend Quick Wins (works with existing snapshots)

### 1.1 Stop sending URLs to the LLM (fixes G-1)
- `workers/generate/snapshotLoader.js:308-365`: keep `buildImageAssetMap` but rename its output `assetDownloadMap`. It is used **only** by the assembler for fetching.
- New `workers/generate/assetResolver.js`:
  - `resolveAssets(snapshot) → { entries: Map<assetId, {kind, format, sourceKeys[], localPath, width, height, nodeIds}> , byRef: Map<imageRef|svgRef|nodeId, assetId> }`.
  - Legacy mode: derive `assetId` from the existing refs (`image_${figmaHash.slice(0,8)}`, `icon_${slug(nodeName)}_${hash6}`).
  - Assign `localPath = figma/{kind}/{slug}-{hash6}.{ext}`.
- `workers/generate/screenPackage.js:326-360`: build the per-screen `assetManifest` as `{ assetId: { kind, w, h } }`. **Never include URLs.**
- `prompts/user-template.md:17-20`: change "Use these asset URLs" to "Use these asset IDs via `assets.<id>`".

### 1.2 Base-path-safe asset registry (fixes G-2, G-8)
- `assetResolver.emitRegistry(entries) → string` writes `src/generated/assets.ts` (shape in `02 §5.1`).
- `assembler.js`: write the registry file. Write binaries to `public/figma/...` instead of `public/assets/...`.
- `iconExtractor.js:108-112`: `<img>` fallbacks use `assets.<id>`, not `/assets/…`.
- `templates/prototype/src/generated/.gitkeep` and a tsconfig path, if needed.

### 1.3 Detect real image format (fixes G-7, works around P-4)
- `utils/mime.js` (new): `sniffImageType(buffer)` returns `png | jpg | gif | webp | svg` from magic bytes (`89 50 4E 47`, `FF D8 FF`, `47 49 46 38`, `RIFF....WEBP`, `<svg`/`<?xml`).
- The assembler writes the file with the sniffed extension. If the sniffed type differs from the registry, update `localPath` before emitting the registry. To make that possible, run asset download *before* registry emission.

### 1.4 Loud, visible failures (fixes A-1)
- `assembler.js:824-845`: replace the transparent PNG and empty SVG with `placeholderSvg(name, w, h)`, a hatched box showing the asset name. Write it with a `.svg` extension and point the registry at it.
- Collect `missing[]` and add the `assets` block to `prototype-status.json` (`02 §8`) in `lambda/assemblePrototype.js`.

### 1.5 Prompt rule changes (interim fix for G-3)
- `prompts/system.md:63-78` and `user-template.md:44,56`: remove "render a clean inline `<svg>`" as the default. New text: use `./icons` or `assets.<id>`. Inline SVG is allowed only for simple geometric decoration with no ref.

### 1.6 Tests (BE, `tests/`)
- `assetResolver.test.js`: legacy refs → IDs, stable naming, no URLs in the per-screen manifest.
- `assembler.assets.test.js`: registry emitted; files land under `public/figma`; a missing asset produces a placeholder and is listed in `missing[]`.
- `mime.test.js`: magic bytes for each format.
- `prompts.test.js`: the rendered prompt contains no `amazonaws.com` and no `X-Amz-`.
- Build test: `vite build --base=/p/x/y/` on a fixture, then assert that the emitted JS contains `/p/x/y/figma/` paths.

**Exit:** for existing snapshots, zero presigned URLs in generated TSX, images resolve under the preview base path, and `prototype-status.json.assets` is populated.

---

## Phase 2: Plugin Export Fidelity (plugin release)

### 2.1 Serializer additions (`src/walker/nodeSerializer.ts`)
- Serialize `exportSettings` (format, suffix, constraint).
- `serializePaints`: include `imageTransform`, `filters` and `scaleMode` for IMAGE paints. Apply it to `strokes` and `backgrounds` too, not just `fills`.
- For `INSTANCE`: record `mainComponent.key` and `mainComponent.name` (the parent `COMPONENT_SET` name plus variant props, if present).

### 2.2 `src/exporter/assetClassifier.ts` (new)
- `classifyAssets(nodeIndex, parentMap, opts) → ClassifiedAsset[]` implements the rule table in `02 §2`, including **covered-descendant suppression**.
- Pure function with no Figma API calls. Unit-test it against serialized fixtures.
- Constants live in `src/exporter/assetConfig.ts`: `ICON_MAX_PX=64`, `MAX_ICONS=500`, `SVG_MAX_BYTES=150_000`, `MAX_ASSET_BYTES=100_000_000`.

### 2.3 Rewrite `src/exporter/assetExporter.ts`
- Iterate the classified assets in priority order. Dedupe icons by `componentKey` (export one, then attach every `nodeId`) and images by `figmaImageHash`.
- Icons and illustrations: `exportAsync({ format:'SVG', svgIdAttribute:false, svgOutlineText:true, svgSimplifyStroke:true })`. Compute `monochrome` by parsing the distinct `fill=`/`stroke=` colours in the SVG string (ignoring `none`).
- Images: `getBytesAsync()`, then `sniffImageType` (a shared port of `utils/mime.js`), then the real extension and MIME. Read the dimensions with `image.getSizeAsync()`.
- `nodeRender` for cropped or filtered images: `exportAsync({ format:'PNG', constraint:{ type:'SCALE', value:2 } })`.
- **Remove the `.slice(0, 50)` cap.** Enforce the budgets and emit `ASSET_BUDGET_EXCEEDED` with the skipped `nodeIds`.
- Replace every `catch (_) {}` with a warning push (`SVG_EXPORT_FAILED`, `ASSET_EXPORT_FAILED`).
- Stamp `node.assetRef` and `node.assetRole` on the serialized nodes. Stamp `fill.assetRef` on image paints.
- Keep emitting the legacy `assetManifest` entries for one release.

### 2.4 SHA-256 in the UI iframe (fixes P-9)
- The sandbox has no Web Crypto. Move hashing into `ExportProgressView` (iframe): on `EXPORT_ASSET`, compute `crypto.subtle.digest('SHA-256', data)`. Set `sha256`, then derive `assetId` and `s3Key = assets/{kind}/{assetId}.{ext}`.
- Hashing is deferred to the UI, so the sandbox sends a provisional `assetKey`. The UI patches the manifest's `assets[]` before upload. Alternatively, keep the FNV hash for IDs and send SHA-256 only for CAS; choose one during implementation and document it in `plugin-shared-schema-contract.md`.

### 2.5 Upload reporting (`src/uploader/uploadOrchestrator.ts`, fixes U-1/U-2/U-3)
- Use `assetsToUpload` in `uploadAssetsWithConcurrency` (line 222).
- Track `res.ok` failures and retry with exponential backoff (3 attempts). Return `failed: {assetId, status, error}[]`.
- **Order:** presign → upload assets → write `uploadReport` (failed assets) into the manifest → upload the manifest → `complete-upload { uploadId, failedAssets }`.
- UI: show "N assets failed to upload" with a retry button. Do not show plain "success".

### 2.6 Contract
- Backend `contracts/figma-snapshot`: add `assets[]`, `uploadReport` and the new warning codes. Run `npm run contracts:sync` to regenerate `src/types/snapshot.v2.ts`.
- Update `docsplan/plugin-shared-schema-contract.md`.

### 2.7 Plugin tests (`tests/*.test.ts`)
- `assetClassifier.test.ts` cases:
  - An icon instance with 5 vectors gives 1 icon, with descendants covered.
  - 30 instances of one component give 1 asset with 30 `nodeIds`.
  - A logo with text gives an icon (outlined).
  - A large vector illustration gives an illustration.
  - An image with CROP gives an image plus a nodeRender.
  - `exportSettings` overrides the heuristics.
  - The budget cap emits a warning.
- `assetExporter.test.ts` (with the Figma API mocked): JPEG bytes produce `.jpg` and `image/jpeg`; SVG failures produce a warning.
- `pipeline.test.ts`: the manifest carries `assets[]` and the legacy `assetManifest`. Upload failures surface in the result.

**Exit:** on the F1–F6 fixture files, every visible icon component is exported exactly once, with no `IconVector*` names and no silent drops.

---

## Phase 3: Ingest Verification, Asset Index & CAS

### 3.1 `workers/ingest/lib/assetIndexer.js` (new)
- Input: `manifest.assets[]` (v2) or, for legacy, `assetManifest` plus nodes.
- For each asset: `HeadObject` the upload key. Copy it to `figma-snapshots/{id}/assets/{kind}/…` **and** `projects/{p}/assets/cas/{sha256}.{ext}` (skipping the copy if CAS already has it). Sniff the MIME from the first 16 bytes using a ranged GET.
- Write `asset-index.json` (`02 §4`). Add `ingest-report.assets = { expected, present, missing[] }`.
- Reuse the candidate-key fallbacks from `copier.js:copyManifestAssets` for the legacy path, then retire the duplicate logic.

### 3.2 Make CAS real (fixes U-4)
- `pluginController.checkExistingAssets` (`:298-340`): check `assets/cas/{sha256}.{ext}`.
- `assetIndexer`: when an asset is absent from the upload prefix but present in CAS (because the plugin skipped it after the dedupe check), resolve it from CAS. The snapshot must not lose assets the plugin deduped.

### 3.3 Status
- Missing assets do **not** fail ingest. Above 0 missing, set the snapshot `warningsCount` and the `ASSET_MISSING` warning. The frontend shows it on the inventory card.

### 3.4 Tests
- `assetIndexer.test.js` cases:
  - A v2 manifest gives the correct `byId`/`byNodeId`/`byFigmaImageHash`.
  - A legacy manifest gives an equivalent index.
  - An asset missing from the upload but present in CAS gives `present:true`.
  - An asset missing everywhere is listed in `missing[]`.
  - A JPEG uploaded as `.png` is sniffed and gets the correct MIME.
- Extend `rederive.js` with an assets-only mode, so old snapshots can be re-indexed without a re-export. Add a CLI script entry for it.

---

## Phase 4: Generation, Icons and Prompt Annotations

### 4.1 `snapshotLoader.js` / `assetResolver.js`
- Load `asset-index.json` when it exists. Otherwise fall back to the Phase 1 legacy derivation. The resolver is the **only** consumer of asset keys. Delete the ad-hoc fallbacks in `screenPackage.js:326-360` and `assembler.js:791-812`.

### 4.2 Rewrite `iconExtractor.js` (fixes G-4/G-5)
- Add dependency: `svgo` (BE `package.json`).
- Input: the resolver's `icon` entries plus their SVG bytes (downloaded once in `prepareScreenPackages`).
- Pipeline: SVGO, then `currentColor` for monochrome icons, then the SVG→JSX attribute map, then the component (`02 §5.2`). Name from the component name, with variant-aware collision resolution.
- Output: `{ iconsTsx, iconNames, iconMap: { assetId → ComponentName } }`.
- Keep the legacy geometry path only as a last resort, and fix it: use **all** `vectorPaths`, honour `windingRule`, use `fill="currentColor"` for filled paths, and use the path bounding box for `viewBox`.
- `icons.tsx` size guard: above 400 KB, split it into `icons/{Name}.tsx` with an `index.ts` barrel.

### 4.3 `screenPackage.js`: annotate and prune (fixes G-6)
- In the pruning pass (`:200-220`), for nodes with `assetRole: icon` (or `byNodeId` → icon): replace the subtree with `{ type:'ICON', icon: iconMap[assetId], w, h, color }`. Take `color` from the first descendant's solid fill token.
- For image paints: add `image: assetId` and drop `imageRef`.
- Add `data-node-id` guidance so the validator can trace nodes to output.
- Record `expectedRefs: { icons:[], images:[] }` in the package for coverage checks.

### 4.4 Prompt (`prompts/system.md`, `user-template.md`, `prompts/index.js`)
- Apply the rules in `02 §6`. Replace the example in `system.md:177-206` (`imageUrl = "/assets/hash_cream_01.png"`) with a registry-based example.
- `{{ASSET_MANIFEST}}` lists `assetId (w×h, kind)`. `{{AVAILABLE_ICONS}}` lists only icons used on *this* screen first, then the remainder, truncated.
- Keep the prompt cache layout intact: the shared icon list stays in the cached block and the per-screen refs go in the uncached block.

### 4.5 Tests
- `iconExtractor.test.js` cases:
  - A multi-path icon keeps all its paths.
  - A filled icon stays filled.
  - A monochrome icon is converted to `currentColor`; a multicolour icon keeps its colours.
  - `icon/arrow-right` is named `IconArrowRight`.
  - `xlink:href` and `stroke-width` are converted to JSX.
  - The output compiles under `tsc`.
- `screenPackage.test.js`: icon subtrees are pruned, the ref annotations are present, and the token count drops on fixture F4.
- Snapshot tests of the prompt for F1/F2.

---

## Phase 5: Validation, Repair, Status & UI

### 5.1 `validator.js`
- Implement the rules in `02 §7`. Parse `src=` and `src={…}`, `url(...)` in CSS modules and inline styles, and `backgroundImage`.
- Add a coverage calculation against `expectedRefs`.

### 5.2 `buildFixer.js`
- Deterministic rewrites: S3/presigned/`/assets/` literal becomes `assets.<id>` (via the reverse index). An unknown `assets.x` becomes the nearest expected ID (Levenshtein, within the screen's refs only).
- When coverage is below `ASSET_COVERAGE_MIN`, run one LLM repair turn in `screenGenerator.js` that lists the missing refs and their node IDs. Then re-validate. If coverage is still low, mark the screen `degraded`.

### 5.3 Status & frontend
- Finalize `prototype-status.json.assets` (`02 §8`). Include coverage in the per-screen health.
- FE (`charisol-design-system-fe`): an asset health badge on the prototype detail page, plus a list of missing assets with node IDs (each a deep-link into Figma, `?node-id=`). Optional, but recommended.

### 5.4 Tests
- Validator cases: a presigned URL is an error and gets rewritten; a literal path is rewritten; an unknown ID is fixed; coverage is calculated correctly.
- Generator integration test with a mocked Bedrock response containing bad URLs: the final TSX is clean.

---

## Phase 6: End-to-End Verification & Rollout

### 6.1 Acceptance criteria (run `scripts/asset-audit.js` on F1–F6 and two real projects)

| Metric | Target |
|---|---|
| Presigned / `amazonaws.com` URLs in generated code | **0** |
| Visible icon components exported (unique) / present in Figma | **100 %** (within budget) |
| Image fills exported with the correct MIME | **100 %** |
| Assets present in the preview / referenced by the registry | **100 %**, otherwise listed in `missing[]` |
| Per-screen asset coverage (median / p10) | ≥ 0.98 / ≥ 0.90 |
| Images still loading 24 h after generation | **Yes** |
| `IconVector*` names in `icons.tsx` | **0** |
| Prompt tokens on F4 (icon-heavy) | ≤ baseline (expected to fall because subtrees are pruned) |

### 6.2 Manual checks
1. Export a file containing an icon library (instances of 20+ icons, multi-path, filled and outlined, multicolour logo), a JPEG hero image, a cropped avatar and a large SVG illustration.
2. Generate a prototype and open the preview at `/p/{projectId}/{jobId}/`. Every icon matches Figma in shape, fill and colour. Images are sharp and correctly cropped.
3. Reload the preview more than 1 hour later: no broken images.
4. Delete one asset from S3 and regenerate. A visible placeholder appears and `prototype-status.json.assets.missing` lists the asset.
5. Re-export the same file. CAS dedupe skips uploads, and ingest still produces a complete `asset-index.json`.

### 6.3 Rollout
1. Ship Phase 1 without a flag. It is backward compatible and fixes current breakage.
2. Ship Phases 3–4 behind `ASSET_PIPELINE_V2`, which is enabled automatically when the snapshot has `asset-index.json`.
3. Release the plugin (Phase 2) with the dual `assets[]` and `assetManifest` output.
4. After one release cycle, remove the legacy `assetManifest` emission. Keep the legacy resolver for old snapshots.

---

## File Change Summary

| Repo | File | Phase | Change |
|---|---|---|---|
| plugin | `src/walker/nodeSerializer.ts` | 2 | exportSettings, image paint transforms, component key/name |
| plugin | `src/exporter/assetClassifier.ts` | 2 | **new**: classification rules |
| plugin | `src/exporter/assetConfig.ts` | 2 | **new**: budgets and thresholds |
| plugin | `src/exporter/assetExporter.ts` | 2 | rewrite: dedupe, formats, no cap, warnings |
| plugin | `src/components/ExportProgressView.tsx` | 2 | SHA-256 hashing, failure UI |
| plugin | `src/uploader/uploadOrchestrator.ts` | 2 | upload order, failure tracking, `assetsToUpload` bug |
| plugin | `src/types/snapshot.v2.ts` | 2 | regenerated |
| be | `contracts/figma-snapshot/*` | 2 | `assets[]`, `uploadReport`, warning codes |
| be | `controllers/pluginController.js` | 3 | CAS key format, accept `failedAssets` |
| be | `workers/ingest/lib/assetIndexer.js` | 3 | **new**: verify, CAS, `asset-index.json` |
| be | `workers/ingest/index.js`, `lib/copier.js`, `lib/rederive.js` | 3 | wire in the indexer, retire duplicate logic |
| be | `utils/mime.js` | 1 | **new**: magic-byte sniffing |
| be | `workers/generate/assetResolver.js` | 1/4 | **new**: single resolver, registry emitter |
| be | `workers/generate/snapshotLoader.js` | 1/4 | download map only, load the asset index |
| be | `workers/generate/screenPackage.js` | 1/4 | ID-only manifest, annotate and prune |
| be | `workers/generate/iconExtractor.js` | 4 | rewrite on real SVG + SVGO |
| be | `workers/generate/prompts/*` | 1/4 | asset and icon rules, examples |
| be | `workers/generate/validator.js`, `buildFixer.js`, `screenGenerator.js` | 5 | URL rules, coverage, repair |
| be | `workers/generate/assembler.js`, `lambda/assemblePrototype.js`, `lambda/prepareScreenPackages.js` | 1/4 | `public/figma`, registry, placeholders, status |
| be | `templates/prototype/src/generated/` | 1 | registry location |
| be | `scripts/asset-audit.js` | 0 | **new**: audit and baseline |
| fe | prototype detail page | 5 | asset health badge (optional) |

## Open Questions

1. **CDN routing:** does CloudFront serve `/p/{projectId}/{jobId}/*` from the preview prefix? Confirm this before relying on `BASE_URL`. If previews are ever served at the root, the registry still works.
2. **Hashing location:** compute SHA-256 in the UI iframe (preferred) or keep the sandbox FNV hash? This affects the `assetId` stability guarantees.
3. **Icon threshold:** is `ICON_MAX_PX=64` right for this team's files? Some design systems use 80–96 px "spot" icons.
4. **Remote library icons:** instances of remote (team-library) components can be exported, but `mainComponent` may be unavailable offline. Fall back to the node name and SVG export in that case.
5. **Budget defaults:** are 500 icons and 100 MB per export acceptable for the largest known files (F4)?
