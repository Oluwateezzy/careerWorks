# Figma Community rejection: Strata Design System

**Date:** 28 October 2026
**Listing:** Strata Design System (plugin id `1689622748009578334`)
**Decision:** Rejected. Review was not completed.

Figma’s email has two requirements, and both have to be true before a resubmit:

1. The Community **details** section must explain, for every API and S3 destination listed on the plugin, what data is sent, who operates the backend, whether that backend is production, how long uploaded data is kept, and how a user can delete it.
2. The plugin must not show a **Failed to fetch** error when a reviewer opens it.

The in-repo privacy policy does not satisfy (1). The build that is currently in `charisol-figma-plugin/dist/` fails (2).

---

## 1. What Figma is asking for

“Details section” is the network-access and security disclosure on the Community publish form. Those domains are also printed on the public plugin page. Reviewers compare that text with `manifest.json` → `networkAccess.allowedDomains`.

The submitted manifest lists four destinations:

| Listed pattern | What a reviewer can conclude |
|---|---|
| `https://api-production-charisol-design-system-be.aws.charisol.io/v1` | One production API, path `/v1` only |
| `https://*.aws.charisol.io` | Any host under `aws.charisol.io`, including dead pull-request stacks |
| `https://*.s3.amazonaws.com` | Any bucket on the legacy global S3 hostname |
| `https://*.amazonaws.com` | Any single-label `*.amazonaws.com` host |

The form was not filled with a per-destination explanation, so review stopped. The privacy policy in the repo describes a different service (`https://api.strata.design`) and does not answer retention or deletion.

---

## 2. Findings

### 2.1 The published build calls a host that does not exist

This is the Failed to fetch error.

| Item | Evidence |
|---|---|
| Build-time API | `charisol-figma-plugin/.env` sets `BACKEND_API_URL` to `https://api-pr-263-charisol-design-system-be.aws.charisol.io/v1`. `esbuild.js` inlines that value. |
| Bundle actually shipping | `dist/ui.html` contains `api-pr-263-…` and the settings copy for `api-pr-256-…`. It does not contain the production host. |
| DNS on 28 Oct 2026 | `api-pr-263-…` and `api-pr-256-…` do not resolve. |
| Production, same request | `POST https://api-production-charisol-design-system-be.aws.charisol.io/v1/plugin/auth/device` returns HTTP 200 and a device code. |
| What the user sees | `LoginScreen` renders `err.message`. A DNS failure from `fetch` is the string `Failed to fetch`. |

`https://*.aws.charisol.io` lets Figma send the request. The request then dies at DNS. The wildcard hides the mistake from the allowlist and still breaks the plugin.

The settings screen makes it worse. `SettingsModal.tsx` tells the user the allowed API is the dead PR-256 host, and it lets them type any URL. A reviewer who follows that hint gets the same error.

`src/services/api.ts` still falls back to `http://localhost:5000/v1` when `BACKEND_API_URL` is missing. Localhost is not in `allowedDomains` or `devAllowedDomains`, so a build without the env var is also blocked inside Figma.

### 2.2 The API allow pattern does not cover the routes the plugin calls

Figma treats a pattern with no trailing slash as a single resource. `…/v1` allows that exact path and blocks `…/v1/plugin/auth/device`, `…/v1/project/all`, and the upload routes.

The wildcard currently covers those routes. Once the wildcard is removed (it should be), the production entry needs a trailing slash: `https://api-production-charisol-design-system-be.aws.charisol.io/v1/`.

### 2.3 S3 uploads are aimed at a host the allowlist does not name

Presigned PUTs are signed by `@aws-sdk/client-s3` `^3.1033.0` and `@aws-sdk/s3-request-presigner` in `us-east-1` (`utils/s3.js`). That SDK signs virtual-hosted regional URLs:

`https://<bucket>.s3.us-east-1.amazonaws.com/<key>`

`https://*.s3.amazonaws.com` matches `bucket.s3.amazonaws.com` only. `https://*.amazonaws.com` matches one label (`s3.amazonaws.com`), not `bucket.s3.us-east-1.amazonaws.com`.

After login works, asset and `manifest.json` PUTs fail inside Figma with the same `Failed to fetch` string. `uploadOrchestrator.ts` surfaces it as `S3 upload failed for manifest.json: Failed to fetch`.

The CDK bucket name is `${stackId}-project-data` (lowercased) in `cdk/lib/data-stack.ts`. Confirm the live hostname from one real presigned URL before putting it in the manifest. Do not guess it into the Community listing.

S3 CORS in CDK allows `*` origins and `PUT`, so the bucket will accept the plugin once the hostname is on the allowlist.

### 2.4 The privacy policy describes a service this plugin does not use

`charisol-figma-plugin/PRIVACY_POLICY.md` (effective 2 October 2026) says:

- The API is `https://api.strata.design`.
- Contact is `privacy@strata.design` and `https://strata.design`.
- Operator name is “Strata”, with no legal entity.

The running service is:

- API: `https://api-production-charisol-design-system-be.aws.charisol.io` (API Gateway, custom domain on `aws.charisol.io`).
- App: `https://strata.charisol.io` (the device-code `verification_uri` returned by production).
- Package author on the backend: Ancla Technologies. The product is operated as Charisol.
- Terms of service repeat `support@strata.design` and `https://strata.design`.

Figma also requires a working support contact. A policy that names hosts the plugin never calls will be read as inaccurate.

The policy is not linked from the login screen. Figma’s security disclosure asks for the same storage explanation in the privacy policy that users see during authentication.

### 2.5 Retention and deletion are not what the policy implies, and they are not written down

The policy says uploads use 1-hour presigned URLs. That is the **upload window**, not how long the files stay.

| Data | Where it lives | How long it actually stays | How a user can delete it |
|---|---|---|---|
| Device code | DynamoDB `PLUGIN_DEVICE#…` | 10 minutes (`DEVICE_CODE_EXPIRY_SECONDS`) | Expires on its own |
| Plugin access token | DynamoDB, hash only; raw token in `figma.clientStorage` | Sliding 90 days (`TOKEN_SLIDING_EXPIRY_DAYS`) | `DELETE /v1/plugin/tokens/:tokenId` while signed in on the web app. The plugin UI has no revoke control. |
| Email and password, if the user picks “Use password instead” | Sent to `POST /v1/auth/login` or `/v1/auth/signin` | Account lifetime on the Charisol user store | Not described anywhere in the plugin |
| Project list, file names, content types, byte sizes, SHA-256 hashes | API requests | Request logs (API Gateway data tracing is enabled on the stage) | Not described |
| `manifest.json` and exported images | S3 `projects/{projectId}/plugin-uploads/{uploadId}/…` | Production bucket `removalPolicy` is `RETAIN` and versioning is on. The CDK lifecycle rule expires only objects tagged `strata-source=pending` after 7 days. Plugin presign does **not** set that tag. Completed plugin uploads are kept until someone deletes them. | Project owner can call `DELETE /v1/project/:queryId` or `DELETE /v1/projects/:projectId`. There is no plugin control, and no privacy-policy sentence, that says this. |

`scripts/configure-s3-lifecycle.js` would expire every object under `projects/` after 7 days. That script is not the CDK rule. Do not tell Figma uploads expire in 7 days unless that rule is actually deployed and scoped to plugin uploads.

### 2.6 The plugin shows developer errors to reviewers

Review guideline: do not use developer error messages as the user-facing copy.

`Failed to fetch` is `TypeError` from `fetch`, rendered in `LoginScreen`, `ProjectPickerScreen`, and `ExportProgressView`. The rejection email quotes that string because it was on screen.

### 2.7 API Gateway CORS is an invalid combination

Live preflight on `OPTIONS /v1/plugin/auth/device` (28 Oct 2026):

- `access-control-allow-origin: *`
- `access-control-allow-credentials: true`

That pair is set in `charisol-design-system-stack.ts` (`allowOrigins: Cors.ALL_ORIGINS` with `allowCredentials: true`). Browsers reject it when the request is credentialed. The plugin’s `fetch` calls do not set `credentials: 'include'`, so this is not the current failure. It is the next failure mode once the host resolves, if Figma’s iframe sends a credentialed request.

The POST itself, when it reaches Express, reflects `Origin: null` correctly. The preflight never gets that far: API Gateway answers OPTIONS on its own.

### 2.8 Other review risks that were not in this email

These are in the same guidelines and are likely on a second pass:

- A Strata account is required. The Community description has to say that before install.
- Password login inside the plugin is a self-hosted credential form. The security disclosure asks how those credentials are stored and protected. The device-code flow already exists and should be the only Community path. `api.ts` also treats a long email string, or one starting with `pt_`, as an access token and skips the server.
- `documentAccess` is `dynamic-page` and `permissions` includes `teamlibrary`. The description should say the plugin reads the open file, local variables, and team library data only after the user runs it.
- `FRONTEND_URL` in the plugin `.env` is `http://localhost:3000`. Production returns `https://strata.charisol.io/plugin/connect`. A build that inlines localhost sends reviewers to their own machine.

---

## 3. What data actually moves

Use this as the source for the Community details text. It matches the code, not the current privacy policy.

**Charisol API** (`https://api-production-charisol-design-system-be.aws.charisol.io`), operated by Charisol on AWS API Gateway in `us-east-1`. This is the production service.

| Call | Data sent |
|---|---|
| `POST /v1/plugin/auth/device` and `/v1/plugin/auth/token` | No design file contents. The server stores a short-lived device code and, after approval, a hashed plugin token bound to the Charisol user. |
| `POST /v1/auth/login` (password fallback only) | Work email and password. |
| `GET /v1/plugin/me`, `GET /v1/project/all` | Bearer plugin token. Response includes the user’s email and name, and the projects they can access. |
| `POST …/plugin/check-assets` | SHA-256 hashes of exported assets. |
| `POST …/plugin/presign-upload` | Project id, file names, content types, sizes. Response is 1-hour S3 PUT URLs. |
| `POST …/plugin/complete-upload` | Upload id and the list of assets that failed. |

**S3** (exact bucket host to be copied from a live presigned URL), operated by Charisol in the same AWS account. Objects are private (`BlockPublicAccess.BLOCK_ALL`). The plugin PUTs directly; the API does not proxy file bytes.

| Object | Data |
|---|---|
| `manifest.json` | Normalized inventory of the open file: node tree, text, styles, variables, components, prototype reactions, file metadata. |
| Image and SVG objects | Renders and assets exported from that file (PNG, JPEG, SVG, WebP, GIF). |

No data is sold. No third-party model provider is called from the plugin.

---

## 4. Options

### Option A — Rewrite the Community form only

Paste a privacy paragraph into the publish form and resubmit the current build.

This answers the sentence in the email and leaves the plugin broken. Reviewers will see Failed to fetch again, because `dist/ui.html` still calls PR-263. Do not do this on its own.

### Option B — Point the build at production and keep the wildcards

Set `BACKEND_API_URL` to the production `/v1` base, set `FRONTEND_URL` to `https://strata.charisol.io`, rebuild, and resubmit with a written disclosure.

Login can succeed. Upload can still throw Failed to fetch, because the S3 host is not on the allowlist. The wildcards also leave the disclosure vague: reviewers asked what is sent to “the listed API and S3 destinations”, and `*.amazonaws.com` is not one destination.

### Option C — Narrow the allowlist, fix the errors, and make the policy match production

This is the resubmit that matches both sentences in the email.

1. Rebuild the Community artifact with production URLs only. Remove the plugin settings field that edits the API base. Put `http://localhost:5000` in `devAllowedDomains`, not `allowedDomains`.
2. Replace `networkAccess.allowedDomains` with the production API origin (trailing slash on `/v1/`) and the single S3 bucket host taken from a live presigned URL. Drop `*.aws.charisol.io`, `*.s3.amazonaws.com`, and `*.amazonaws.com`.
3. Map network failures in the UI to a sentence a reviewer can act on, for example that the plugin could not reach the Charisol service and the user should check their connection. Keep `Failed to fetch` out of the UI.
4. Change API Gateway CORS to echo `https://www.figma.com` and `null`, and stop sending `Access-Control-Allow-Credentials: true` together with `*`.
5. Replace `PRIVACY_POLICY.md` and `TERMS_OF_SERVICE.md` so the operator, hosts, support email, retention, and deletion match section 3. Host that policy on `https://strata.charisol.io` and link it on the connect screen and in the Community listing.
6. Pick one retention behavior and implement it before describing it:
   - **Recommended:** tag plugin uploads, expire unfinished uploads after 7 days, keep a completed inventory until the project owner deletes the project, and say that in the policy. Token revoke stays `DELETE /v1/plugin/tokens/:tokenId`.
   - **Honest minimum:** say completed uploads are kept until the project owner deletes the project, and name that control. Do not claim a 7-day expiry that the presign path does not tag.
7. Ship device-code sign-in only in the Community build. Leave password sign-in out of that build so the security form does not have to defend a password typed into the Figma iframe.
8. Before resubmit, install the production build in a clean Figma file, confirm the connect screen shows a code (no raw fetch error), approve it on `strata.charisol.io`, and upload a small file. Confirm the PUT host equals the host in the manifest.

### Option D — Stop sending file data off Figma

Drop network access, or upload nothing until the user exports a file themselves. There is then no API or S3 destination to explain.

That removes the product. The plugin’s purpose is to send the inventory to Charisol. Option C is the path that keeps the product and passes this review.

---

## 5. Recommended sequence

Do option C. Order matters because a correct paragraph on a broken build is another rejection.

1. Confirm the live S3 hostname with one presigned PUT from production.
2. Fix manifest, env, CORS, error copy, and the privacy policy.
3. Decide retention and wire the lifecycle tag or the project-delete path so the policy is true.
4. Run the clean-file test in section 4, step 8.
5. Fill the Community details and the security disclosure with the text in section 6, using the confirmed bucket host.
6. Resubmit that build only.

---

## 6. Draft text for the Community details field

Replace `<BUCKET>` after checking a live presigned URL. Replace the support address with a mailbox Charisol actually reads.

```
Strata Design System is operated by Charisol. The plugin talks to two production destinations, both run by Charisol in AWS us-east-1. It does not talk to any other API or bucket.

1. https://api-production-charisol-design-system-be.aws.charisol.io
   This is the production Charisol API (Amazon API Gateway).
   The plugin sends: a device-login code, a bearer plugin token, the signed-in user's project list request, SHA-256 hashes of exported assets, file names, content types, file sizes, and an upload-complete notice.
   It does not send the design file to this API. File bytes go to S3.
   Plugin tokens are stored as a hash and expire after 90 days of disuse. The user can revoke a token from their Charisol account (DELETE /v1/plugin/tokens/:tokenId). Device codes expire after 10 minutes.

2. https://<BUCKET>.s3.us-east-1.amazonaws.com
   This is the production Charisol project bucket (private Amazon S3). The plugin uploads with a presigned PUT URL that expires after 1 hour.
   The plugin sends: manifest.json (node structure, text, styles, variables, components, and prototype links from the open file) and the exported images and SVGs for that file.
   Objects stay in the project until the project owner deletes the project in Charisol. Unfinished uploads expire after 7 days. Access is limited to that project's members and Charisol operators. The bucket is not public.

Support and privacy requests: <real support email>
Privacy policy: https://strata.charisol.io/privacy
```

Only publish the 7-day sentence after unfinished uploads are actually tagged and expired. If that rule is not in production yet, say uploads are kept until the project owner deletes the project.

---

## 7. Checks before resubmit

- [ ] `dist/ui.html` contains the production API host and does not contain `api-pr-` or `localhost`.
- [ ] `allowedDomains` is the production API (with trailing slash) plus one S3 bucket host.
- [ ] A clean Figma session shows a device code, not `Failed to fetch`.
- [ ] A small export PUT succeeds, and the PUT host matches the manifest.
- [ ] The privacy policy URL loads and names Charisol, the same two hosts, retention, and deletion.
- [ ] The support email in the listing receives mail.
- [ ] The Community description says a Charisol account on `https://strata.charisol.io` is required.
