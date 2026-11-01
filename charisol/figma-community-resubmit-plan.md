# Plan: resubmit Strata Design System to Figma Community

**Date:** 28 October 2026
**Depends on:** [figma-community-rejection-findings.md](./figma-community-rejection-findings.md)
**Outcome:** a production build that connects and uploads, a privacy disclosure that matches that build, and a Community form reviewers can approve.

Do the phases in order. A corrected listing on the current `dist/` will fail again, because that bundle calls a pull-request host that does not resolve.

---

## Decisions locked for this plan

| Decision | Choice |
|---|---|
| Which API the Community build calls | `https://api-production-charisol-design-system-be.aws.charisol.io/v1/` only |
| Which app the device code opens | `https://strata.charisol.io` |
| Who is named as operator | Charisol. Support mailbox already used on the pricing page: `support@charisol.io`. Confirm that inbox is read before it goes on the listing. |
| Sign-in in the Community build | Device code only. Password entry and the pasted-token shortcut stay out of this build. |
| Network allowlist | Production API prefix, plus the one S3 bucket host copied from a live presigned URL. No `*.amazonaws.com`, `*.s3.amazonaws.com`, or `*.aws.charisol.io`. |
| Unfinished uploads | Tag `strata-plugin=pending` on presign. S3 expires that tag after 7 days. |
| Finished uploads | Ingest copies them to `projects/{projectId}/figma-snapshots/{snapshotId}/`. Those objects stay until the project owner deletes the project. |
| Deletion | Project delete must remove both S3 prefixes. Token revoke already exists on the profile page, Connected Plugins (`DELETE /v1/plugin/tokens/:tokenId`, 90-day sliding expiry). |
| What reviewers install | The GitHub Actions artifact, which already sets the production URL. A laptop `npm run build` is allowed only after the build refuses pull-request and localhost URLs. |

`deleteProjectById` now deletes stored plugin data before the DynamoDB project row. That behavior is in the repo and is not on the live API until the phase 2 production deploy. The privacy text matches that code. Do not resubmit until the deploy is live, because the 7-day and project-delete sentences are false on the current production API.

---

## Phase 0 — Record the real S3 host

**Done.** Production plugin uploads go to:

`https://production-charisol-design-system-be-project-data.s3.us-east-1.amazonaws.com`

Recorded in `charisol-design-system-be/constants/pluginUploadDestination.js`, `charisol-figma-plugin/manifest.json`, and the phase 4 disclosure below.

How it was confirmed on 28 Oct 2026:

- The production data stack names the bucket `production-charisol-design-system-be-project-data`. A HEAD of that bucket returns 403 from Amazon S3 with `x-amz-bucket-region: us-east-1`. The bucket exists and is private. `charisol-design-system-data` returns 404.
- The installed `@aws-sdk/s3-request-presigner` signs that bucket, in `us-east-1`, as the regional virtual-hosted host above. `tests/unit/plugin-upload-host.test.js` fails if a later SDK changes the host.
- This is the host production Lambda emits. `generatePresignedPutUrl` only falls back to `bucket.s3.amazonaws.com` when AWS credentials are missing, which is the local and test path, not production.

---

## Phase 1 — Make the plugin reach production

**Done.** A production `npm run build` writes `dist/ui.html` with the production API and `strata.charisol.io` only. The same command exits before writing a bundle when `BACKEND_API_URL` is a pull-request host or `FRONTEND_URL` is localhost. `allowedDomains` is the production API prefix and the phase 0 bucket. The connect screen is device-code only, and a failed `fetch` is shown as "Strata could not reach the Charisol service. Check your connection and try again."

This removes `Failed to fetch` on open, and the same error on upload.

### 1.1 Stop local builds from baking a dead API

`esbuild.js` loads `charisol-figma-plugin/.env` and inlines `BACKEND_API_URL`. That file currently points at `api-pr-263-…`. CI in `.github/workflows/ci.yml` already sets the production URL and `FRONTEND_URL=https://strata.charisol.io`, and `loadEnv` does not override an existing env var. A desktop publish of a local `dist/` skips CI.

- Point the local `.env` `BACKEND_API_URL` at the production `/v1` base and `FRONTEND_URL` at `https://strata.charisol.io`. Do not commit `.env`.
- Add `.env.example` with those two production values.
- In `esbuild.js`, fail the build when `BACKEND_API_URL` or `FRONTEND_URL` contains `api-pr-`, `localhost`, or `127.0.0.1`, unless the command is `npm run watch`.
- After a production build, grep `dist/ui.html` and fail if `api-pr-` or `localhost` appears.

`src/services/api.ts` falls back to `http://localhost:5000/v1`. Change that fallback to the production URL so a missed define still matches the allowlist. Keep localhost in `devAllowedDomains` for `npm run watch`.

### 1.2 Allow only the two production hosts

In `charisol-figma-plugin/manifest.json`:

```json
"networkAccess": {
  "allowedDomains": [
    "https://api-production-charisol-design-system-be.aws.charisol.io/v1/",
    "https://production-charisol-design-system-be-project-data.s3.us-east-1.amazonaws.com"
  ],
  "devAllowedDomains": [
    "http://localhost:5000"
  ],
  "reasoning": "The plugin signs in to the Charisol production API and uploads the open file's inventory to the Charisol production S3 bucket."
}
```

The API entry needs the trailing slash. Figma treats a pattern without one as a single path, so `…/v1` blocks `…/v1/plugin/auth/device`.

The S3 entry is the phase 0 host. Use that full host, no wildcard.

### 1.3 Remove the screens that send reviewers to the wrong place

| Surface | File | Change |
|---|---|---|
| API URL field showing `api-pr-256-…` | `src/components/SettingsModal.tsx`, `src/ui.tsx`, `src/components/Header.tsx` | Remove the settings view and the gear button from the Community UI. `setApiBase` goes with it. |
| IPC benchmark | `src/components/IPCBenchmarkView.tsx`, header button in `Header.tsx` | Remove the button. Leave the component unused or delete it. Reviewers should not see a developer harness. |
| Password form and `pt_` token shortcut | `src/components/LoginScreen.tsx`, `login()` in `src/services/api.ts` | Device code only. Delete the password mode and the branch that treats a long email as a token. |
| Raw `err.message` | `LoginScreen.tsx`, `ProjectPickerScreen.tsx`, `ExportProgressView.tsx` | Add `src/services/errors.ts` with one mapper. `TypeError` / `Failed to fetch` becomes: "Strata could not reach the Charisol service. Check your connection and try again." HTTP status stays specific ("Sign-in expired", "You do not have access to this project") and never includes a stack or the words Failed to fetch. |

`ExportProgressView.tsx` opens `process.env.FRONTEND_URL` after a sync. That value must be `https://strata.charisol.io` in the same build.

### 1.4 Tests

- Unit-test the error mapper: a `TypeError` whose message is `Failed to fetch` returns the connection sentence.
- Extend `tests/pipeline.test.ts` so a thrown fetch on the presign call returns that sentence, not the raw message.
- Add a small node script, `scripts/assert-community-build.js`, run from `npm run build`, that reads `dist/ui.html` and `manifest.json` and exits non-zero if it finds `api-pr-`, `localhost`, `*.amazonaws.com`, or `*.aws.charisol.io`.

**Done when:** `npm run build` on a machine whose `.env` still has a pull-request URL exits with an error, and a build with the production env produces a `dist/ui.html` whose only Charisol host is production.

---

## Phase 2 — Make the backend match the disclosure

**Code is in the repo. Production is not updated yet.** There is no AWS CLI on this machine, so the production API stack and data stack have not been deployed. Until that deploy, live preflight still returns `Access-Control-Allow-Origin: *` with credentials, unfinished uploads are not tagged, and deleting a project still leaves S3 objects in place. Do not write the privacy policy, and do not resubmit, until this deploy is live.

Ship this to production before the Figma test in phase 4. The plugin can be coded in parallel. The review build cannot be published first.

### 2.1 CORS on the API

`cdk/lib/charisol-design-system-stack.ts` sets `allowOrigins: Cors.ALL_ORIGINS` and `allowCredentials: true`. A live preflight already returns both `access-control-allow-origin: *` and `access-control-allow-credentials: true`.

Change the API Gateway preflight to:

- Allow origins `https://www.figma.com`, `https://figma.com`, and `null`.
- Allow methods `GET, POST, PUT, OPTIONS`.
- Allow headers `Content-Type`, `Authorization`, `X-Figma-Token`.
- Leave credentials off. The plugin `fetch` calls do not send cookies.

Deploy the production stack. Then repeat the curl from the findings doc:

- `OPTIONS /v1/plugin/auth/device` with `Origin: null` returns `204` and `access-control-allow-origin: null`.
- `POST` the same path returns a device code and the same allow-origin value.

Express in `app.js` already reflects the request origin. Leave that in place so the POST, which API Gateway forwards to Lambda, stays consistent with the preflight.

### 2.2 Tag plugin uploads and expire the unfinished ones

Copy the image-source pattern in `services/imageSourceService.js`.

1. In `controllers/pluginController.js` `presignPluginUpload`, pass `tagging: "strata-plugin=pending"` into `generatePresignedPutUrl`.
2. Return `headers: { "Content-Type": file.contentType, "x-amz-tagging": "strata-plugin=pending" }` on each URL. The tag is signed, so the plugin must send that header exactly.
3. In `src/uploader/uploadOrchestrator.ts`, send `headers` from the presign response on every PUT, including `manifest.json`.
4. On a successful `completePluginUpload`, retag the uploaded keys `strata-plugin=ready`. The existing helper is `utils/s3.js` `tagObject` (image sources use `tagObjectInS3`).
5. In `cdk/lib/data-stack.ts`, add a lifecycle rule: prefix `projects/`, tag `strata-plugin=pending`, expire after 7 days, plus abort incomplete multipart uploads after 1 day. Do not enable `scripts/configure-s3-lifecycle.js`. That script expires every object under `projects/` after 7 days, including finished project data.

Update `tests/unit/plugin-upload.test.js` to expect the tagging header.

### 2.3 Delete stored file data when the project is deleted

`controllers/projectController.js` `deleteProjectById` deletes the DynamoDB item only. Ingest keeps a second copy at `projects/{projectId}/figma-snapshots/{snapshotId}/`.

On project delete:

1. List and delete `projects/{projectId}/plugin-uploads/` and `projects/{projectId}/figma-snapshots/`.
2. Delete related inventory and job rows for that project if those tables are keyed by project id.
3. Return only after the S3 deletes succeed, or return a clear failure. A success response with objects still in the bucket makes the privacy policy false.
4. Cover it with a unit test that seeds both prefixes in the mock S3 map and asserts they are gone.

Token deletion is already implemented: `charisol-design-system-fe/components/settings/ConnectedPlugins.tsx` lists tokens and calls `DELETE /api/plugin/tokens/:tokenId`. Mention that screen by name in the policy. Confirm the screen is linked from account settings before calling it the user-facing control.

### 2.4 Deploy and check

- Deploy the production API stack and the data stack.
- Presign one file, PUT it with the tagging header from a throwaway script, confirm the object tag is `strata-plugin=pending`, complete the upload, confirm the tag is `ready`.
- Delete a disposable project and confirm both prefixes are empty.

**Done when:** production CORS, the 7-day pending rule, and project-delete cleanup are deployed. The privacy policy is written after this, so every sentence describes production.

---

## Phase 3 — Rewrite the policy and show it during sign-in

### 3.1 One policy, two copies that match

Replace `charisol-figma-plugin/PRIVACY_POLICY.md` and add `charisol-design-system-fe/app/privacy/page.tsx` so `https://strata.charisol.io/privacy` renders the same text.

The page states:

- Operator: Charisol. Contact: `support@charisol.io`.
- The plugin reads the open Figma file only after the user runs it, then sends the data below.
- API host from phase 1, and the data in each call (device code, plugin token, project list, asset hashes, file names, content types, sizes, upload id). Design-file bytes go to S3, not through the API.
- S3 host from phase 0. Objects are private. Contents: `manifest.json` (nodes, text, styles, variables, components, prototype links) and exported images and SVGs.
- Presigned URLs expire after 1 hour. That is the upload window.
- Objects tagged pending expire after 7 days. A completed snapshot stays until the project owner deletes the project in Charisol, which deletes `plugin-uploads/` and `figma-snapshots/` for that project.
- Plugin tokens last 90 days from last use. Revoke them from Connected Plugins on `https://strata.charisol.io/profile`. Logging out of the plugin also clears the token stored on that computer.
- Charisol account required. No ads, no sale of design data.

Update `TERMS_OF_SERVICE.md` the same way: Charisol as operator, `support@charisol.io`, `https://strata.charisol.io`. Remove `support@strata.design`, `privacy@strata.design`, `api.strata.design`, and `strata.design`.

### 3.2 Show it before connect

On `LoginScreen`, under the device code, add two links opened with `OPEN_EXTERNAL_URL`:

- Privacy policy → `https://strata.charisol.io/privacy`
- Create or open a Charisol account → `https://strata.charisol.io/plugin/connect`

The connect page at `app/plugin/connect/page.tsx` already approves the device code. No API change there.

**Code done. Production URL still required.**

Written and checked on 28 Oct 2026:

- `PRIVACY_POLICY.md` and `TERMS_OF_SERVICE.md` name Charisol, `support@charisol.io`, and `https://strata.charisol.io`. A search of the plugin repo finds no `strata.design`.
- `app/privacy/page.tsx` renders the same facts. Local `http://localhost:3456/privacy` returned 200 with title "Privacy Policy | Strata", both production hosts, the 7-day pending rule, project deletion, and the profile revoke link. The footer Privacy link points at `/privacy`.
- The connect screen always shows "Create or open a Charisol account" (`https://strata.charisol.io/plugin/connect`) and "Privacy policy" (`https://strata.charisol.io/privacy`), both opened with `OPEN_EXTERNAL_URL`. Plugin typecheck and `npm run build` passed. The production bundle contains both URLs and no `strata.design`, `api-pr-`, `localhost`, or `Failed to fetch`.

`https://strata.charisol.io/privacy` will not show this page until the frontend is deployed. The retention sentences stay false in production until the phase 2 API and data stacks are deployed.

---

## Phase 4 — Reviewer test, then the Community form

**Stopped on 28 Oct 2026. Not submitted.**

The form step says to stop while phase 2 is still off production. That is still true:

- Live `OPTIONS /v1/plugin/auth/device` with `Origin: null` returns 204, `access-control-allow-origin: *`, and `access-control-allow-credentials: true`.
- `https://strata.charisol.io/privacy` returns 404. The frontend privacy page is uncommitted on `main`.
- Phase 2 backend edits are uncommitted on `Fix-figma-plugin-issues`.
- Plugin commits are on `fix/phase2-asset-export-validation`, which has no upstream, so GitHub Actions has not uploaded `figma-plugin-dist-*` for this build.
- Figma desktop is not installed on this machine, so the reviewer path was not run.

The local production bundle does pass the 4.1 content checks: `dist/ui.html` contains the production API host, `strata.charisol.io`, and the privacy URL, and it does not contain `api-pr-`, `localhost`, or `Failed to fetch`. `allowedDomains` is the two production hosts. That check is on a laptop build. Reviewers still install the CI artifact after these commits are pushed.

### 4.1 Build the artifact reviewers will run

Publish from the CI artifact (`figma-plugin-dist-*`), not from a developer `dist/` folder.

On that zip, confirm:

- `dist/ui.html` contains `api-production-charisol-design-system-be.aws.charisol.io` and `strata.charisol.io`.
- It contains neither `api-pr-` nor `localhost`.
- `manifest.json` `allowedDomains` is the two hosts from phase 0 and 1.1.

### 4.2 Run the path Figma will run

Use a Figma file the reviewer account has not opened before. Install this build.

1. Open the plugin. The connect screen shows a confirmation code. The UI does not contain `Failed to fetch`.
2. Open `https://strata.charisol.io/plugin/connect`, approve the code, and land on the project list.
3. Export a file with one frame and one image. The progress view reaches complete.
4. In the browser network log, or from the backend log, copy the S3 PUT host and match it to `allowedDomains`.
5. Turn the network off and open the plugin again. The UI shows the connection sentence from phase 1.3.
6. Delete the disposable project and confirm the snapshot prefix is gone.
7. Revoke the token from Connected plugins and confirm the plugin asks to connect again.

Fix anything that fails and repeat from step 1. Resubmit only after this pass.

### 4.3 Fill the Community details field

The bucket host below is the phase 0 result. Keep this aligned with the privacy page. If phase 2 is not in production yet, stop and finish phase 2 first.

```
Strata Design System is operated by Charisol. The plugin uses two production destinations, both run by Charisol in AWS us-east-1.

1. https://api-production-charisol-design-system-be.aws.charisol.io
   Production Charisol API on Amazon API Gateway.
   Sent: device-login code, bearer plugin token, project-list request, SHA-256 asset hashes, file names, content types, file sizes, and an upload-complete notice.
   Design-file bytes are not sent to this API.
   Plugin tokens are stored as a hash, last 90 days from last use, and can be revoked from Connected Plugins at https://strata.charisol.io/profile. Device codes expire after 10 minutes.

2. https://production-charisol-design-system-be-project-data.s3.us-east-1.amazonaws.com
   Production Charisol project bucket on private Amazon S3. The plugin uploads with a presigned PUT URL that expires after 1 hour.
   Sent: manifest.json (node structure, text, styles, variables, components, and prototype links from the open file) and the exported images and SVGs from that file.
   Unfinished uploads are tagged and deleted after 7 days. A completed snapshot is kept until the project owner deletes the project in Charisol. Deleting the project deletes these objects. Access is limited to that project's members and Charisol operators. The bucket is not public.

A Charisol account is required: https://strata.charisol.io
Support and privacy requests: support@charisol.io
Privacy policy: https://strata.charisol.io/privacy
```

On the security disclosure form, answer with the same facts:

- Backend is hosted by Charisol on AWS.
- Data from the Figma plugin API is stored in the private project bucket above, until the project owner deletes the project.
- Authentication is the device-code flow. The plugin does not ask for a password. Tokens are hashed at rest and revocable.
- The listing description states that a Charisol account is required and links the privacy policy and `support@charisol.io`.

Also say in the description, in one or two sentences: the plugin reads the open file, its variables, and team-library data only after the user runs it, then syncs that inventory to the Charisol project they pick.

---

## Work split

| Track | Can start | Blocks resubmit |
|---|---|---|
| Phase 0 S3 hostname | Done | Host is recorded. Phase 1 still removes the wildcards. |
| Phase 1 plugin | Done | Production build and allowlist are in the repo. Publish that build, not an older `dist/`. |
| Phase 2 backend | Code done. Deploy still required. | Yes. The 7-day and delete sentences are false until this is in production. |
| Phase 3 policy page | Code done. Frontend deploy still required. | Yes. `https://strata.charisol.io/privacy` must load this page, and phase 2 must be in production before the retention sentences are true. |
| Phase 4 test and form | Stopped. Phase 2 and the privacy page are not in production, and there is no CI artifact yet. | This is the resubmit. Do not fill the Community form until the gate below is true. |

Phase 1 and phase 2 can proceed together. Phase 4 starts only when both are done.

---

## Files

| File | Change |
|---|---|
| `charisol-figma-plugin/manifest.json` | Two production domains, trailing slash, `devAllowedDomains`, `reasoning` |
| `charisol-figma-plugin/esbuild.js` | Refuse pull-request and localhost URLs outside watch mode |
| `charisol-figma-plugin/.env.example` | Production `BACKEND_API_URL` and `FRONTEND_URL` |
| `charisol-figma-plugin/src/services/api.ts` | Production fallback. Remove password login and token-paste shortcut |
| `charisol-figma-plugin/src/services/errors.ts` | User-facing network errors |
| `charisol-figma-plugin/src/components/LoginScreen.tsx` | Device code, policy link, mapped errors |
| `charisol-figma-plugin/src/components/SettingsModal.tsx` | Remove from the Community UI |
| `charisol-figma-plugin/src/components/Header.tsx` | Remove settings and IPC buttons |
| `charisol-figma-plugin/src/uploader/uploadOrchestrator.ts` | Send presign `headers` on PUT |
| `charisol-figma-plugin/PRIVACY_POLICY.md`, `TERMS_OF_SERVICE.md` | Charisol, real hosts, retention, deletion |
| `charisol-figma-plugin/scripts/assert-community-build.js` | Fail the build if the bundle or manifest still points at a dead or wildcard host |
| `charisol-design-system-be/cdk/lib/charisol-design-system-stack.ts` | Explicit CORS origins, credentials off |
| `charisol-design-system-be/cdk/lib/data-stack.ts` | 7-day lifecycle on `strata-plugin=pending` |
| `charisol-design-system-be/controllers/pluginController.js` | Sign the pending tag; retag on complete |
| `charisol-design-system-be/controllers/projectController.js` | Delete both S3 prefixes with the project |
| `charisol-design-system-fe/app/privacy/page.tsx` | Public policy page |

---

## Resubmit gate

All of these are true on the artifact about to be published:

- [ ] Open plugin shows a device code.
- [ ] Offline open shows the connection sentence, and the screen does not contain `Failed to fetch`.
- [ ] Export PUT host equals the single S3 domain in `manifest.json`.
- [ ] `allowedDomains` has no wildcard.
- [ ] `https://strata.charisol.io/privacy` loads and matches the details field.
- [ ] Deleting a test project removes `plugin-uploads/` and `figma-snapshots/` for that project.
- [ ] Revoking the token in Connected plugins signs the plugin out.
- [ ] `support@charisol.io` is a mailbox the team reads.
- [ ] The listing says a Charisol account is required.
