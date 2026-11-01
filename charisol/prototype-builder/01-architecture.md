# 01 — Architecture

## 1. Frontend (implemented)

### 1.1 Why the old page didn't work

The old `PrototypeSection` stacked about 10 blocks vertically: the header, an error banner, two job banners, generation progress, a "ready" bar, the preview, a list of raw job IDs, a snapshot header with 7 count tiles, a 7-tab data explorer, the GitHub card, and a sticky footer with 4 buttons and a modal. Generation was a side effect of a data explorer. Specific problems:

- There were two generate buttons with different modes, plus a localStorage "Save selection".
- History showed raw job IDs. "Push to GitHub" was permanently disabled.
- The preview pushed the explorer down the page.
- There was no notion of an *app* that spans generations.

### 1.2 New structure

```
Prototype Studio                       [GitHub: owner/repo ✓] [ App | Design source ]
┌─────────────────────────────────────────────┬──────────────────────┐
│  PreviewFrame (viewport, routes, fidelity,  │  v3 · Current version │
│  asset health, warnings)                    │  What changed         │
│                                             │  How to test (☐ list) │
│                                             │  [Download] [GitHub]  │
│                                             │  [+ Add pages/update] │
│                                             ├──────────────────────┤
│                                             │  Versions v3 v2 v1    │
└─────────────────────────────────────────────┴──────────────────────┘
                                                        (✦ Builder)  ← floating button
```

- **App view**: the live preview plus a version rail. With no versions it shows an empty state with two source cards (*From Figma*, *From screenshots*) and the three-step story.
- **Design source view**: the existing Figma explorer (screens, flows, components, tokens, assets, layers, warnings). Ticking screens here queues them for the builder.
- **Builder** (floating button, then a chat drawer): every action lives here as an inline card in a thread:
  - `ScreenPickerCard`: Figma screens. Frames already in the app are badged **In app** and locked.
  - `ScreenshotCard`: screenshots plus assets, with optional notes.
  - `UpdateCard`: image / icon-logo / font / colour / text, applied to one page or the whole app.
  - `GitHubCard`: connect, or open a PR for the current version.
  - `JobCard`: live progress. On success it turns into **What changed + How to test** with *Open preview* and *Push to GitHub*.
  - The composer turns free text into the right card using `detectIntent()` in `lib/prototype-changes.ts`. The text is carried along as `instructions`.

Key files: `components/sections/project/PrototypeSection.tsx` (orchestrator), `prototype/builder/**`, `prototype/workspace/**`, `lib/prototype-changes.ts`, `hooks/useJobScreens.ts`, plus four new BFF routes under `app/api/projects/[id]/prototype/`.

### 1.3 How the FE uses today's backend safely

| FE action | Request | Today's backend | After this plan |
| :--- | :--- | :--- | :--- |
| Add pages (Figma) | `POST /generate` with `selectedNodeIds = existing ∪ new`, `strategy: "append"`, `baseGenerationId` | Ignores the new fields and rebuilds everything. The result is still correct. | Reuses unchanged existing screens and only generates the new ones |
| What changed / How to test | reads `job.changeReport` | absent → FE diffs `warningSummary.routes` between versions | backend report wins |
| Screenshots | `POST /sources/presign`, `POST /generate-from-images` | 404 → "isn't enabled yet" | works |
| Update image/font/… | `POST /edits` | 404 → "isn't enabled yet" | works |
| Push version to GitHub | `POST /generations/:id/github-push` | 404 → "isn't enabled yet; download the .zip" | opens PR |

## 2. Backend target architecture

### 2.1 Core concept: the screen manifest

Every successful build writes `projects/{p}/builds/{jobId}/screens-manifest.json`:

```json
{
  "jobId": "job_gen_…",
  "baseGenerationId": "job_gen_… | null",
  "source": "figma | image | mixed",
  "screens": [
    {
      "screenId": "home",
      "nodeId": "1:1",
      "route": "/",
      "name": "Home",
      "source": "figma",
      "contentHash": "sha256:…",
      "generatorVersion": "screen-gen@2026-10",
      "files": ["src/screens/Home.tsx"],
      "status": "generated | reused | edited"
    }
  ],
  "appFiles": { "theme": "src/theme.css", "assets": "src/assets.ts", "routes": "src/routes.tsx" },
  "edits": []
}
```

`contentHash` = sha256 of the **screen package inputs**: the pruned node JSON, the render PNG hash, the tokens hash, the asset IDs referenced, the per-screen instructions, and `generatorVersion`. That is enough to decide whether a frame "has been generated before" **and** is unchanged.

### 2.2 Append strategy (generate only what's new or changed)

```
POST /generate {strategy:"append", baseGenerationId, selectedNodeIds, instructions}
        │
        ▼
prepareScreenPackages ── load base screens-manifest.json
        │   for each selected node:
        │     hash == base.hash  → mark REUSE (no Map item, no LLM call)
        │     hash != base.hash  → mark REGENERATE (status "updated")
        │     not in base        → mark GENERATE (status "added")
        ▼
Step Functions Map (only GENERATE + REGENERATE items)
        ▼
assemblePrototype ── copy REUSE files from builds/{base}/src → new build
                  ── rebuild routes/nav from the union, vite build, upload
                  ── write screens-manifest.json + changeReport
```

- `strategy: "full"` keeps today's behaviour but **still** uses the hash cache against the project's latest generation (`PrototypeConfig.latestGenerationId`). So "a frame that has been generated before" is never sent to the LLM again unless it changed. `options.force` bypasses the cache.
- Screens in the base but not in `selectedNodeIds` are **kept** in append mode, and dropped (status `removed`) in full mode.
- The idempotency hash must include `strategy` and `baseGenerationId`.
- The daily limit (`prototypeController.js:63`) should count **LLM-generated screens**, not jobs, so cheap appends aren't penalised.

### 2.3 Edit strategy (no screen regeneration)

`POST /prototype/edits` creates a job with `workflowType: "generation"` and `generationOptions.strategy: "edit"`, so it appears as a new version in the same history. It copies the base build source, applies the edit operations, then runs only the **build → validate → upload** tail of the existing pipeline. That tail is the assembler's vite build plus buildFixer, with no Map state.

| Edit kind | Operation | LLM? |
| :--- | :--- | :--- |
| `image`, `asset` | Replace bytes behind an `assets.ts` registry entry (target by `assetId`, or resolve from route plus instruction), or add a new registry entry and rewrite one reference | No when `assetId` is known. Otherwise a small resolver call picks the asset ID from the screen's registry usage |
| `font` | Update `--font-*` variables in `theme.css`, add `@font-face`/import (reuse `/fonts` uploads) | No |
| `color` | Override the token CSS variable in `theme.css` (`primary → #4F46E5`) | No |
| `text` | Exact-match string replace in the target screen file. If ambiguous, a constrained single-file patch (unified diff) checked by `validator.js` | Only for ambiguity |
| `other` | Single-screen patch from `instruction`, validated and buildFixer-repaired | Yes, scoped to one file |

`manualEditProtectionService.js` and `ownership.json` (written at `assemblePrototype.js:491`) decide which files edits may touch. A later Figma append must **preserve** edited files: in the manifest those screens have `status: "edited"` and a `lockedBy` edit ID. A re-generation of an edited screen is reported as a conflict in the change report instead of silently overwriting the edit.

### 2.4 Change report

`assemblePrototype` computes the report by diffing the base manifest against the new one:

- `screens[]` with `added | updated | unchanged | removed`, plus `edits[]` from the edit job.
- `testSteps[]`, built **deterministically**:
  - added routes → "Open /x and compare with the design";
  - flows from `/:snapshotId/flows` that touch changed screens → "From Home click *Sign in* → lands on /login";
  - edits → "Check the hero image on / is the new photo";
  - plus a responsive check.
- `summary` (one line). An optional cheap LLM pass can polish the wording, but it is not required.

The report is stored on the job as `changeReport`; cap it at ~32 KB because of the DynamoDB item limit, with the full version in S3. It is also stored in `prototype-status.json` and written as `CHANGELOG.md` plus a `TESTING.md` section in the source zip and PR body.

### 2.5 Screenshot sources

```
POST /sources/presign ──▶ S3 projects/{p}/sources/{uploadId}/{n}  (PUT, 15 min)
POST /generate-from-images {screens[{name,route,screenshotKey,notes}], assetKeys[], baseGenerationId}
        │
        ▼
imageSourceService ── HEAD-verify uploads, sniff type, read dimensions
                   ── build an "image snapshot": synthetic node id  img:{sha256(screenshot)[0:12]}
                   ── assets → asset-index.json entries (same schema as Figma CAS assets)
        ▼
prepareScreenPackages (image variant: PNG + asset list + notes, no node JSON)
        ▼
same Map → generateScreen (vision-only prompt variant in prompts/) → assemble
```

Using synthetic node IDs means append, reuse, change reports and "In app" badges work the same for both sources. `baseGenerationId` lets one app mix Figma and screenshot pages. The job's `source` is `"image"` (or `"mixed"` when the base was Figma).

### 2.6 GitHub push per version

`POST /prototype/generations/:id/github-push`:

1. Reads the build source from S3, excluding `node_modules` and `dist`.
2. Calls `githubAppService` publish (`githubAppService.js:936-980`) with the files on branch `strata/prototype-v{n}`.
3. Uses the PR body `changeReport.summary` + `testSteps` + preview link.
4. Records `repositoryRef {owner, repo, branch, commitSha, prNumber}` on the job and code_bundle artifact, and creates a `github_push` job, a type that already exists in the enum.

Later versions push to a new branch, or update the open PR when it is still open, so developers review **diffs between versions**, not whole apps.

### 2.7 Cross-cutting

- **Instructions**: `instructions` (job-level) and per-screen `notes` go into the screen package and the prompt. They are capped at 2,000 chars and included in `contentHash`.
- **Security**: presigned uploads are scoped to the project prefix, with content-type allowlists and size caps. Edit and push endpoints require editor role, the same check as `/generate`. While here, add validation to `PATCH /prototype/config`, which today merges `req.body` straight in (`prototypeController.js:665`), and fix `POST /github/pull-requests`, which is mis-wired to `connectExistingRepository` (`routes/v1/index.js:588`).
- **Observability**: log reuse ratio per job (`reused / selected`) and edit-job duration. These are the numbers that prove the "don't regenerate" promise.
