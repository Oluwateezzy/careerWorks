# Figma Asset, Image & Icon Fidelity Plan

## Goal

Every image, icon and illustration that a designer places in Figma must appear in the generated prototype:

- **Exported** by the plugin in the right format.
- **Stored and verified** by ingest.
- **Referenced** by the generated screen code through a stable, local, base-path-safe URL.
- **Present** in the deployed preview. No expiring links, no invisible placeholders, no hand-drawn substitute icons.

## Directory Contents

| Document | Focus |
| :--- | :--- |
| [01-gap-analysis.md](./01-gap-analysis.md) | Traces assets end to end (plugin → upload → ingest → generate → assemble → preview). Lists every defect found, with file and line references. |
| [02-target-architecture.md](./02-target-architecture.md) | Target asset contract: asset classification, the manifest `assets[]` v2 schema, the snapshot `asset-index.json`, the generated `assets.ts` registry and `icons.tsx`, and prompt annotations. |
| [03-implementation-plan.md](./03-implementation-plan.md) | Phased, file-by-file implementation steps, tests and acceptance criteria. Backend-only quick wins come first, so improvements ship before a plugin release. |
| [04-incident-hifi-landing-page.md](./04-incident-hifi-landing-page.md) | **Start here.** Root-cause analysis of the "Hi-fi" job (`job_gen_1792659872396_muzw0`). Wrong layout, text, images and fonts come from screen-level causes (no screenshot sent, board not unwrapped, pruner dropping sections, one LLM call per long page, fonts never loaded) as well as asset causes. Adds Phases S0–S5, which run **before** the asset phases. |

> **Priority order:** 04 Phases S0 → S1 → S4 (= 03 Phase 1) → S3 → S2 → S5, then 03 Phases 2–6.

## The Five Root Causes (summary)

1. **The LLM is handed 1-hour presigned S3 URLs** as image sources. Generated code embeds them, so images break about an hour after generation.
2. **Asset paths ignore the Vite base path.** Previews are built with `--base=/p/{projectId}/{jobId}/`, but generated code and `icons.tsx` hard-code `/assets/...`.
3. **Icons are not exported as icons.** The plugin only SVG-exports raw `VECTOR`/`STAR`/... nodes, capped at the first 50. Icon *instances*, groups and frames are never exported. The geometry fallback (`vectorPaths`) is never serialized, and the prompt tells the model to "render a clean inline `<svg>`", so the model invents icons.
4. **Failures are silent at every stage**: empty `catch` blocks in the exporter, unchecked `res.ok` in the uploader, missing objects in ingest, and a transparent 1×1 PNG or empty SVG written by the assembler.
5. **Format and identity are lossy.** Every image fill is saved as `.png` whatever its real bytes are. Asset hashes mix the Figma image hash, SHA-256 and an FNV fallback. The CAS dedupe check reads a prefix nothing writes to.

## Guiding Principles

- **Deterministic, not LLM**: asset discovery, naming, dedupe, SVG→JSX conversion and URL resolution are plain code. The LLM only chooses *where* to place a referenced asset.
- **Reference by ID, never by URL**: prompts and generated code use logical IDs (`assets.img_hero_3fa2c1`, `<IconArrowRight />`). The build resolves them to files.
- **Compile-time safety**: a typed asset registry means an unknown asset ID is a TypeScript error, not a broken image.
- **Fail loud, degrade visibly**: every missing asset is counted and reported in `prototype-status.json`, and rendered as a visible placeholder.
- **Backward compatible**: snapshots from older plugin builds keep working through a legacy resolver path.
