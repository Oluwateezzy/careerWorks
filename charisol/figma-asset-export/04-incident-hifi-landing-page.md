# 04 — Incident Analysis: "Hi-fi" landing page generated with the wrong layout, text and images

## 1. Symptom

| | Figma (selected) | Generated preview |
|---|---|---|
| Frame | `Hi-fi`, **5183 × 15400 px** (node `35:4175`). It contains a ~1440 px-wide Plexo landing page (~16 sections) **plus loose elements** on the board (green buttons top-left, chips mid-left). | `job_gen_1792659872396_muzw0` |
| Hero | "What are we building today?" + prompt input + product mockup image | "Build it with a prompt." + three text cards (Websites / Funnels / Blog) |
| Images | Hero mockup, product screenshots, photo of a person with boxes, UI cards | **None**, not even placeholders |
| Typography | Sans-serif brand font | Browser default **Times** (serif) |
| Sections | ~16 | Header, an invented hero, then "Your tools are costing you customers." … (later sections missing or invented) |

The backend PR (`e261594 fix(prototype): wire existing figma svg and image assets…`) did not change the result.

## 2. Timeline (evidence)

| Time (BST, 22 Oct 2026) | Event |
|---|---|
| 21 Oct 17:06 | `e261594` asset wiring committed (the deployed PR) |
| **22 Oct 10:04:32** | Generation job `job_gen_1792659872396_muzw0` starts (timestamp taken from the job ID) |
| **22 Oct 11:13:03** | `845c8b6 feat: add screen root unwrapping and rendering utilities` committed |

**The job ran before the unwrapping and render-slicing work existed.** Those were the changes aimed at exactly this frame shape. The fix for this case has therefore never been exercised in production. It also needs the corrections in §4.2 before it can be trusted.

The signed preview URL expired at epoch `1792663559` (11:05:59 BST), so the live output could not be inspected for this analysis. See §6 for the artifacts needed to confirm the hypotheses.

## 3. Root Causes

Each cause below can produce this output on its own. Together they guarantee it. Labels: **[C]** confirmed from code, **[H]** hypothesis that needs the job artifacts in §6.

### RC-1 [C]: The model never saw the design (no screenshot)
Before `845c8b6`, `screenPackage.js` looked for the render only at:
```
renders/35_4175.png · renders/35:4175.png · assets/cas/35:4175.png
```
The plugin writes renders as `renders/{id}_{hash8}.png`. For this job that is `renders/35_4175_d18d798c.png`, the file in the repo root. **No key matched, so `pngKey = null`, and the generation ran on layer JSON only.** That is why the layout and copy are "plausible Plexo" rather than *this* design.

Even with a matching key, the render is **5263 × 15480 px**, above Claude's 8000 px per-side limit. Bedrock would reject it, or a downscale to model size would leave the ~1440 px page about 150 px wide and unreadable.

### RC-2 [C]: The selection is a board, not a screen
`Hi-fi` is a 5183 px-wide presentation board. The real page is one child frame, and the loose elements are siblings of it. With no unwrapping (pre-`845c8b6`):
- The node tree starts one level too high, which spends a level of the depth budget (RC-3).
- The loose buttons and chips are sent as screen content, and the model works them into the layout.
- Coordinates are relative to the board, not the page.

### RC-3 [C]/[H]: The pruner throws away most of a long page
`pruneNodeTree` (`screenPackage.js`) applies fixed limits that suit a 1440×900 app screen but not a 15,000 px landing page:

| Limit | Value | Effect on this page |
|---|---|---|
| `maxRepeated` | 3, keyed on `type + name with digits → #` | **[H] main suspect.** Auto-named section frames (`Frame 427318…`, `Frame 12`) all collapse to `FRAME:Frame #`. **Only the first 3 sections are kept**, and the rest become `"…additional repeated items omitted"`. Cards inside sections suffer the same way. |
| `maxDepth` | 8 (counted from the board) | Board → page → section → container → row → card → stack → text sits at depth 8. Headings and images nested deeper become `_hasMoreChildren` stubs. **[H]** This would explain "What are we building today?" being replaced by invented copy. |
| `maxChildren` | 20 | Any further children are dropped as `_truncatedChildrenCount`. |
| Child order | Figma z-order | For a non-auto-layout page, z-order ≠ top-to-bottom order. Combined with the 3-repeat cap, the *kept* sections can be arbitrary. |
| Token budget (15k) | **warning only** | `prepareScreenPackages.js:126-131` logs the overrun and generates anyway. |

### RC-4 [C]: One LLM call for a 16-section page
The whole page is one `generateScreen` call with `maxTokens = 16384`. Full TSX plus a CSS module for ~16 rich sections exceeds that easily, so it relies on continuation and repair. The model compensates by summarising, which explains generic cards where the design has detailed sections. **[H]** Check `stopReason: max_tokens` in the logs.

### RC-5 [C]: Images cannot appear
- Image-bearing nodes are often deep (RC-3) or inside collapsed sections, so the model never received their `imageRef`.
- When refs do arrive, the manifest values are **1-hour presigned S3 URLs** (`snapshotLoader.buildImageAssetMap`). `e261594` *widened* this: it now presigns svg, render, thumbnail and `nodeId` refs as well, and adds the **whole-frame `renderRef`** of the root as a usable "asset". Any image that was emitted stopped working an hour later. See `01-gap-analysis.md` G-1.
- `e261594` also changed the prompt to *allow* "render a clean inline `<svg>`", which invites the model to invent graphics (G-3).

### RC-6 [C]: Fonts are never loaded
- `prepareScreenPackages.js:163` builds `brandContext = source.brandContext || { fonts: source.manifest?.fonts || [] }`.
- The plugin manifest (`manifestAssembler.ts`) has **no `fonts` field**, and `snapshotLoader` never sets `brandContext`. So `fonts` is always `[]`.
- As a result, `injectGoogleFontsIntoHtml` is skipped, the template has no default `font-family`, and the browser falls back to Times. The screenshot confirms this.

## 4. Status of `845c8b6` (not yet live for this job)

### 4.1 What it fixes
- `resolveScreenRoot` unwraps boards. For this frame the width ratio is 5183/~1440 ≈ 3.6 (≥ 1.5) and the dominant child easily exceeds 60 % of the area, so the page **would** be unwrapped. **This fixes RC-2.**
- The render lookup now uses `renderRef` first. **This fixes the RC-1 key mismatch.**
- `screenRender.prepareScreenImages` crops to the page and slices it into ≤ 10 JPEG tiles of 960 × 1200. For this page (~1440 × 15000) the scale is ≈ 0.64, giving ~8 tiles. **This mostly fixes the RC-1 size problem.** Body text at 0.64 scale is borderline legible.
- The crop maths is correct because `snapshotLoader` converts absolute bounds to parent-relative (`snapshotLoader.js:141-146`).

### 4.2 Defects to fix before relying on it
| # | Issue | Fix |
|---|---|---|
| F-1 | When `sharp` fails to load or decode, the code **falls back to sending the raw render** (`screenGenerator.loadScreenImages`). For this frame that is 5263×15480, which Bedrock rejects (> 8000 px). Generation then fails or retries without an image. | Never send an image > 8000 px per side or > 3.75 MB. If slicing fails, send no image and mark the screen `degraded` with reason `RENDER_UNUSABLE`. |
| F-2 | `sharp` is a native module in a zip-packaged Lambda (Node 20, built on the CI runner). It is probably fine (prebuilt `@img/sharp-linux-x64`), but unverified. | Log `sharp` availability at cold start, and add a CI smoke test that `require('sharp')` succeeds inside `../package`. |
| F-3 | It does not address RC-3, RC-4, RC-5 or RC-6. | See §5. |

## 5. Fix Plan (inserted ahead of `03-implementation-plan.md` Phase 1)

### Phase S0: Confirm & unblock (½ day)
1. Deploy `845c8b6`. Record the deployed SHA in the job record (`PrototypeJob.codeVersion`) so that "was the fix live?" can be answered from the job itself.
2. Fix F-1 and F-2.
3. Regenerate `Hi-fi`. Collect the §6 artifacts and confirm or refute H-RC-3 and H-RC-4.

### Phase S1: Pruner fit for long pages (BE, `screenPackage.js`)
- **Never collapse or truncate the direct children of the content root.** They are the page's sections.
- **Repetition key = structural signature**, not name: `type + ordered child types (depth 2) + width bucket`. Only collapse siblings that are genuinely identical in structure, and keep the first one's full subtree as the template.
- **Order children visually**: when the parent has no auto-layout, sort children by `(y, x)`.
- **Budget-driven depth instead of a fixed depth 8.** Expand breadth-first until the token budget is reached, measuring depth from the *content root*.
- **Never drop text or images silently.** For a subtree cut by budget, attach `texts: [...]` (all `characters` in order) and `images: [assetId…]`, so copy and imagery survive even when the structure is summarised.
- `prepareScreenPackages`: a budget overrun triggers **section splitting (S2)**, not just a warning.

### Phase S2: Section-level generation (structural fix for RC-3/RC-4, BE + CDK)
For screens taller than `SECTION_SPLIT_MIN_HEIGHT` (default 2000 px), or over the token budget:
1. **Split.** Sections are the content root's children, sorted by y. Merge siblings shorter than 120 px into their neighbours, and recursively split any section over the budget.
2. **Package per section**: its own pruned tree (full depth budget), its own render crop at a **legible scale** (width ≤ 1440 → tiles 1200 px tall, ≤ 3 tiles), and its own asset refs.
3. **Generate per section.** Step Functions Map iterates over `sectionPackages[]` (flattened across screens), so concurrency and retries already work. Output is `{SectionName}.tsx` plus `.module.css`.
4. **Compose deterministically.** `{Screen}.tsx` imports the sections in order inside a page wrapper (width, background, tokens). No LLM is involved.
5. **Section fallback.** A failed section renders its **Figma render crop as an `<img>`**, so the preview stays visually complete. It is marked `degraded` in `prototype-status.json`.
6. The output-token cap now applies per section, so continuation and repair are rarely needed.

### Phase S3: Fonts (RC-6)
- `snapshotLoader`: collect the distinct `fontFamily` and weights from all TEXT nodes (including `segments[]`) into `brandContext.fonts`, with usage counts.
- `assembler`: inject Google Fonts for families on Google Fonts. Otherwise emit a `FONT_NOT_AVAILABLE` warning, and use a metric-compatible fallback stack.
- Template base CSS: `body { font-family: var(--font-body, system-ui, -apple-system, "Segoe UI", sans-serif); }`. The page can never fall back to Times again.
- Plugin (optional): emit `fonts[]` in the manifest.

### Phase S4: Assets (RC-5)
Run `03-implementation-plan.md` **Phase 1** immediately after S1. Priority items:
- Remove presigned URLs from the prompt, **including those that `e261594` added**.
- Exclude `renderRef`/`thumbRef` from the per-screen asset manifest. Renders are model input, never page content (except the S2 fallback).
- Restore the prompt rule that forbids inventing icons and graphics.

### Phase S5: Fidelity gates (catch this automatically)
Add these to `prototype-status.json` per screen and section, and fail the badge below the thresholds:
| Metric | How | Threshold |
|---|---|---|
| **Text coverage** | % of Figma TEXT `characters` (normalised) found in the generated TSX | ≥ 0.95 |
| **Section coverage** | Generated sections / Figma sections | 1.0 |
| **Image coverage** | Image refs used / expected (see `02 §7`) | ≥ 0.9 |
| **Visual diff** | Playwright (already in the generate Docker image) screenshots each section of the built preview; compare SSIM against the Figma render crop | ≥ 0.80 (tune on fixtures) |
| **Render sent** | The model received a screenshot (`pngKey` set and tiles > 0) | true |

This job would have failed all five gates.

## 6. Artifacts Needed to Close the Hypotheses

Run these with the right AWS profile (`!` prefix in this session). Replace `$BUCKET` and `$STACK` as appropriate.

```bash
P=01a1349d-3848-717c-85e3-4b4d06fcaa4e; J=job_gen_1792659872396_muzw0
aws s3 ls s3://$BUCKET/projects/$P/jobs/$J/ --recursive | head -50
aws s3 cp s3://$BUCKET/projects/$P/jobs/$J/screens/35:4175/package.json -   # exact key: take it from the ls above. Pruned tree: look for _collapsed, _hasMoreChildren, pngKey
aws s3 cp s3://$BUCKET/projects/$P/jobs/$J/prototype-status.json -
aws logs filter-log-events --log-group-name /aws/lambda/$STACK-screen-generator \
  --filter-pattern '"35:4175"' --start-time 1792659800000 | head -100       # stopReason, image presence, retries
aws logs filter-log-events --log-group-name /aws/lambda/$STACK-prepare-screens \
  --filter-pattern '"token budget exceeded"' --start-time 1792659800000
```

What each answers:
- `package.json` → `pngKey: null` confirms RC-1. A count of `_collapsed` entries at depth 2 confirms RC-3 (repeat collapse). `_hasMoreChildren` on hero nodes confirms the depth cut.
- Generator logs → `stopReason: "max_tokens"` or continuation attempts confirm RC-4.
- `prototype-status.json` → which assets were written or fell back.

## 7. Acceptance for This Incident

Regenerate `Hi-fi` after S0–S4:
- [ ] The hero reads "What are we building today?". Every Figma section is present, **in order**, and the loose board elements are absent.
- [ ] Text coverage ≥ 0.95, and section coverage = 1.0.
- [ ] Hero mockup, product screenshots and the photo are visible, and **still visible > 1 h later**.
- [ ] Brand font loaded, with no serif fallback.
- [ ] `prototype-status.json` reports `renderSent: true` for every section, and the visual diff is above threshold.
