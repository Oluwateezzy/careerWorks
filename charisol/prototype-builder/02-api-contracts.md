# 02 — API contracts

All paths are relative to `/v1/projects/:projectId/prototype`. The FE calls them through BFF routes under `app/api/projects/[id]/prototype/**`. FE types are in `charisol-design-system-fe/types/prototype.ts`.

The FE treats `404`, `405`, `501` and `code: "NOT_IMPLEMENTED"` as "feature not enabled on this server" (`isUnsupportedResponse` in `services/prototype.service.ts`). New endpoints must therefore return a **different** code, such as 400/422/409, for real validation failures.

## 1. Changes to existing endpoints

### `POST /generate` — new optional body fields

```jsonc
{
  "snapshotId": "snap_…",
  "selectedNodeIds": ["1:1", "1:2"],          // FE sends existing ∪ new in append mode
  "strategy": "full" | "append",              // NEW, default "full"
  "baseGenerationId": "job_gen_… | null",     // NEW, required when strategy = "append"
  "instructions": "Login links to Dashboard", // NEW, ≤ 2000 chars
  "options": { "mode": "llm" | "deterministic", "force": true }
}
```

- `options.mode` keeps its current meaning (LLM vs deterministic). That is why the new field is called `strategy`.
- Unknown `baseGenerationId`, or one from another project → `422 INVALID_SOURCE`.
- In append mode, `selectedNodeIds` may include nodes that are not top-level frames of the current snapshot **only** if they exist in the base manifest. Those are kept as-is. Today these IDs return 422 (`prototypeController.js:84`). The FE currently filters them out, so screens whose frames were deleted from the new snapshot are lost on append. Phase 2 removes that limitation.

### Job objects (`GET /jobs`, `GET /jobs/:id`, `POST /generate` response) — new fields

```jsonc
{
  "jobId": "…",
  "workflowType": "generation",
  "source": "figma" | "image" | "mixed",               // NEW
  "baseGenerationId": "job_gen_… | null",              // NEW
  "generationOptions": { "selectedNodeIds": ["…"], "strategy": "append" },  // ensure serialized
  "warningSummary": { "routes": [{ "path": "/", "name": "Home", "nodeIds": ["1:1"] }], "…": "…" },
  "changeReport": {                                     // NEW, present on succeeded jobs
    "summary": "2 pages added · hero image swapped",
    "screens": [
      { "screenId": "home", "nodeId": "1:1", "name": "Home", "route": "/", "status": "added|updated|unchanged|removed" }
    ],
    "edits": [
      { "kind": "image|asset|font|color|text|other", "description": "hero photo replaced", "route": "/", "files": ["public/figma/img_hero.webp"] }
    ],
    "testSteps": ["Open /about and compare it with the design.", "…"]
  }
}
```

`routes[].name` is optional, but the FE shows it in "What changed" when present.

As built (Phase 2), the report also carries:
- `counts` with `{ added, updated, unchanged, removed, failed, edits, conflicts }`;
- `conflicts[]` with `{ screenId, name, route, lockedBy?, reason: "design_changed" | "force_regenerate", status: "conflict" }`. These are edited screens whose design changed; the edited file was kept, and the screen itself is listed as `unchanged` with `edited: true`;
- `truncated` and `fullReportKey` when the job copy was capped at 32 KB. The full report is in `builds/{jobId}/change-report.json`.

Jobs also expose `llmScreenCount` and `generationPlan { strategy, baseGenerationId, selected, generated, reused, removed, conflicts }`. The daily cap counts `llmScreenCount` (`PROTOTYPE_DAILY_SCREEN_LIMIT_PER_PROJECT`, default 100 screens). A bad `baseGenerationId` returns `422 BASE_GENERATION_NOT_FOUND`, not `INVALID_SOURCE` as first written above.

## 2. New endpoints

### `POST /sources/presign`

```jsonc
// request
{ "files": [{ "name": "home.png", "contentType": "image/png", "sizeBytes": 812345, "role": "screenshot" | "asset" }] }
// 200
{ "status": true, "data": { "slots": [
  { "name": "home.png", "role": "screenshot", "presignedUrl": "https://s3…", "uploadKey": "src_…/0" }
] } }
```

- `slots` must be in **request order**, with the same length.
- Limits: screenshots ≤ 10 MB (PNG, JPEG, WebP); assets ≤ 5 MB (PNG, JPEG, WebP, SVG, GIF); ≤ 20 files per request. Violations → `422`.
- The FE PUTs with `Content-Type` equal to the file type. The presign must sign that header.

### `POST /generate-from-images`

```jsonc
{
  "baseGenerationId": "job_gen_… | null",
  "screens": [{ "name": "Checkout", "route": "/checkout", "screenshotKey": "src_…/0", "notes": "…" }],
  "assetKeys": ["src_…/1", "src_…/2"],
  "instructions": "Sign in goes to Dashboard",
  "idempotencyKey": "optional (also accepted as x-idempotency-key header)"
}
// 202 → job object (source: "image" | "mixed")
```

### `POST /edits`

```jsonc
{
  "baseGenerationId": "job_gen_…",
  "edits": [{
    "kind": "image" | "asset" | "font" | "color" | "text" | "other",
    "target": { "route": "/", "nodeId": "1:1", "assetId": "img_hero_3fa2c1" },  // all optional
    "uploadKey": "src_…/0",            // image / asset (from /sources/presign, role "asset")
    "value": "Inter" | "#4F46E5" | "primary → #4F46E5" | "New copy",
    "instruction": "the hero photo on the landing page"
  }]
}
// 202 → job object (generationOptions.strategy = "edit"); the result shows up as the next version
```

- `font` and `color` with no `target` apply app-wide.
- If the backend cannot resolve the target unambiguously → `422 EDIT_TARGET_AMBIGUOUS`, with `details.candidates[] {assetId|route, label}`. A later FE iteration can present these as choices.

As built (Phase 4):
- Targets are resolved when the request arrives, against the base `source.zip` and `screens-manifest.json`, in `services/prototypeEditService.js` `planEdits`. That is why the 422 is synchronous. `details` is `{ editIndex, kind, candidates }`. A colour candidate is `{ variable, label }`; an asset candidate also has `routes`; a screen candidate also has `screenId`. The resolved `operations` are stored in `generationOptions`, and the worker only applies them.
- Theme file: the plan says "theme.css"; the build actually uses `src/design/tokens.css`, and the registry is `src/generated/assets.ts`. A font edit changes the `--font-*` family variables plus a marked `/* strata-edit:font */` override block in `tokens.css`, and the Google Fonts link in `index.html`. Nothing else changes. The override block is needed because screens set `font-family` literally from the design.
- `value` formats:
  - color: `"primary → #4F46E5"`. It must name exactly one token; a name that matches several (e.g. `primary`) → 422.
  - text: `"old → new"`, or the new copy with the old text quoted in `instruction`.
  - Text with several matches in one file → one LLM call picks the occurrence. Matches on several pages → 422. Text that isn't found, with exactly one page in scope → a single-file LLM patch.
  - `other` requires one target page and does a full-file LLM rewrite of that TSX (not a unified diff), checked by `validator.js` and `verifyTsxSyntax`.
- Image/asset swap: the new file goes to `public/figma/edited/` and the registry entry is repointed. CSS that uses the old public path is rewritten. An asset shared by several pages, with a single page targeted, gets a page-only registry copy (`<assetId>_<editId>`), and that screen becomes `edited`. The bytes are kept at `builds/{editJob}/public/…` for replays.
- `uploadKey` must match `src_…/n` and exist at `projects/{p}/sources/{uploadKey}`; otherwise → `422 SOURCE_UPLOAD_INVALID` with `details.editIndex`. A base without `screens-manifest.json` → `422 BASE_GENERATION_NOT_FOUND`.
- Persistence: app-level edits (asset swaps, colours, fonts) are stored in `screens-manifest.json.edits[]`, and `assemblePrototype` re-applies them after every later assembly (`applyPersistentEdits`). Text and `other` edits survive through the `edited` screen status.
- Pipeline: a Step Functions Choice on `generationOptions.strategy == "edit"` → the `applyPrototypeEdits` Lambda. The build/upload tail it shares with `assemblePrototype` lives in `workers/generate/buildTail.js`. Edit jobs record `llmScreenCount: 0`.

### `POST /generations/:generationId/github-push`

```jsonc
// request (all optional)
{ "commitMessage": "Prototype v3", "prTitle": "Prototype v3", "prBody": "…" }
// 200
{ "status": true, "data": { "branch": "strata/prototype-v3", "commitSha": "…", "prNumber": 12, "prUrl": "https://github.com/…/pull/12" } }
```

- No connected repository → `409 MISSING_CREDENTIAL`, which the FE shows as an error, not as "unsupported".
- The PR body defaults to the change report and the test steps.

## 3. Error codes to add

`EDIT_TARGET_AMBIGUOUS`, `SOURCE_UPLOAD_INVALID`, `BASE_GENERATION_NOT_FOUND`. Add them to `constants/prototypeContracts.js`, and mirror them in `PrototypeErrorCode` on the FE.
