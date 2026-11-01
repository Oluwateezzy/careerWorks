# 02 — Target Architecture & Asset Contract

## 1. End-to-End Target Flow

```
┌───────────────────────────────────────────────────────────────────────────┐
│ PLUGIN                                                                    │
│  walk → classifyAssets() → exportAssets() → hash in UI iframe (SHA-256)   │
│  • icon / illustration / image / render / thumbnail                       │
│  • dedupe: icons by componentKey, images by sha256                        │
│  • manifest.assets[] v2 (+ per-node assetRef / iconRef)                   │
│  upload assets FIRST → record failures → manifest → complete-upload       │
└─────────────────────────────────────┬─────────────────────────────────────┘
                                      ▼
┌───────────────────────────────────────────────────────────────────────────┐
│ INGEST                                                                    │
│  HEAD every asset → copy to snapshot (and CAS) → sniff MIME from bytes    │
│  write figma-snapshots/{id}/asset-index.json  (single source of truth)    │
│  ingest-report.assets = { expected, present, missing[] }                  │
└─────────────────────────────────────┬─────────────────────────────────────┘
                                      ▼
┌───────────────────────────────────────────────────────────────────────────┐
│ PREPARE (deterministic, once per job)                                     │
│  assetResolver.js: asset-index → stable local paths                       │
│     public/figma/{kind}/{slug}-{hash8}.{ext}                              │
│  emits  src/generated/assets.ts   (typed registry, BASE_URL-aware)        │
│  emits  src/components/icons.tsx  (from real SVG via SVGO → JSX)          │
│  screenPackage: annotate nodes { iconRef | assetRef }, prune icon subtrees│
└─────────────────────────────────────┬─────────────────────────────────────┘
                                      ▼
┌───────────────────────────────────────────────────────────────────────────┐
│ GENERATE (LLM)                                                            │
│  Prompt lists asset IDs + icon component names (no URLs)                  │
│  Rule: node with iconRef → <IconX/>, node with assetRef → assets.<id>     │
└─────────────────────────────────────┬─────────────────────────────────────┘
                                      ▼
┌───────────────────────────────────────────────────────────────────────────┐
│ VALIDATE + REPAIR (deterministic)                                         │
│  reject S3/presigned/abs URLs · rewrite known ones to registry            │
│  coverage: expected refs vs used refs · tsc catches unknown asset IDs     │
└─────────────────────────────────────┬─────────────────────────────────────┘
                                      ▼
┌───────────────────────────────────────────────────────────────────────────┐
│ ASSEMBLE                                                                  │
│  copy files from asset-index → public/figma/… · missing = visible         │
│  placeholder + prototype-status.json.assets report                        │
│  vite build --base=/p/{projectId}/{jobId}/  (registry resolves correctly) │
└───────────────────────────────────────────────────────────────────────────┘
```

## 2. Asset Classification (plugin)

`classifyAssets(nodeIndex)` runs once over the walked tree and assigns each node at most one asset role. Evaluate the rules top-down. The first match wins, and the matched node's descendants are marked **covered**, so they are never exported separately.

| Order | Rule | Kind | Export |
|---|---|---|---|
| 1 | Node has non-empty `exportSettings` (designer intent) | per setting (`icon` if SVG, `image` otherwise) | Use the node's own `exportSettings` |
| 2 | `INSTANCE`/`COMPONENT` whose subtree is **vector-only*** and both sides ≤ `ICON_MAX_PX` (default 64) | `icon` | SVG, deduped by `mainComponent.key` |
| 3 | Name matches `/(^|[\/_\-\s])(icon|ic|glyph|logo)([\/_\-\s]|$)/i` and size ≤ `ICON_MAX_PX` | `icon` | SVG |
| 4 | `GROUP`/`FRAME`/`BOOLEAN_OPERATION` whose subtree is vector-only*, size ≤ `ICON_MAX_PX` (outermost such ancestor) | `icon` | SVG |
| 5 | Vector-only* subtree larger than `ICON_MAX_PX` | `illustration` | SVG if ≤ `SVG_MAX_BYTES` (default 150 KB), else PNG @2x |
| 6 | Any paint of type `IMAGE` in `fills`/`strokes`/`backgrounds` | `image` | Original bytes via `getImageByHash` |
| 6b | …and the paint has `scaleMode: 'CROP'`, non-identity `imageTransform`, or `filters` | `image` (+ `nodeRender`) | Also export the node as PNG @2x, so the crop and filters are baked in |
| 7 | Root frame | `render`, `thumbnail` | Unchanged from today |

\*Vector-only means every visible descendant is one of `VECTOR`, `BOOLEAN_OPERATION`, `STAR`, `POLYGON`, `ELLIPSE`, `LINE`, a `RECTANGLE` without an image fill, `GROUP`, or a `FRAME` used as a wrapper. There is no `TEXT` (unless `svgOutlineText` is enabled for logos) and no image fills.

Budgets replace the hard cap of 50: `MAX_ICONS` (default 500 unique), `MAX_ASSET_BYTES` (default 100 MB per export). Exceeding a budget emits an `ASSET_BUDGET_EXCEEDED` warning that lists the skipped `nodeIds`. The order of priority is: designer intent, then icons on selected or root screens, then images, then illustrations.

SVG export options: `{ format: 'SVG', svgIdAttribute: false, svgOutlineText: true, svgSimplifyStroke: true }`.

## 3. Manifest `assets[]` v2 Contract

This replaces `assetManifest`. The plugin still writes `assetManifest` for one release so older backends keep working.

```jsonc
{
  "assets": [
    {
      "assetId": "icon_arrow_right_9c1e2a",     // stable: `${kind}_${slug}_${sha256[0:6]}`
      "kind": "icon",                            // icon | illustration | image | nodeRender | render | thumbnail
      "sha256": "9c1e2a…",                       // ALWAYS SHA-256 of the bytes (computed in the UI iframe)
      "figmaImageHash": null,                    // set for kind=image
      "componentKey": "a1b2…",                   // set for component-backed icons
      "name": "icon/arrow-right",                // source name (component name preferred)
      "format": "svg",                           // svg | png | jpg | gif | webp (sniffed from bytes)
      "mime": "image/svg+xml",
      "width": 24, "height": 24,                 // intrinsic px (node size for svg, pixel size for raster)
      "monochrome": true,                        // svg: single paint colour → safe to map to currentColor
      "s3Key": "assets/icon/icon_arrow_right_9c1e2a.svg",
      "sizeBytes": 412,
      "nodeIds": ["12:34", "56:78"]              // every node that uses this asset
    }
  ]
}
```

Per-node fields added by the serializer or classifier:

```jsonc
{ "id": "12:34", "type": "INSTANCE", "assetRef": "icon_arrow_right_9c1e2a", "assetRole": "icon" }
{ "id": "90:12", "type": "RECTANGLE", "fills": [{ "type": "IMAGE", "imageRef": "<figmaHash>", "assetRef": "image_hero_3fa2c1", "scaleMode": "FILL" }] }
```

The JSON schema in `contracts/` (`snapshot.v2`) gains `files[].kind = "assets"` entries, `assets[]`, and warning codes `ASSET_BUDGET_EXCEEDED`, `ASSET_UPLOAD_FAILED`, `SVG_EXPORT_FAILED`. Regenerate the types with `npm run contracts:sync` in `charisol-design-system-be`.

## 4. Snapshot `asset-index.json` (ingest output)

Ingest writes this once. It is the **only** structure that downstream code reads to find asset binaries.

```jsonc
{
  "version": 1,
  "byId": {
    "icon_arrow_right_9c1e2a": {
      "kind": "icon", "format": "svg", "mime": "image/svg+xml",
      "storageKey": "projects/{p}/figma-snapshots/{s}/assets/icon/icon_arrow_right_9c1e2a.svg",
      "casKey": "projects/{p}/assets/cas/9c1e2a….svg",
      "width": 24, "height": 24, "monochrome": true,
      "name": "icon/arrow-right", "nodeIds": ["12:34", "56:78"], "present": true
    }
  },
  "byNodeId": { "12:34": "icon_arrow_right_9c1e2a" },
  "byFigmaImageHash": { "<figmaHash>": "image_hero_3fa2c1" },
  "missing": []
}
```

**Legacy snapshots** (no `assets[]`): `assetIndexLegacy.js` builds the same structure from `assetManifest` plus `stampAssetRefs`, reusing the existing candidate-key logic in `copier.js`. All lookups then go through one code path.

**As built (Phase 3, `workers/ingest/lib/assetIndexer.js`):**

- Legacy and v2 manifests both go through `buildAssetIndex`. There is no separate `assetIndexLegacy.js`, and `copier.js:copyManifestAssets` has been removed. `index.source` is `"v2"` or `"legacy"`.
- Uploaded bytes are read in full, not with a 16-byte ranged GET. MIME is sniffed from the bytes, and **SHA-256 is recomputed server-side**. The CAS key always uses the verified hash. A wrong client hash sets `hashMismatch: true` and is never written to CAS.
- CAS objects are `projects/{p}/assets/cas/{sha256}.{ext}`. `check-assets` still accepts bare hashes and looks them up by the `{sha256}.` prefix.
- Snapshot copies keep the manifest `s3Key` as the destination, so existing `svgRef`/`renderRef` lookups keep working until Phase 4 switches to the index.
- `render`/`thumbnail` assets are kept out of `byNodeId` and go in `rendersByNodeId: { [nodeId]: { render?, thumbnail? } }`. When several assets share a node, a present asset beats a missing one, and `icon`/`illustration`/`nodeRender` beat `image`.
- `missing[]` entries carry `reason`: `upload_failed` (listed in `manifest.uploadReport.failedAssets`), `not_uploaded`, or `copy_failed`. Each entry also adds a digest `ASSET_MISSING` warning, so the snapshot's `warningsCount` reflects it.
- Old snapshots: `yarn assets:reindex --project <p> --snapshot <s> --upload <u>`.

## 5. Generated Prototype Artefacts

### 5.1 `src/generated/assets.ts` (typed registry)

```ts
// AUTO-GENERATED by assetResolver — do not edit
const base = import.meta.env.BASE_URL; // honours vite --base=/p/{projectId}/{jobId}/
export const assets = {
  image_hero_3fa2c1: `${base}figma/image/hero-3fa2c1.jpg`,
  illustration_empty_state_77d0aa: `${base}figma/illustration/empty-state-77d0aa.svg`,
} as const;
export type AssetId = keyof typeof assets;
```

- Files live under `public/figma/{kind}/`, separate from Vite's own `assets/` output.
- An unknown key is a **TypeScript error**, so `tsc`/`vite build` and `buildFixer` catch it deterministically.

### 5.2 `src/components/icons.tsx` (from real SVG)

`iconExtractor.js` is rewritten to work from the exported SVG bytes:

1. Optimize the SVG with SVGO (`preset-default`, keep `viewBox`, `removeDimensions`, `prefixIds`).
2. If the icon is `monochrome`, replace its single fill or stroke colour with `currentColor`. Multicolour icons keep their colours.
3. Convert the SVG to JSX (attribute camelCasing: `stroke-width` → `strokeWidth`, `class` → `className`, `xlink:href` → `href`). Spread `{...props}` on the root and keep the original `viewBox`. Use the plugin's `width`/`height` as default props.
4. Name the component from `componentKey` → component name → node name, using `sanitizeIconName`: `icon/arrow-right` → `IconArrowRight`, `Icons / 24 / Chevron Down` → `IconChevronDown`. Strip size tokens (`16`, `24`, `sm`). Resolve collisions with a variant suffix (`IconChevronDownFilled`) rather than `2`, `3`.
5. Fallback when the SVG is missing or invalid: `export const IconX = (p) => <img src={assets.icon_x_…} … />`. Never emit an empty `<svg>`.

Also emit an `iconMap` (`assetId → ComponentName`) for prompt annotation and validation.

**As built (Phase 4, `workers/generate/assetResolver.js`, `iconExtractor.buildIconSet`, `assembler.js`):**

- `ASSET_PIPELINE_V2` is **on by default**. Set `ASSET_PIPELINE_V2=false` to get the legacy URL path back. `snapshotLoader` reads `asset-index.json`, or derives the same index from the manifest for older snapshots (`assetIndexSource: "derived"`).
- `assets.ts` lists only the IDs that the generated TSX (and shared components) reference, plus icons that fell back to `<img>`. Only those binaries are copied, so a hallucinated ID is a tsc error and unused assets cost nothing. An empty registry omits `base`, to satisfy `noUnusedLocals`.
- The assembler sniffs the downloaded bytes. A JPEG declared as PNG is written as `.jpg`, and the registry follows. Missing binaries become a placeholder `.svg` with `reason: NOT_IN_S3` (fetch failed) or `NOT_IN_SNAPSHOT` (index says `present:false`).
- Icon naming drops the icon and size words, and collisions use the variant (`IconChevronDownFilled`) before a number. Vectors under an exported icon are never turned into geometry icons. The legacy `extractSharedIcons`/`sanitizeIconName` path is unchanged.
- Above 400 KB, icons are written as `src/components/icons/{Name}.tsx` plus `index.ts`. `./icons` resolves either way.

## 6. Prompt Contract

`screenPackage.js` changes the pruned node tree that is sent to the model:

```jsonc
// before: 9 vector children, ~1.2k tokens
// after:
{ "id": "12:34", "type": "ICON", "icon": "IconArrowRight", "w": 24, "h": 24, "color": "--color-text-primary" }
{ "id": "90:12", "type": "RECTANGLE", "image": "image_hero_3fa2c1", "scaleMode": "FILL", "w": 320, "h": 180 }
```

- **Icon subtrees are pruned** and replaced by a single `ICON` node, which saves tokens.
- `{{ASSET_MANIFEST}}` becomes a list of `assetId` and dimensions. It contains **no URLs**.
- System-prompt rules (these replace `system.md:63-78` and `user-template.md:44,56`):
  - A node with `icon` **must** render `<IconName />` imported from `./icons`. Set its colour with CSS `color` and its size with `width`/`height` props.
  - A node with `image` **must** use `assets.<id>` (`import { assets } from '../generated/assets'`), as either `<img src={assets.x}>` or `style={{ backgroundImage: \`url(${assets.x})\` }}`. Map `scaleMode` to `object-fit`: FILL → cover, FIT → contain, TILE → background-repeat, CROP → already baked in.
  - **Never** write a URL, a `/assets/...` literal, or a hand-drawn `<svg>` for a node that has an `icon` or `image` ref. Inline `<svg>` is allowed only for simple shapes that have no ref (dividers, dots).

The vision render (`screenRender.js`) keeps using a presigned URL. It is model *input*, never written into code.

**As built (Phase 4):** the Bedrock call has no prompt caching, so the shared icon list is part of the per-screen user message. It lists this screen's icons first, then the rest, capped at 150 names. `data-node-id` is a system-prompt instruction; checking it is Phase 5. `coverageByScreen` is not in `prototype-status.json` yet (Phase 5).

## 7. Validation & Repair Rules

| Rule | Severity | Auto-repair |
|---|---|---|
| `src`/`url()` contains `amazonaws.com`, `X-Amz-`, or `https://` pointing to our bucket | error | Map the URL back to its `assetId` using the reverse index, then rewrite to `assets.<id>` |
| A literal `"/assets/…"` or `"/figma/…"` path | error | Rewrite to `assets.<id>` if it resolves, otherwise error |
| `assets.<unknown>` | error (tsc) | Use `buildFixer` to pick the nearest ID among the screen's expected refs |
| An expected `icon`/`image` ref is not used in the TSX | warning, counted in coverage | — |
| Inline `<svg><path …>` inside a node that has an `iconRef` | warning | Replace it with `<IconX/>` when the node ID is traceable via a `data-node-id` attribute |

`coverage = usedRefs / expectedRefs` is computed per screen. Below `ASSET_COVERAGE_MIN` (default 0.9), the screen is marked `degraded` and one targeted repair turn is triggered. The repair prompt lists only the missing refs.

**As built (Phase 5, `validator.validateAssetUsage`, `buildFixer.repairAssetRefs`, `screenGenerator.finalizeAssets`):**
- The reverse index is a per-screen `assetAliases` map (S3 key, file name, stem, `localPath`, Figma image hash → `assetId`) built in `screenPackage.js`, alongside `refNodes` (ref → node IDs) for the repair prompt. Neither is sent to the model.
- CSS modules cannot import the registry, so they use `url(asset://<id>)`. The assembler swaps the marker for the copied file's public path, and Vite adds the base.
- Deterministic repair runs before validation. The repair turn runs when errors remain or coverage is below `ASSET_COVERAGE_MIN`, and its output is kept only if it has fewer errors or higher coverage. An unresolvable literal stays in the code and is reported as an error; the screen is marked `degraded` with `degradationReason: "asset_coverage_low"`.
- Non-bucket `https://` image sources are a warning (`external_url`), not an error. The inline-`<svg>` rule is a warning only; there is no auto-replace.

## 8. Assembly & Status Reporting

- Copy each `asset-index.byId` entry that the registry references into `public/figma/{kind}/…`, using its `storageKey` (falling back to `casKey`).
- **Missing binary**: write a *visible* placeholder (an SVG with a hatched box and the asset name), and set `present: false`.
- `prototype-status.json` gains:

```jsonc
"assets": {
  "expected": 42, "written": 41, "missing": [{ "assetId": "image_hero_3fa2c1", "reason": "NOT_IN_S3" }],
  "icons": { "total": 18, "fromSvg": 17, "imgFallback": 1 },
  "coverageByScreen": { "home": 1.0, "settings": 0.86 }
}
```

The frontend can surface this as an "Assets" health badge on the prototype page. This is optional and tracked in the implementation plan.

**As built (Phase 5):** `missing[]` rows also carry `name`, `kind` and `nodeIds`. The block adds `coverage` (the mean of the screen ratios), and each screen entry in `prototype-status.json` gets an `assets` health object. The job's `warningSummary.assets` holds a summary (`lambda/assemblePrototype.summarizeAssetHealth`): the first 50 missing rows, screens sorted lowest coverage first, and `figmaFileKey`. The FE (`prototype/AssetHealth.tsx`) shows it as a badge on the preview, with a panel that deep-links each missing node to Figma.
