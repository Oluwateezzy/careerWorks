# 03 — Backend implementation plan

Every phase ends with the BE CLAUDE.md checklist: unit + integration tests, lint, and contracts:check. The FE is already built against [02-api-contracts.md](./02-api-contracts.md). After each phase, verify the matching FE flow end to end.

## Phase 1 — Serialization and contract plumbing (≈0.5 day)

Small changes that improve the FE immediately.

1. `services/prototypeJobService.js`: make sure job responses include `generationOptions.selectedNodeIds`, `baseGenerationId` and `source`, default `"figma"`.
2. `controllers/prototypeController.js` `startGeneration` (:46-160):
   - accept `strategy`, `baseGenerationId` and `instructions`;
   - validate them (append requires the base to exist, be in this project, and have succeeded → `BASE_GENERATION_NOT_FOUND`);
   - store them in `generationOptions`;
   - add `strategy` and `baseGenerationId` to the idempotency hash (:97-105).
3. `workers/generate/assembler.js:504`: include `name` in each routes-manifest entry.
4. `constants/prototypeContracts.js`: add the new error codes and the `strategy` enum. Update the OpenAPI spec and the contracts schema.
5. Tests: controller validation for append without a base, base from another project, and instructions over 2,000 chars; job serialization snapshot.

**Exit:** the FE builder shows real page names in "What changed", and append requests are validated.

## Phase 2 — Screen manifest, reuse, append, change report (≈4–5 days) ★ core

1. **Hashing**, in `workers/generate/screenPackage.js`: `computeScreenHash(pkg)` over the pruned node JSON, the render PNG hash, the tokens hash, referenced asset IDs, notes/instructions, and `GENERATOR_VERSION`. Unit-test that it is stable under key order and changes when any input changes.
2. **Manifest writer**, in `workers/generate/lambda/assemblePrototype.js`: write `screens-manifest.json` (schema in 01 §2.1) beside `prototype-status.json`.
3. **Planner**, in `workers/generate/lambda/prepareScreenPackages.js`:
   - load the base manifest: `baseGenerationId`, or `PrototypeConfig.latestGenerationId` for `full`;
   - classify each node as reuse / generate / regenerate; `options.force` → everything is generated;
   - emit Map items only for generate + regenerate;
   - pass `reuse[] {screenId, files, baseJobId}` to the assembler via state.
4. **Assembler**: copy reused files from `builds/{base}/src` (S3 copy, not download/upload, where possible). In append mode, also keep base screens that were not selected. Rebuild routes/nav from the union.
5. **Change report**, in a new `services/prototypeChangeReportService.js`:
   - `buildChangeReport(baseManifest, newManifest, flows, edits)`, with deterministic `testSteps`;
   - store it on the job (`changeReport`, ≤ 32 KB, full version in S3), in `prototype-status.json`, and in `CHANGELOG.md` in the source.
6. **Limits**: count LLM-generated screens for the daily cap (`prototypeController.js:63`).
7. **Edited-screen protection**: when a regenerate hits a screen with `status: "edited"`, keep the edited file and add a `conflict` entry to the report. Reuse `services/manualEditProtectionService.js`.
8. Tests:
   - planner classification table;
   - assembler reuse copies exactly the manifest's files;
   - change report fixtures for first version, append, update, removal and conflict;
   - integration: two consecutive appends → the second job makes 0 LLM calls for existing screens.

**Exit:**
- adding 1 page to a 10-page app runs 1 screen generation, and the preview contains all 11 pages;
- regenerating with an unchanged snapshot makes 0 LLM calls (without `force`);
- the FE "What changed" shows the backend summary.

## Phase 3 — Push a version to GitHub (≈1.5 days)

1. New route `POST /generations/:generationId/github-push` → `githubController.pushGeneration`.
2. Read the build source from S3 (the same source as `downloadSource`), filter out `node_modules` and `dist`, and call `githubAppService` publish (:936-980) with `files[]`. Use branch `strata/prototype-v{n}`; reuse the open PR when present.
3. PR body template: summary, what changed, how-to-test checklist, preview link.
4. Persist `repositoryRef` on the job and the code_bundle artifact, and record a `github_push` job.
5. Fix the `POST /github/pull-requests` mis-wire (`routes/v1/index.js:588`).
6. Tests: file filtering, PR body rendering, the no-repo → 409 path, and the second push updating the existing PR.

**Exit:** the FE "Open pull request" returns a PR URL, and the PR contains only prototype source.

## Phase 4 — Targeted edits (≈4 days)

1. Route `POST /edits` → `prototypeController.requestEdit`. Validate edits (1–10 per request, known kinds, and an `uploadKey` that belongs to this project).
2. `services/prototypeEditService.js`: one operation per kind (01 §2.3):
   - `image`/`asset` swap via the `assets.ts` registry;
   - `font`/`color` via `theme.css` variables;
   - `text` via an exact replace, with an LLM fallback only when the match is ambiguous;
   - `other` via a single-file patch.
3. Pipeline: a lightweight Step Functions branch (or a direct Lambda) that copies the base source, applies the operations, and runs the existing build → buildFixer → validate → upload tail. Write a manifest marking the touched screens as `edited`.
4. Ambiguity → `422 EDIT_TARGET_AMBIGUOUS` with `details.candidates`.
5. Tests:
   - each operation against a fixture build;
   - an edit followed by an append preserves the edit;
   - a font edit touches only `theme.css` and `index.html`.

**Exit:** swapping the hero image produces a new version in under 90 s with no screen-generation calls, and the change report lists the edit and a test step.

## Phase 5 — Screenshot sources (≈5 days)

Depends on Phase 4's presign.

1. `POST /sources/presign`: S3 presigned PUT under `projects/{p}/sources/{uploadId}/`, with type and size allowlists (02 §2). Use a lifecycle rule to expire unreferenced uploads after 7 days.
2. `services/imageSourceService.js`:
   - verify uploads (HEAD, magic-byte sniff, read dimensions);
   - create synthetic node IDs `img:{hash12}`;
   - register assets in the asset-index format used by figma-asset-export Phase 3.
3. `POST /generate-from-images` → build image screen packages (PNG + assets + notes), then the existing Map. Add a vision-only prompt variant in `workers/generate/prompts/` that references assets by registry ID, never by URL. This follows the figma-asset-export principles.
4. Mixed apps: with `baseGenerationId`, reuse the Phase 2 planner, which treats image screens like any other manifest entry. Set `source` to `image` or `mixed`.
5. Tests:
   - upload validation;
   - synthetic IDs are stable across retries;
   - the prompt contains asset IDs and no presigned URLs;
   - an integration run with 2 screenshots and 3 assets.

**Exit:**
- a user with no Figma plugin uploads 2 screenshots and a logo and gets a 2-page prototype that uses the real logo file;
- adding a third screenshot later reuses the first two pages.

## FE follow-ups (after the backend lands)

- Show `EDIT_TARGET_AMBIGUOUS` candidates as choice chips in `UpdateCard`.
- Show `conflict` entries from the change report, with "keep my edit / use new design".
- Remove the now-unused legacy components (`GenerationPanel`, `GenerationHistory`, `PrototypeActionFooter`, `PrototypeJobStatus`, `GenerationProgress`) and their tests once the team confirms they aren't needed elsewhere.
- Persist the builder thread per project (today it lives for the page session only). This could use a small `GET/PUT /prototype/builder-thread` or localStorage.
