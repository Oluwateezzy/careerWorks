# Strata Live Prototype: Detailed Developer Handoff

## 1. Purpose and scope

This document is the implementation handoff for the Strata live-prototype workflow. A developer should be able to select any task below, understand the surrounding architecture, identify the files and contracts involved, implement the task independently, and know exactly how to validate it.

The workflow starts with a Figma file or page URL. Strata inventories the selected Figma content, renders the selected frames or components, lets the user review the selection, generates a deterministic responsive React prototype, creates or updates a GitHub repository through a pull request, deploys the review branch to a Strata-managed Vercel project, and exposes controlled preview links.

This plan does not replace the existing Figma token-import flow. It adds a separate Prototype workflow inside the existing project dashboard. Phase One must not reorganize, reclassify, rename, or otherwise change the existing design-system token model.

## 2. How to use this document

Start with the phase that matches the assigned work. Read the shared conventions in section 6 before changing code. Then read the selected task's goal, files, inputs, implementation steps, validation steps, and completion criteria.

Each task is intentionally self-contained. A task may depend on an earlier task, but the dependency is stated explicitly. Do not skip a dependency when the task needs its schema, endpoint, worker, or UI state.

When a task adds a backend route, update the OpenAPI source and run the backend contract checks. When a task adds frontend code, run the frontend lint, typecheck, test, and build commands. When a task changes infrastructure, synth the CDK stack and inspect the generated IAM permissions.

The current repository root is:

`/Users/oluwatobiloba/Desktop/charisol/design_system`

The plan file is:

`docsplan/1789987065356-strata-live-prototype-plan.md`

## 3. Product decisions and boundaries

The following decisions are fixed for this implementation.

The Prototype workflow is a new tab in the existing project dashboard. It must not replace the existing Figma token-import modal, token editor, branch workflow, snapshot publishing workflow, or connected-surface workflow.

The user supplies a Figma file or page URL and selects an existing encrypted Figma credential. The workflow must not ask the user to paste a raw Figma token into the Prototype UI.

The workflow has explicit stages. First inventory the Figma source. Then render selected nodes. Then let the user review the inventory and selection. Then explicitly generate the prototype. A generation job must not start merely because an inventory completed.

The selection UI displays pages and nodes as a tree. A selected top-level frame or component becomes one generated route or page. Nested children belong to that page unless the user explicitly selects them as separate top-level outputs.

Generation is deterministic. The normalized inventory and pinned template are the source of truth. AI may help with naming or copy, but it must not make layout, routing, component hierarchy, or interaction decisions in the core generation path.

Explicit Figma prototype connections are preserved. When a Figma connection is missing or cannot be represented, the generator creates a deterministic fallback route map.

The generated prototype uses React 19, TypeScript, and Vite. It must render at desktop, tablet, and mobile sizes.

The generated repository contains a self-contained design layer. That layer is isolated so a future Strata design-system adapter can replace it without rewriting the generated routes and interaction code.

The generated prototype includes navigation, component states, forms, and mock data. It does not include production authentication, payments, business rules, database access, or external production integrations.

New repositories and existing repositories are both supported. New repositories default to private. The template repository is private, owned by Strata, and referenced by a pinned commit.

GitHub access uses a GitHub App. The App uses fine-grained permissions and short-lived installation tokens. User PATs are not accepted or stored.

Each generation is pushed to a dedicated branch. A pull request is opened or updated for the generation. Generated-file ownership metadata and file-level diffs protect manual edits.

Vercel is the Phase One deployment provider. Strata owns the Vercel team and project. Users do not need their own Vercel account.

A generation branch is deployed immediately for review. The stable Strata URL changes only after a PR is approved or merged, or after an explicit owner publish action.

Preview links are unlisted, revocable, and non-expiring. The path shape is `/project/:projectId/preview/:token`. The system stores only a hash of the token and audits access.

Long-running work is asynchronous. API handlers enqueue jobs. Step Functions coordinates Fargate or ECS workers. DynamoDB stores state and metadata. S3 stores large inventories, renders, generated bundles, screenshots, and validation reports.

Vercel is implemented first behind a deployment-provider interface. A Plexo adapter may be added later without changing the core job, artifact, or UI contracts.

Editors may start generation. Owners manage GitHub and Vercel connections, repository visibility, promotion, preview-token rotation, and deletion. Viewers may inspect approved previews.

## 4. Existing codebase map

### Frontend application

The frontend is a Next.js application using React 19, Chakra UI, and the existing project dashboard.

The main project page is:

`charisol-design-system-fe/app/project/[id]/page.tsx`

The project page currently renders `ProjectDetailsScreen`. That component owns the project ID, active section, project loading, branch query parameter, and project data synchronization.

The project navigation is:

`charisol-design-system-fe/components/sections/project/ProjectSidebar.tsx`

`ProjectSection` currently includes `brand-bible`, `handoff`, `tokens`, `components`, `settings`, `collaboration`, and `branch-publish`. Add `prototype` to this union and add one navigation item with an icon, label, active state, and role-based visibility.

The settings UI is:

`charisol-design-system-fe/components/sections/project/ProjectSettingsSection.tsx`

Use this component for owner-only repository and deployment settings, or create a dedicated settings subsection and link to it from the Prototype tab. Do not remove the existing project name, visibility, collaboration, webhook, migration, or deletion controls.

The project API client is:

`charisol-design-system-fe/services/project.service.ts`

Add prototype methods here only if they fit the existing service ownership. Otherwise create a dedicated `prototype.service.ts`. Keep all authenticated calls on the existing `/api` BFF boundary and preserve cookie credentials.

The existing Figma BFF routes are:

`charisol-design-system-fe/app/api/figma/connect/route.ts`

`charisol-design-system-fe/app/api/figma/extract-preview/route.ts`

These routes proxy to the backend and forward the browser cookie. New Prototype BFF routes should follow the same pattern. They must not expose provider credentials to browser code.

The backend API base URL helper is:

`charisol-design-system-fe/lib/api-config.ts`

Use it for server-side BFF requests. Do not add `/v1` twice.

The frontend package scripts are defined in:

`charisol-design-system-fe/package.json`

The relevant commands are `yarn lint`, `yarn typecheck`, `yarn test`, and `yarn build`.

### Backend application

The backend is an Express application deployed to AWS Lambda through the existing CDK stack.

The current Figma controller is:

`charisol-design-system-be/controllers/figmaController.js`

It already contains Figma connection, preview extraction, import, sync, mapping, and deprecated extraction endpoints. Reuse its URL parsing and Figma helper imports where appropriate. Do not turn the existing synchronous preview endpoint into the new long-running workflow.

The current route registration is:

`charisol-design-system-be/routes/v1/index.js`

Add new Prototype routes near the existing Figma and project routes. Every new route must have an authentication middleware and the appropriate project authorization middleware.

The current Figma utilities are:

`charisol-design-system-be/utils/index.js`

Reuse `parseFigmaApiAndDesignFileUri`, `buildParentMap`, `buildNodeIndex`, `collectRenderableNodes`, `collapseByRenderableIntoRelatedLayout`, and `renderNodesToImages` where their behavior matches the new workflow. The new inventory worker must not inherit the old depth-1 or first-page limitations.

The current project model is:

`charisol-design-system-be/models/Project.js`

It stores project metadata, versions, design-system data, and S3 pointers. Do not put large Figma inventories or generated bundles directly into this document.

The current profile model is:

`charisol-design-system-be/models/Profile.js`

It currently stores `figmaTokens[].tokenValue` as plaintext. This must be migrated to encrypted credential storage while preserving compatibility with the old import flow during rollout.

The current snapshot publisher is:

`charisol-design-system-be/utils/publishSnapshot.js`

It publishes design-system snapshots to S3 and CloudFront. Reuse the ideas of immutable versions, manifests, retention, and CDN invalidation only where they fit. Do not treat a generated prototype deployment as a design-system snapshot.

The backend package scripts are defined in:

`charisol-design-system-be/package.json`

Backend tasks must run `yarn lint`, `yarn typecheck`, `yarn test:unit`, `yarn test:integration`, `yarn test:functional`, `yarn build`, `yarn docs:validate`, and `yarn audit:refresh-spec` according to the backend AGENTS instructions.

The OpenAPI source is:

`charisol-design-system-be/docs/openapi.js`

Every new backend route must be represented there. The backend AGENTS instructions require route and OpenAPI changes to land together.

The current CDK stack is:

`charisol-design-system-be/cdk/lib/charisol-design-system-stack.ts`

It currently provisions the API Lambda, API Gateway, CloudFront, DynamoDB access, S3 access, and related environment variables. Extend it for Step Functions, Fargate or ECS, Secrets Manager, KMS, dead-letter queues, alarms, and worker IAM permissions.

The current Lambda timeout is 30 seconds. The API Lambda may enqueue work but must not perform Figma inventory, rendering, generation, GitHub, or Vercel work synchronously.

## 5. Target architecture

### 5.1 End-to-end request flow

The normal flow is:

1. The browser opens the Prototype tab in an existing project.
2. The frontend calls a Next.js BFF route with the browser cookie.
3. The BFF forwards the request to the authenticated backend API.
4. The backend validates project membership, role, input, and idempotency.
5. The backend creates or reuses a durable job and returns HTTP 202.
6. Step Functions starts the appropriate worker sequence.
7. Workers read and write small state records in DynamoDB and large artifacts in S3.
8. The frontend polls the job endpoint and updates the UI.
9. The user reviews the inventory and selection.
10. The user explicitly starts generation.
11. The generation worker creates a deterministic bundle and validation report.
12. The GitHub worker creates or updates a branch and pull request.
13. The deployment worker deploys the review branch to Vercel.
14. The user or an approved merge promotes the artifact to the stable Strata URL.
15. The preview gateway resolves hashed preview tokens and serves the authorized artifact.

### 5.2 Component responsibilities

The Next.js Prototype tab owns the user-facing workflow. It displays source connection, inventory progress, page and node selection, render status, generation status, repository state, deployment state, and preview controls.

The Next.js BFF routes own cookie forwarding and backend URL construction. They do not implement provider authentication or long-running work.

The backend API layer owns authentication, project authorization, input validation, idempotency, job creation, job queries, artifact metadata, and provider configuration reads.

The Step Functions workflow owns orchestration. It retries transient provider failures, honors Figma rate limits, handles cancellation, records phase transitions, and routes terminal failures to a recoverable state.

The Fargate or ECS workers own provider calls and artifact production. Each worker should be independently testable and should accept a job reference rather than secrets or large payloads.

The provider clients own protocol details for Figma, GitHub, Vercel, and the future Plexo adapter. They return normalized results and normalized errors to the workers.

The S3 artifact store owns immutable source inventories, rendered PNGs, generated ZIP bundles, manifests, screenshots, validation reports, and provider metadata.

The DynamoDB store owns small indexes, state machines, hashes, pointers, timestamps, and audit records.

### 5.3 Job state machine

Use a durable state field and a separate phase field. The exact enum names must be finalized in task 0.1, but the implementation must support at least the following states:

`queued`

`validating`

`inventory_running`

`inventory_complete`

`render_queued`

`render_running`

`render_complete`

`generate_queued`

`generate_running`

`generate_complete`

`validate_running`

`github_queued`

`github_running`

`deploy_queued`

`deploy_running`

`ready_for_review`

`promotion_pending`

`promoted`

`failed`

`cancelled`

A job may stop after inventory or render completion until the user explicitly requests the next action. Do not automatically generate a prototype when inventory completes.

A job must be safe to poll repeatedly. Polling must not mutate the job except for updating last-access metadata when that is intentional.

A job must be safe to retry. Retrying a completed step must reuse the existing artifact when the idempotency key and input hash match.

A failed job must retain its partial artifacts and error report. A user or operator must be able to retry from the failed phase when the failure is transient.

### 5.4 DynamoDB entities

Use separate entities instead of embedding large prototype data in `Project`.

`PrototypeConfig` stores the project's Figma source reference, selected nodes, repository preferences, deployment preferences, stable preview state, and feature-flag state.

`PrototypeJob` stores workflow type, state, phase, progress, idempotency key, input hash, source version, selected node IDs, current artifact IDs, error details, timestamps, and retry metadata.

`FigmaInventory` stores inventory version, source file key, source page and node counts, schema version, warnings, truncation status, and the S3 inventory pointer.

`PrototypeArtifact` stores generation ID, template commit, input hash, output hash, manifest pointer, validation report pointer, ownership metadata pointer, repository reference, deployment reference, and promotion state.

`GitHubConnection` stores installation ID, repository owner, repository name, default branch, connection status, branch name, pull-request number, and last synchronization time. It must never store a GitHub installation access token.

`Deployment` stores provider name, provider deployment ID, branch, environment, internal URL, stable URL state, health status, promotion target, and rollback target.

`PreviewToken` stores only the token hash, project ID, artifact or deployment reference, active state, creation time, revocation time, access count, and last-access time.

`AuditEvent` stores actor, project, action, result, job ID, timestamp, and a redacted request reference. It must not store secrets or raw provider responses.

Use content hashes in object names for immutable artifacts. Use DynamoDB conditional writes for state transitions that must not be overwritten by an older worker.

### 5.5 S3 artifact layout

Use this layout unless task 0.4 defines a versioned replacement:

`projects/<projectId>/prototype/figma/<fileKey>/<inventoryVersion>/inventory.json`

`projects/<projectId>/prototype/figma/<fileKey>/<inventoryVersion>/metadata.json`

`projects/<projectId>/prototype/renders/<inventoryVersion>/<nodeId>@<scale>.png`

`projects/<projectId>/prototype/generated/<generationId>/bundle.zip`

`projects/<projectId>/prototype/generated/<generationId>/manifest.json`

`projects/<projectId>/prototype/generated/<generationId>/ownership.json`

`projects/<projectId>/prototype/validation/<generationId>/report.json`

`projects/<projectId>/prototype/validation/<generationId>/screenshots/<viewport>.png`

`projects/<projectId>/prototype/github/<connectionId>/<generationId>/patch.json`

`projects/<projectId>/prototype/deployments/<deploymentId>/metadata.json`

Do not put secrets, signed provider tokens, raw Figma credentials, or browser cookies in object keys, object metadata, or artifact contents.

## 6. Shared engineering conventions

### 6.1 API behavior

Use HTTP 202 for actions that enqueue work. Return a stable job ID, current state, and a short user-facing message.

Use HTTP 200 for reads and completed artifact metadata. Use HTTP 404 when a project, job, artifact, connection, or token does not exist. Use HTTP 409 when an idempotency key is reused with different input. Use HTTP 422 for a valid request that cannot be processed because the source is incomplete or incompatible. Use HTTP 429 when a provider rate limit should be retried by the client or worker. Use HTTP 5xx for infrastructure or provider failures that are not user-correctable.

Use one error envelope across new routes:

```json
{
  "status": false,
  "code": "FIGMA_INVENTORY_FAILED",
  "message": "We could not inventory the selected Figma content.",
  "details": {
    "retryable": true,
    "phase": "inventory"
  },
  "requestId": "req_..."
}
```

Do not return raw provider responses, stack traces, signed URLs, access tokens, cookies, or environment variables.

Every new route must be added to `charisol-design-system-be/docs/openapi.js`. Run the backend OpenAPI validation and refresh the snapshot when required by the backend AGENTS instructions.

### 6.2 Authentication and authorization

Reuse `authenticateUser` and `authorizeProjectAccess` from the backend middleware layer.

Viewers may read job status and approved preview metadata.

Editors may connect a Figma source, inventory, render, and generate.

Owners may manage GitHub and Vercel connections, create or connect repositories, change repository visibility, promote deployments, create or revoke preview tokens, and delete prototype resources.

Check authorization at the API boundary and again before a worker performs an external side effect. A job ID alone must not authorize a cross-project action.

### 6.3 Secrets and logging

Store Figma credentials, GitHub App private keys, GitHub webhook secrets, Vercel tokens, and preview-token signing material in AWS Secrets Manager or encrypted DynamoDB fields backed by KMS.

Never log a secret, authorization header, cookie, signed URL, raw provider response containing credentials, or raw preview token.

Log only allowlisted fields such as project ID, job ID, phase, provider, status code, retry count, artifact hash, and sanitized error code.

Add a redaction helper and use it at API and worker boundaries. Test the helper with representative secret-bearing payloads.

### 6.4 Idempotency and retries

Create an idempotency key from project ID, workflow type, source version, selected node IDs, generation options, and repository or deployment options.

Store the input hash with the job. If the same idempotency key arrives with a different input hash, return HTTP 409.

Use conditional writes for state transitions. A worker must not overwrite a newer job state with an older result.

Use bounded exponential backoff for transient provider failures. Honor Figma `Retry-After` headers. Use dead-letter handling for permanent failures.

A retry must reuse completed artifacts when the input hash and template commit are unchanged.

### 6.5 Generated code conventions

The generated repository must be a clean Vite React TypeScript project. It must install and build without access to Strata internal services.

Separate generated design tokens and styles from generated components, routes, state, mock data, and runtime manifest.

Use stable file names and deterministic ordering. Sort nodes, routes, tokens, and manifests before writing files.

Do not include timestamps in generated source files unless the timestamp is required in a non-content manifest. If a manifest contains generated time, exclude that field from the deterministic content hash.

Every generated file that Strata owns must be listed in `ownership.json`. Manual files must never be listed as generated-owned unless the user explicitly opts in.

### 6.6 Testing conventions

Frontend tasks should add unit or component tests beside the relevant code or in the existing test structure. Use the existing Jest and Testing Library setup.

Backend tasks should add unit and integration tests under the existing backend test structure. Use mocked provider clients for external services.

Worker tasks should expose pure functions where possible so tests do not require AWS, Figma, GitHub, or Vercel.

Generated artifact tasks must validate a clean generated repository with install, build, TypeScript, lint, route manifest, responsive screenshot, and interaction smoke checks.

Every task's validation section is part of its definition of done. Do not mark a task complete because the code compiles if its stated behavioral checks have not run.

## 7. Phase 0: Contracts, security, and workflow foundation

Phase goal

Phase 0 is the foundation for every later phase. Do not start Phase 1 until the shared contracts, models, security, and refactors are in place.

### Task 0.1: Define the prototype API contracts and state machine

Goal:

Create one source of truth for route shapes, job states, error codes, idempotency rules, and provider interfaces so frontend, backend, workers, and tests do not invent incompatible contracts.

Files and areas to inspect:

`charisol-design-system-be/docs/openapi.js`

`charisol-design-system-be/routes/v1/index.js`

`charisol-design-system-be/controllers/figmaController.js`

`charisol-design-system-fe/services/project.service.ts`

`charisol-design-system-fe/app/api/figma/connect/route.ts`

`charisol-design-system-fe/app/api/figma/extract-preview/route.ts`

Inputs:

The product decisions in section 3.

The existing backend response envelope.

The existing Next.js BFF proxy pattern.

The job state machine in section 5.3.

Implementation steps:

Create a versioned backend request and response type definition for each proposed Prototype endpoint.

Define the canonical endpoint paths. Use `/v1/projects/:projectId/prototype/...` for backend routes and matching `/api/prototype/...` BFF routes unless an existing routing convention requires a different prefix.

Define the job state and phase enums. Include queued, running, complete, failed, cancelled, ready-for-review, and promoted states.

Define the standard error codes for invalid source, missing credential, insufficient permission, job not found, idempotency conflict, provider rate limit, provider authentication failure, worker failure, artifact not found, and invalid preview token.

Define the idempotency key algorithm and the fields included in the input hash.

Define the provider interface methods for Figma inventory, GitHub repository operations, Vercel deployment operations, and the future Plexo adapter.

Add every new route to the OpenAPI source with request bodies, response schemas, authentication requirements, role requirements, and error responses.

Add a small contract test fixture that proves each route has a matching OpenAPI operation.

Validation:

Run `yarn docs:validate` in the backend.

Run the backend unit and integration tests.

Confirm every proposed route has an OpenAPI operation.

Confirm every job transition has a documented allowed next state.

Confirm the same idempotency key with identical input returns the existing job and with different input returns HTTP 409.

Completion criteria:

A developer can implement any later task using only the contracts defined here.

No new route is implemented without a matching OpenAPI entry.

### Task 0.2: Add encrypted Figma credential storage and migration

Goal:

Replace plaintext Figma token storage with encrypted per-user credentials while keeping the existing Figma import flow usable during migration.

Files and areas to inspect:

`charisol-design-system-be/models/Profile.js`

`charisol-design-system-be/controllers/profileController.js`

`charisol-design-system-be/controllers/figmaController.js`

`charisol-design-system-be/utils/figmaHelpers.js`

`charisol-design-system-be/cdk/lib/charisol-design-system-stack.ts`

Inputs:

The existing `Profile.figmaTokens` shape.

The existing `resolveFigmaTokenFromProfile` behavior.

The backend AGENTS security rules.

Implementation steps:

Add a versioned encrypted credential shape to the Profile model. Include credential ID, encrypted token, KMS key reference, creation time, update time, last-used time, and status.

Keep the legacy `figmaTokens` field during the migration window. Do not delete it in the first release.

Add a KMS-backed encryption and decryption helper. Keep plaintext values in memory only for the shortest possible time.

Update profile create, list, add-token, and remove-token behavior so the new credential shape is used without returning plaintext.

Update Figma resolution helpers to accept either a legacy token ID or a new credential ID. Preserve the existing public behavior where possible.

Add a migration script that reads each legacy plaintext token, encrypts it, writes the new credential, decrypts it for verification, marks the legacy item migrated, and records a migration audit event.

Make the migration resumable. If one profile fails, later profiles must still be processed and the failed profile must be reported.

Add a rollback path that can restore the legacy field from the encrypted value only during the controlled migration window.

Remove the legacy plaintext field only after the migration verification and rollback window are complete.

Validation:

Run a migration dry run against representative legacy profiles.

Run the migration against a staging copy and verify decrypted credentials work with Figma.

Search logs, test output, and error fixtures for plaintext token patterns.

Verify old Figma import and sync endpoints still work before and after migration.

Verify profile list and mapping endpoints never return the encrypted or plaintext token.

Run all backend checks required by the backend AGENTS instructions.

Completion criteria:

No new Figma workflow stores or logs plaintext credentials.

Existing Figma behavior remains compatible until the migration is explicitly completed.

### Task 0.3: Add role enforcement and audit events

Goal:

Make every Prototype action enforce the intended role and create an auditable record without exposing sensitive data.

Files and areas to inspect:

`charisol-design-system-be/middlewares/index.js`

`charisol-design-system-be/middlewares/authorizeProjectAccess.js`

`charisol-design-system-be/routes/v1/index.js`

`charisol-design-system-be/models/Project.js`

`charisol-design-system-be/models/helpers/dynamodb.js`

Inputs:

The role matrix in section 3.

The existing project authorization middleware.

Implementation steps:

Define Prototype-specific authorization helpers or route middleware for read, generate, integration management, promotion, token management, and deletion.

Apply viewer authorization to job and artifact reads.

Apply editor authorization to inventory, render, and generation starts.

Apply owner authorization to GitHub connection, repository creation, repository connection, deployment promotion, preview-token creation and revocation, and deletion.

Add audit event creation at the API boundary for each sensitive action.

Add audit event creation in workers for external side effects such as repository creation, PR creation, deployment, promotion, and rollback.

Ensure audit events include actor, project, action, result, job ID, and timestamp.

Ensure audit events never include secrets, raw provider responses, cookies, or preview tokens.

Validation:

Add authorization tests for owner, editor, viewer, member of another project, and unauthenticated user.

Add audit tests that verify allowed fields and redaction.

Run backend unit and integration tests.

Completion criteria:

A user cannot perform an action above their role.

A reviewer can reconstruct who started, promoted, revoked, or deleted a Prototype resource.

### Task 0.4: Add job and artifact infrastructure

Goal:

Provide durable job state, artifact metadata, idempotency, cancellation, and S3 storage primitives for all later workers.

Files and areas to inspect:

`charisol-design-system-be/models/Project.js`

`charisol-design-system-be/models/helpers/dynamodb.js`

`charisol-design-system-be/utils/s3.js`

`charisol-design-system-be/utils/index.js`

`charisol-design-system-be/cdk/lib/charisol-design-system-stack.ts`

Inputs:

The entity definitions in section 5.4.

The artifact layout in section 5.5.

The idempotency rules in section 6.4.

Implementation steps:

Create DynamoDB models or entity helpers for PrototypeJob, FigmaInventory, PrototypeArtifact, GitHubConnection, Deployment, PreviewToken, and AuditEvent.

Create a common job record factory that sets IDs, timestamps, state, phase, idempotency key, input hash, and initial status.

Create conditional state transition helpers. Include expected-state checks and optimistic concurrency protection.

Create artifact metadata helpers for S3 upload, content hash calculation, immutable object naming, and signed or authorized read references.

Create a cancellation mechanism that workers check between meaningful units of work.

Create a common warning and error structure for partial inventory, render, generation, and validation results.

Create cleanup behavior for temporary files and incomplete uploads. Do not delete completed artifacts during normal cleanup.

Validation:

Add unit tests for duplicate job creation, duplicate requests, concurrent state transitions, cancellation, retry, missing artifacts, and partial failure.

Add integration tests for S3 upload and metadata reads.

Verify DynamoDB queries can list jobs and artifacts by project.

Verify no large payload is stored directly in DynamoDB.

Completion criteria:

A later worker can create, update, poll, retry, cancel, and retrieve a job without knowing the internal storage implementation.

### Task 0.5: Add Step Functions and Fargate execution skeleton

Goal:

Move long-running work out of the API Lambda and provide a reliable orchestration skeleton.

Files and areas to inspect:

`charisol-design-system-be/cdk/lib/charisol-design-system-stack.ts`

`charisol-design-system-be/cdk/bin/cdk-app.ts`

`charisol-design-system-be/index.js`

`charisol-design-system-be/package.json`

Inputs:

The target flow in section 5.1.

The existing Lambda, API Gateway, DynamoDB, and S3 resources.

Implementation steps:

Add a Step Functions Standard Workflow with separate states for inventory, render, generate, validate, GitHub, and deploy work.

Add Fargate or ECS task definitions for each worker type. Use separate task roles where permissions differ.

Add environment variables for bucket names, table names, workflow names, provider configuration names, and feature flags. Do not put provider secrets in environment variables.

Grant each worker only the DynamoDB, S3, Secrets Manager, KMS, and provider-network permissions it needs.

Add retry and catch blocks for transient and permanent failures.

Add a dead-letter queue or equivalent failure destination for messages that cannot be processed.

Add CloudWatch log groups, retention, metrics, and alarms for worker failures, timeouts, and throttling.

Update the API Lambda only to start executions and return job IDs.

Validation:

Run the CDK build and `cdk synth`.

Inspect the synthesized IAM policies for least privilege.

Run a smoke workflow with a no-op or fixture worker.

Verify a queued job transitions to a worker phase and returns to a durable state.

Verify API Lambda duration remains independent of worker duration.

Completion criteria:

A long-running job can run beyond the API Lambda timeout.

A failed worker leaves a recoverable job and diagnostic artifact.

### Phase 0 exit criteria

All new API contracts are versioned, documented, and contract-tested.

Figma credentials are encrypted and the legacy import flow remains compatible during migration.

Every Prototype action has role enforcement and an audit event.

Jobs, artifacts, idempotency, cancellation, retries, and S3 metadata work independently of provider-specific code.

Step Functions and worker infrastructure synthesize and pass a smoke execution.

No secret appears in logs, job input, DynamoDB metadata, S3 metadata, or test fixtures.

## 8. Phase 1: Figma inventory, selection, and import

### Task 1.1: Implement the complete Figma inventory worker

Goal:

Fetch a complete, traceable inventory for the selected Figma file, page, or node without silently dropping content.

Files and areas to inspect:

`charisol-design-system-be/utils/index.js`

`charisol-design-system-be/controllers/figmaController.js`

`charisol-design-system-be/utils/figmaUrlParser.js`

`charisol-design-system-be/utils/figmaExtraction.js`

The new worker entry point created in Phase 0.

Inputs:

A validated Figma URL.

A resolved encrypted Figma credential.

A source scope containing file key, optional page IDs, and optional node IDs.

Implementation steps:

Parse the Figma URL and normalize file key, page ID, and node ID.

Resolve the credential through the encrypted credential helper. Never pass the credential to Step Functions input or logs.

Fetch file metadata and the complete selected page or node document. Use the Figma nodes endpoint for selected pages and nodes when the full file document is too large.

Build a parent map and node index.

Traverse every selected page and node, including nested children, instances, component sets, hidden nodes, text nodes, layout nodes, and prototype connections.

Capture stable source metadata for each node. Include node ID, parent ID, name, type, visibility, bounds, layout properties, fills, strokes, effects, text content, styles, component references, component properties, and prototype interactions when available.

Capture file-level variables and styles separately from node-level references.

Record API failures per page or node. Continue with other selected content when one branch fails.

Record truncation explicitly when Figma returns truncated children, a depth limit is reached, a response exceeds a worker limit, or a page is skipped.

Write the raw normalized inventory and metadata to S3.

Update the FigmaInventory and PrototypeJob records with counts, warnings, source version, artifact pointers, and completion state.

Validation:

Add fixture tests for nested nodes, multiple pages, instances, component sets, hidden nodes, text-heavy nodes, missing permissions, rate limits, partial failures, and truncated responses.

Verify inventory counts match fixture source counts.

Verify every retained node has a parent relationship or an explicit root marker.

Verify a failed child does not prevent unrelated pages from being inventoried.

Verify truncation is visible in job metadata and the UI.

Completion criteria:

A developer can trace every generated selection back to a Figma file version, page, and node ID.

The worker never silently presents incomplete content as complete.

### Task 1.2: Define and validate the normalized inventory schema

Goal:

Create a stable, versioned schema that isolates generation from Figma API shape changes.

Files and areas to inspect:

The new inventory worker from task 1.1.

The new artifact manifest helpers from task 0.4.

`charisol-design-system-be/docs/openapi.js` if inventory metadata is exposed through an API.

Inputs:

The raw Figma fields captured in task 1.1.

The generated prototype requirements in Phase 2.

Implementation steps:

Define a schema version and a top-level inventory object.

Include source metadata, file metadata, page list, node list, parent-child relationships, variables, styles, component metadata, prototype links, renders, warnings, and truncation metadata.

Use stable IDs for generated routes and components. Preserve Figma IDs separately.

Define explicit types for bounds, layout, paint, text, style reference, component reference, interaction, and warning objects.

Define a validation function that rejects missing required fields and reports recoverable warnings for optional fields.

Add a migration or compatibility layer for future schema versions.

Document which fields are required for generation and which are informational.

Validation:

Run schema validation against at least one small, one large, one multi-page, and one partially failed inventory fixture.

Verify unknown optional fields do not break generation.

Verify missing required fields produce a clear worker error.

Verify the schema version is present in the inventory artifact and API metadata.

Completion criteria:

The generator can consume the normalized inventory without importing Figma-specific traversal code.

The inventory can be replayed later from S3 without refetching Figma when the source version is unchanged.

### Task 1.3: Implement selected-node rendering

Goal:

Render each selected top-level frame or component to an immutable PNG that can be used for review and visual validation.

Files and areas to inspect:

`charisol-design-system-be/utils/index.js`, especially `renderNodesToImages`.

The new inventory worker and artifact helpers.

The CDK worker permissions from task 0.5.

Inputs:

A completed normalized inventory.

A list of selected node IDs.

Render options such as format, scale, and use-absolute-bounds.

Implementation steps:

Select only user-selected top-level frames or components by default. Allow explicit child selection only when the UI marks the child as a separate output.

Reuse the existing Figma image API batching and token queue behavior.

Request renders using the exact Figma file version when available.

Store each PNG under an immutable S3 key that includes inventory version, node ID, scale, and content hash.

Store render metadata containing source node ID, dimensions, scale, format, S3 key, hash, creation time, and render status.

Treat missing images, 429 responses, timeouts, and invalid node IDs as explicit render warnings or failures.

Honor `Retry-After` and bounded backoff.

Update the job with render counts, failed counts, artifact pointers, and completion state.

Validation:

Verify every selected node has zero or one successful render artifact, never duplicate successful artifacts for the same input.

Verify failed renders are listed in the job and UI.

Verify cached renders are validated by hash before reuse.

Verify 429 handling respects the provider retry signal.

Completion criteria:

The review UI can show a thumbnail and status for every selected node.

The generation and validation workers can retrieve renders by stable artifact metadata.

### Task 1.4: Add the Prototype selection and review UI

Goal:

Give users a clear project-dashboard workflow for connecting Figma, reviewing inventory, selecting nodes, and continuing to generation.

Files and areas to inspect:

`charisol-design-system-fe/components/sections/project/ProjectDetailsScreen.tsx`

`charisol-design-system-fe/components/sections/project/ProjectSidebar.tsx`

`charisol-design-system-fe/components/sections/project/ProjectSettingsSection.tsx`

`charisol-design-system-fe/services/project.service.ts`

The existing Figma modal and processing modal components.

Inputs:

The job and inventory API contracts from Phase 0.

The normalized inventory schema from task 1.2.

The render metadata from task 1.3.

Implementation steps:

Add `prototype` to the project section union and navigation items.

Create a dedicated Prototype section component rather than placing all workflow logic in `ProjectDetailsScreen`.

Create subcomponents for source connection, job status, inventory tree, selection summary, render review, and continue action.

Use the existing project ID from the route and existing authentication context.

Call the BFF endpoints through a dedicated service or clearly scoped project service methods.

Poll jobs with an abortable timer. Stop polling when the job reaches a terminal state or the component unmounts.

Persist the current job ID and selected node IDs in local storage so a user can close the UI and reattach to a background job.

Show explicit states for idle, connecting, inventory running, inventory complete, render running, render complete, partial failure, failed, cancelled, and ready for review.

Show page and node counts, selected counts, warnings, truncation, and failed renders.

Make selection changes update a local draft before starting generation.

Require an explicit Continue or Generate action after review.

Keep the existing Figma token-import modal available and unchanged.

Validation:

Add component tests for empty inventory, large tree, hidden nodes, partial failure, selection changes, background reattachment, polling cleanup, and keyboard navigation.

Run frontend lint, typecheck, tests, and build.

Manually verify the new tab does not disturb existing project tabs or branch query parameters.

Completion criteria:

A user can complete inventory and selection without using the old Figma modal.

A user can close the tab while a job runs and reattach to the same job later.

The UI never implies that an incomplete inventory is complete.

### Task 1.5: Store imported source metadata without changing design-system organization

Goal:

Persist the selected Figma source and node mapping for generation without altering existing tokens or components.

Files and areas to inspect:

`charisol-design-system-be/models/Project.js`

The new PrototypeConfig and FigmaInventory entities.

`charisol-design-system-be/controllers/figmaController.js`

Inputs:

The completed inventory and selected node list.

The existing project design-system data.

Implementation steps:

Write selected source metadata to PrototypeConfig and reference the inventory artifact.

Do not write the full inventory into the existing Project document.

Do not call token classification, reclassification, naming, grouping, or design-system organization code.

Preserve existing `figmaSync`, design-system variables, components, imports, manual edit markers, and project versions.

Record the source file key, source version, selected node IDs, inventory version, and selection timestamp.

Expose read-only source metadata to the Prototype UI.

Validation:

Compare project design-system data before and after the metadata write.

Verify token keys, values, classifications, component paths, and manual-edit markers are unchanged.

Verify the PrototypeConfig can be loaded by a generation job.

Completion criteria:

The new workflow is additive and does not mutate the existing design system.

A generation job can reproduce the exact selected source later.

### Phase 1 exit criteria

A real Figma file can be inventoried without the old depth-1 or first-page limitations.

Selected nodes render to traceable PNG artifacts.

The user can review the inventory and explicitly accept the selection.

The existing design-system organization and token behavior are unchanged.

Inventory and render failures are visible and recoverable.

## 9. Phase 2: Deterministic prototype generation

### Task 2.1: Build the normalized model-to-prototype compiler

Goal:

Convert the normalized inventory into a deterministic intermediate model that the template can render.

Files and areas to inspect:

The normalized inventory schema from task 1.2.

The new worker entry points from Phase 0.

The future generated template structure described in task 2.2.

Inputs:

A completed inventory artifact.

A selected node list.

Render artifact metadata.

Implementation steps:

Create a compiler module that reads the normalized inventory and selected node IDs.

Create one page model for each selected top-level node.

Create stable route IDs from node IDs and names. Normalize names and resolve collisions deterministically.

Map Figma layout and style references to intermediate design values without inventing production business behavior.

Map text nodes to editable text content and copy placeholders.

Map image and fill references to render artifact pointers.

Map component and instance metadata to intermediate component descriptors.

Map explicit Figma prototype interactions to navigation actions.

Create fallback navigation only when an explicit interaction is absent or unsupported.

Emit warnings for unsupported nodes, unsupported interactions, missing renders, ambiguous names, and oversized content.

Sort all output collections before serialization.

Write an intermediate model and compiler report to S3.

Validation:

Create golden-model fixtures for a simple frame, nested components, multiple pages, explicit prototype links, missing links, and unsupported nodes.

Run the compiler twice with identical input and verify byte-identical intermediate output.

Verify unsupported constructs appear in warnings and do not silently disappear.

Verify route IDs are unique and stable.

Completion criteria:

The compiler output is independent of Figma API response ordering.

The generator can build a prototype without calling Figma again.

### Task 2.2: Create the pinned React and Vite template

Goal:

Provide a clean, private, reproducible repository template for generated prototypes.

Files and areas to inspect:

The repository organization conventions in the frontend application.

The generated-code conventions in section 6.5.

Inputs:

The intermediate model from task 2.1.

The product decision to use React 19, TypeScript, and Vite.

Implementation steps:

Create or designate a private Strata-owned template repository.

Pin the template to an immutable commit and record that commit in every generated manifest.

Create separate directories for design tokens, design styles, components, routes, state, mock data, assets, and runtime configuration.

Add a deterministic entry point that reads the generated manifest and registers routes.

Add responsive layout primitives for desktop, tablet, and mobile.

Add a self-contained design-layer adapter interface.

Add a runtime manifest schema containing project ID, generation ID, template commit, route list, asset list, and feature flags.

Add install, build, lint, typecheck, and test scripts.

Add fixture tests that render a minimal generated model.

Validation:

Clone or copy the template into a clean directory.

Run its install, build, lint, typecheck, and test commands.

Verify the template contains no hardcoded customer secrets or production API URLs.

Verify the template commit is recorded in a generated manifest.

Completion criteria:

The generator can produce a clean repository from the pinned template.

A future design-system adapter can replace the generated design layer without changing the route runtime.

### Task 2.3: Generate responsive pages and routes

Goal:

Emit one navigable route per selected top-level frame or component with deterministic fallback navigation.

Files and areas to inspect:

The compiler from task 2.1.

The template from task 2.2.

The generator worker entry point from Phase 0.

Inputs:

The intermediate model.

The selected render artifacts.

The pinned template commit.

Implementation steps:

Create a route entry for every selected top-level node.

Generate page components that render the intermediate node tree through the template's design-layer primitives.

Generate a route manifest with route ID, path, title, source node ID, parent route, and fallback targets.

Preserve explicit Figma navigation targets by source node ID.

Resolve fallback links to deterministic route paths.

Generate responsive variants or responsive layout rules for desktop, tablet, and mobile.

Generate asset references using immutable S3 or bundle-relative paths.

Generate a page index and a not-found route.

Validation:

Run route-manifest tests for one page, multiple pages, nested nodes, duplicate names, explicit links, missing links, and unsupported links.

Build the generated repository and verify every route can be loaded.

Take screenshots at desktop, tablet, and mobile viewport sizes.

Verify no route points to an external production system.

Completion criteria:

Every selected top-level node has exactly one generated route.

Explicit Figma links are preserved where possible.

Fallback navigation is deterministic and visible in the manifest.

### Task 2.4: Generate interactive prototype behavior

Goal:

Add navigation, component states, forms, and mock data while keeping the output clearly non-production.

Files and areas to inspect:

The intermediate model and generated route components.

The template state and mock-data directories.

Inputs:

The route model, component descriptors, text nodes, and interaction model.

Implementation steps:

Generate a small state store or component-local state model for interactive elements.

Generate click, hover, focus, open, closed, selected, submitted, and error states only when the source model or template supports them.

Generate forms with mock fields and local submission behavior. Do not call production APIs.

Generate mock data files with stable deterministic values.

Generate navigation handlers from the route manifest.

Add visible prototype-only indicators or metadata where appropriate so developers do not mistake the output for production code.

Exclude authentication, payments, production analytics, production database calls, and external business integrations.

Validation:

Add interaction tests for navigation, state transitions, form submit, reset, and validation states.

Search generated output for prohibited production integration strings and external API calls.

Run the generated repository build and interaction smoke tests.

Completion criteria:

The prototype is interactive enough for review.

The generated code cannot accidentally call production services.

### Task 2.5: Add ownership metadata and deterministic diffs

Goal:

Protect manual edits when a generated repository is regenerated or updated through GitHub.

Files and areas to inspect:

The generated repository output from tasks 2.2 through 2.4.

The GitHub worker planned in Phase 3.

Inputs:

The generated file list and source-node mapping.

The previous artifact, if one exists.

Implementation steps:

Write an `ownership.json` file listing every generated-owned file, source node IDs, generation ID, and ownership mode.

Mark template-owned runtime files separately from generated content files.

Create a deterministic file hash for every output file.

Create a diff operation that compares the previous generated artifact with the new artifact.

Classify changes as generated-only, manual-only, generated-and-manual, added, removed, or unchanged.

When a file is generated-owned and manually modified, preserve the manual version and emit a conflict warning unless the user explicitly chooses to overwrite it.

Never force-push over a branch containing unreviewed manual edits.

Validation:

Modify a generated file and a manual file in a fixture repository.

Regenerate and verify the manual file is preserved.

Verify the generated-file diff is reported accurately.

Verify repeated generation with identical input produces the same content hash, excluding documented manifest metadata.

Completion criteria:

A later GitHub worker can update only files Strata owns.

Manual edits are visible and protected.

### Task 2.6: Run generation validation

Goal:

Produce a machine-readable validation report before any GitHub or deployment side effect.

Files and areas to inspect:

The generated repository and template scripts.

The validation worker entry point from Phase 0.

Inputs:

The generated bundle or checked-out generated repository.

The route manifest and ownership metadata.

Implementation steps:

Install dependencies in an isolated worker directory.

Run the generated repository build, TypeScript, and lint commands.

Validate the route manifest, asset references, ownership manifest, and source-node mappings.

Generate desktop, tablet, and mobile screenshots.

Run interaction smoke tests against the generated app.

Write a validation report containing commands, exit codes, durations, warnings, screenshot pointers, and artifact hashes.

Fail the job on build, type, lint, route, or security-policy violations.

Allow non-blocking warnings only when the report marks them explicitly.

Validation:

Run the validation worker against a known-good generated fixture.

Run it against fixtures with a missing route, broken asset, TypeScript error, lint error, and prohibited production integration.

Verify the job becomes failed with a useful report for each case.

Completion criteria:

No artifact proceeds to GitHub or Vercel without a validation report.

A developer can diagnose a failed generation from the report without accessing provider logs.

### Phase 2 exit criteria

The same normalized model and template commit generate the same prototype bundle.

Generated output is responsive, navigable, interactive, and self-contained.

Validation reports exist before external side effects.

Ownership metadata protects manual edits.

## 10. Phase 3: GitHub handoff

### Task 3.1: Implement GitHub App authentication

Goal:

Authenticate Strata to GitHub using a GitHub App and short-lived installation tokens without storing user PATs.

Files and areas to inspect:

The backend provider-client area created in Phase 0.

`charisol-design-system-be/cdk/lib/charisol-design-system-stack.ts`

The backend secrets and logging conventions.

Inputs:

GitHub App ID, private key, webhook secret, and installation metadata stored in Secrets Manager.

Implementation steps:

Create a GitHub App client that can create a JWT, exchange an installation ID for a short-lived installation token, and refresh the token when needed.

Store only the App ID, encrypted private key, webhook secret, and installation metadata in Secrets Manager.

Create a backend callback or connection endpoint that validates the GitHub OAuth or App installation flow and associates the installation with the project owner.

List accessible repositories through the installation token.

Never accept or persist a user PAT as the normal connection method.

Redact all GitHub responses and tokens from logs.

Validation:

Mock token creation, expiry, refresh, and failure.

Verify no PAT field exists in the connection model or API response.

Verify expired tokens are refreshed before a repository operation.

Verify secrets are absent from logs and job records.

Completion criteria:

The backend can call GitHub as the Strata App using short-lived tokens.

### Task 3.2: Support new repository creation

Goal:

Create a private repository from the pinned template and push the first generated branch.

Files and areas to inspect:

The GitHub App client from task 3.1.

The generated artifact from Phase 2.

The GitHubConnection and Deployment entities.

Inputs:

Owner authorization.

Repository name and visibility preference.

A validated generated artifact.

Implementation steps:

Validate repository name, owner permissions, visibility, and name availability.

Create the repository through the GitHub App installation.

Set the default branch and repository metadata.

Clone or materialize the pinned template in the worker.

Apply the generated bundle to the working tree without overwriting template files that are not generated-owned.

Create a dedicated branch using a stable naming convention such as `strata/prototype/<generationId>`.

Commit the generated files with a deterministic commit message and generation ID.

Push the branch and store the commit SHA.

Validation:

Mock GitHub repository creation, name collision, permission failure, push failure, and retry.

Verify new repositories default to private.

Verify the branch contains the generated artifact and template commit metadata.

Verify a repeated request reuses the existing repository or returns a clear conflict instead of creating duplicates.

Completion criteria:

A new repository is ready for a pull request and its metadata is stored in DynamoDB.

### Task 3.3: Support existing repository connection

Goal:

Connect an existing repository, verify permissions, create a generation branch, and open or update a pull request.

Files and areas to inspect:

The GitHub App client.

The repository selection UI planned in task 3.5.

The GitHubConnection entity.

Inputs:

Owner authorization.

Repository owner and name.

Default branch and protection rules.

A validated generated artifact.

Implementation steps:

List repositories available to the installation and let the owner select one.

Verify contents write, pull-request write, and branch creation permissions.

Read the default branch and protection rules.

Create or reset a dedicated generation branch from the correct base commit.

Apply generated-owned files and preserve manual files.

Open a pull request when none exists for the generation branch.

Update the existing pull request body, branch, and commit when a generation is repeated.

Record PR number, base branch, head branch, commit SHA, status, and URL.

Validation:

Mock missing permissions, protected branches, existing PRs, stale branches, disconnected installations, and idempotent retries.

Verify the worker never pushes to the default branch directly.

Verify an existing PR is updated rather than duplicated when possible.

Completion criteria:

An existing repository can receive a generated branch and PR without losing manual edits.

### Task 3.4: Preserve manual edits

Goal:

Make regeneration safe for repositories that developers have modified after the first generation.

Files and areas to inspect:

The ownership metadata from task 2.5.

The GitHub worker from tasks 3.2 and 3.3.

Inputs:

Previous generated artifact, new generated artifact, and current repository tree.

Implementation steps:

Read `ownership.json` from the previous artifact and current branch.

Classify each file as generated-owned, template-owned, manual, or unknown.

Apply only generated-owned changes by default.

Preserve manual files and emit a conflict report when generated and manual changes overlap.

Create a patch artifact containing added, modified, removed, preserved, and conflicted files.

Require owner or editor review for conflicts before merging or force-updating a PR.

Never use a force push to resolve a conflict automatically.

Validation:

Create fixtures with generated-only changes, manual-only changes, overlapping changes, deleted generated files, and new manual files.

Verify manual files remain unchanged.

Verify conflicts are reported with file paths and suggested resolutions.

Completion criteria:

Verify conflicts are reported with file paths and suggested resolutions.

Verify the PR is never force-pushed.

Verify the patch artifact is stored in S3 and linked to the job.

Completion criteria:

A developer can safely regenerate a prototype in a repository they have edited.

### Task 3.5: Add repository status UI

Goal:

Let users understand and manage the GitHub handoff from the Prototype tab or Project Settings.

Files and areas to inspect:

`charisol-design-system-fe/components/sections/project/ProjectDetailsScreen.tsx`

`charisol-design-system-fe/components/sections/project/ProjectSettingsSection.tsx`

The frontend service layer.

Inputs:

GitHubConnection and job API responses.

Implementation steps:

Create a repository status component with unconnected, connecting, connected, branch created, PR open, checks pending, PR merged, failed, and disconnected states.

Show repository URL, branch, PR URL, base branch, last generation, validation status, and required action.

Show owner-only controls for connecting, changing repository, changing visibility, and opening the PR.

Show editor-visible generation status without exposing owner-only controls.

Handle external navigation safely and copy URLs through explicit user actions.

Validation:

Add component tests for every state.

Verify owner and non-owner controls are rendered correctly.

Verify stale or failed connections have retry and reconnect actions.

Run frontend lint, typecheck, tests, and build.

Completion criteria:

A user can understand the GitHub handoff without reading worker logs.

### Phase 3 exit criteria

A project can create a new private repository or connect an existing repository through the GitHub App.

Every generation lands on a dedicated branch and produces or updates a PR.

Manual edits are preserved and conflicts are visible.

Repository and PR state is recoverable after failure or interruption.

## 11. Phase 4: Vercel deployment and preview links

### Task 4.1: Implement the Vercel provider adapter

Goal:

Create a narrow, testable deployment provider for Strata-managed Vercel projects.

Files and areas to inspect:

The provider interface from task 0.1.

The worker infrastructure from task 0.5.

The CDK secrets configuration.

Inputs:

Vercel team ID, project ID, token, and environment configuration stored in Secrets Manager.

Implementation steps:

Define provider methods for create project, deploy bundle or commit, read deployment status, promote or assign a production alias, rollback, and read deployment metadata.

Keep provider-specific URLs and tokens out of job input and logs.

Map Vercel errors to normalized provider errors with retryable and user-correctable flags.

Store only the provider deployment ID, branch, commit, status, and internal URL in DynamoDB.

Validation:

Mock project creation, deployment, status polling, rate limits, authorization failure, rollback, and idempotent retries.

Verify provider credentials are read from Secrets Manager.

Verify no Vercel token or signed URL appears in logs or job records.

Completion criteria:

The deployment worker can deploy a generated artifact without embedding Vercel-specific logic in the core workflow.

### Task 4.2: Deploy review branches immediately

Goal:

Make every validated generation available for review before PR approval.

Files and areas to inspect:

The Vercel provider adapter from task 4.1.

The Deployment entity.

The generated artifact and GitHub PR metadata.

Inputs:

A validated artifact.

A repository branch and commit.

Implementation steps:

Trigger a Vercel deployment after generation validation succeeds.

Associate the deployment with the generation ID, artifact hash, repository, branch, and commit.

Poll deployment status until it is ready or fails.

Store the internal Vercel URL only in backend deployment metadata.

Expose a Strata review URL through the gateway, not the raw Vercel URL.

Record deployment health, timestamps, and failure details.

Validation:

Mock successful deployment, pending deployment, failed deployment, and retry.

Verify a review deployment exists before the PR is shown as ready for review.

Verify the internal provider URL is not returned to browser code.

Completion criteria:

A reviewer can open a Strata-controlled review URL for each validated generation.

### Task 4.3: Implement stable URL promotion

Goal:

Control when a review deployment becomes the stable prototype URL.

Files and areas to inspect:

The Deployment entity.

The Vercel provider adapter.

The preview gateway route.

Inputs:

Owner action or verified PR merge event.

A healthy review deployment.

Implementation steps:

Define promotion policy. Support explicit owner promotion and promotion after a verified PR merge.

Check deployment health before promotion.

Update the Vercel alias or stable routing target only after the health check passes.

Write a new Deployment record or immutable promotion event with previous deployment as rollback target.

Emit an audit event and update PrototypeConfig stable preview state.

Handle concurrent promotion attempts with a conditional write.

Validation:

Mock failed health checks, successful promotion, PR merge promotion, concurrent promotion, and rollback.

Verify the stable URL does not change while a deployment is pending or failed.

Verify the previous deployment remains available as a rollback target.

Completion criteria:

Only an approved or explicitly published deployment becomes stable.

### Task 4.4: Implement the preview-token gateway

Goal:

Provide unlisted, revocable, non-expiring preview links without exposing the raw token or internal deployment URL.

Files and areas to inspect:

The PreviewToken entity.

The public Next.js route area.

The backend artifact and deployment APIs.

Inputs:

A validated artifact or promoted deployment.

Owner authorization for token creation.

Implementation steps:

Generate a high-entropy random token using a cryptographically secure generator.

Store only a SHA-256 or stronger hash of the token in DynamoDB.

Associate the token with one project and one artifact or deployment.

Create a public Next.js route at `/project/:projectId/preview/:token`.

Have the route call an authenticated or signed internal backend verification endpoint. The public route must not contain provider credentials.

Resolve the token hash server-side, check active state and project match, and serve or proxy the authorized artifact.

Do not redirect to a raw Vercel URL unless the gateway can hide the internal URL and still enforce token checks. Prefer a gateway proxy or signed internal fetch.

Add rate limiting, revocation, audit access, and safe not-found behavior.

Show the token to the owner once at creation time and never return it again.

Validation:

Add security tests for malformed tokens, revoked tokens, cross-project tokens, repeated access, rate limiting, and audit events.

Verify only the hash is stored.

Verify a raw token cannot be recovered from DynamoDB, logs, or S3 metadata.

Verify the public route returns a safe 404 or revoked response without revealing whether a token existed.

Completion criteria:

Preview links are unlisted, controlled, revocable, and audited.

### Task 4.5: Add deployment and preview UI

Goal:

Give users clear controls for review deployments, stable promotion, preview links, and rollback.

Files and areas to inspect:

The Prototype tab components.

The Project Settings component.

The frontend service layer.

Inputs:

Deployment and PreviewToken API responses.

Implementation steps:

Create deployment status cards for pending, ready, failed, review, promoted, and rolled back states.

Show review URL, stable URL state, deployment health, branch, commit, and promotion action.

Show owner-only promote, rollback, create preview link, and revoke preview link controls.

Show viewer-safe preview status without owner controls.

Display preview links as copyable values with a clear warning that the token is shown once.

Show revocation confirmation and audit result.

Validation:

Add component tests for every deployment and preview state.

Verify role-based controls.

Verify revoked links no longer open.

Verify copy actions do not log or persist the raw token.

Run frontend lint, typecheck, tests, and build.

Completion criteria:

Users can review, promote, share, revoke, and roll back deployments from the UI.

### Task 4.6: Define the future Plexo boundary

Goal:

Keep Vercel replaceable without building Plexo behavior in Phase One.

Files and areas to inspect:

The provider interface from task 0.1.

The Vercel adapter from task 4.1.

Implementation steps:

Define a provider factory that selects Vercel in Phase One.

Keep provider-specific configuration in backend secrets and project configuration.

Add compile-time or unit tests that swap the Vercel adapter for a fake provider without changing job, artifact, or UI contracts.

Document the additional secrets, endpoints, and deployment semantics a Plexo adapter would require.

Validation:

Run a fake-provider deployment test.

Verify no Vercel-specific fields leak into the core Deployment entity beyond a generic provider metadata object.

Verify the UI uses generic deployment states.

Completion criteria:

A future provider can be added without rewriting the core workflow.

### Phase 4 exit criteria

Review deployments are available immediately after generation.

The stable Strata URL changes only after approval, merge, or explicit publish.

Preview links are hashed, revocable, unlisted, and audited.

Provider credentials and internal URLs are hidden from browser code and logs.

The deployment provider can be replaced behind the existing interface.

## 12. Phase 5: UI integration, operations, and rollout

### Task 5.1: Integrate the Prototype tab into the project page

Goal:

Make the complete workflow available from the existing project dashboard without disrupting current project features.

Files and areas to inspect:

`charisol-design-system-fe/app/project/[id]/page.tsx`

`charisol-design-system-fe/components/sections/project/ProjectDetailsScreen.tsx`

`charisol-design-system-fe/components/sections/project/ProjectSidebar.tsx`

Inputs:

The Prototype section and all API services from earlier phases.

Implementation steps:

Add the Prototype section to the project page render switch.

Add the Prototype navigation item and active state.

Pass project ID, role, branch state, and project refresh callbacks into the Prototype section.

Keep existing tabs, branch query parameters, publish modal, export modal, and brand-context modal behavior unchanged.

Add a feature flag that can hide or disable the new tab in an environment.

Validation:

Run existing project-page tests.

Manually exercise tokens, components, handoff, settings, collaboration, branch and publish, and Prototype tabs.

Verify branch query parameters still work.

Verify users without permission see the correct disabled or hidden state.

Completion criteria:

The Prototype tab is reachable and does not regress existing project behavior.

### Task 5.2: Add end-to-end workflow states

Goal:

Make every normal, error, retry, cancellation, and success state understandable and accessible.

Files and areas to inspect:

The Prototype section and subcomponents.

The existing Chakra and inline styling patterns.

Inputs:

The job state machine and API error envelope.

Implementation steps:

Implement idle, connecting, inventory running, inventory complete, render running, render complete, generation queued, generation running, validation running, GitHub pending, deployment pending, ready for review, promoted, failed, cancelled, and rolled back states.

Use clear headings, status text, retry actions, and progress indicators.

Use live regions for status changes and move focus to important actions when a phase completes or fails.

Make all dialogs and selection controls keyboard accessible.

Ensure polling stops on unmount and on terminal states.

Add a background-job reattachment flow using persisted job ID.

Validation:

Add component tests for every state and transition.

Run an E2E flow from source connection through preview creation.

Verify no infinite polling, infinite loading, typecheck, and build.

Completion criteria:

A user can recover from every expected failure without developer assistance.

### Task 5.3: Add observability and operational controls

Goal:

Give operators enough visibility to diagnose failures without exposing secrets.

Files and areas to inspect:

The backend worker and API logging.

The CDK monitoring resources.

The audit event model.

Inputs:

The shared logging conventions.

Implementation steps:

Emit metrics for job starts, phase durations, failures, retries, cancellation, artifact size, Figma rate limits, GitHub failures, Vercel failures, and preview access.

Add structured logs with project ID, job ID, phase, provider, status, and sanitized error code.

Add alarms for worker failures, timeouts, dead-letter messages, and repeated provider throttling.

Add an operator view or API response that shows job phase, last error, artifact pointers, retry count, and safe recovery action.

Add a kill switch that prevents new jobs from starting without deleting existing jobs or artifacts.

Validation:

Run synthetic success, failure, timeout, and rate-limit jobs.

Verify metrics and alarms are emitted.

Verify logs contain no secrets or raw provider payloads.

Verify the kill switch stops new jobs and leaves existing jobs readable.

Completion criteria:

Operators can diagnose and recover a failed workflow from Strata tooling.

### Task 5.4: Add migration and compatibility checks

Goal:

Protect existing projects and workflows while the new system rolls out.

Files and areas to inspect:

Existing project, Figma import, snapshot, branch, and connected-surface code.

The migration script from task 0.2.

Inputs:

Legacy project fixtures and existing API responses.

Implementation steps:

Backfill default PrototypeConfig records for existing projects without changing project data.

Preserve existing `figmaSync` metadata and old Figma endpoint behavior.

Keep old project branches and publish endpoints functional.

Add compatibility tests for legacy project shapes, legacy Figma tokens, existing snapshots, and existing connected surfaces.

Document any fields that are read but not migrated.

Validation:

Run migration tests against legacy fixtures.

Run existing backend and frontend regression suites.

Verify old Figma import, sync, snapshot, branch, publish, and surface endpoints still pass.

Completion criteria:

Existing users see no breaking change when the Prototype feature is disabled or not yet connected.

### Task 5.5: Roll out behind a feature flag and kill switch

Goal:

Goal:

Start with internal projects and controlled fixtures.

Implementation steps:

Add environment and project-level feature flags.

Start with internal projects and a known Figma fixture.

Validate small, large, multi-page, new-repository, and existing-repository flows in staging.

Expand access only after security, visual, provider, and recovery checks pass.

Add a kill switch that prevents new jobs from starting.

Keep completed jobs, artifacts, repositories, PRs, and deployments readable during rollback.

Validation:

Run a staging rollout checklist.

Disable the feature flag and verify new jobs cannot start.

Re-enable the flag and verify existing jobs remain readable.

Verify rollback does not delete user-owned resources.

Completion criteria:

The feature can be launched, paused, and rolled back safely.

### Task 5.6: Document operating procedures

Goal:

Give operators and future developers a repeatable way to run and recover the system.

Files and areas to inspect:

The CDK and deployment documentation.

The backend and frontend deployment guides.

Implementation steps:

Document AWS KMS and Secrets Manager setup for Figma, GitHub, Vercel, and webhook secrets.

Document GitHub App installation, repository permissions, webhook verification, and rotation.

Document Vercel team and project setup, domain or gateway configuration, and token rotation.

Document how to inspect a job, retry a phase, cancel a job, recover a worker, revoke a preview token, roll back a deployment, and delete a Prototype resource.

Document the expected S3 and DynamoDB artifacts for a failed job.

Document the feature flag and kill-switch procedure.

Validation:

Run each runbook step in a clean staging environment.

Verify each command or UI path is current and does not require undocumented access.

Have a developer who did not implement the feature follow one recovery runbook.

Completion criteria:

An operator can recover the workflow without reading source code or asking the original implementer.

### Phase 5 exit criteria

The complete workflow is usable from the existing project dashboard.

Existing functionality remains regression-free.

Operations can recover failed jobs, revoke previews, roll back deployments, and audit sensitive actions.

Staging validation uses real Figma, GitHub, and Vercel integrations with non-production credentials.

Runbooks are complete and tested.

## 13. Cross-cutting validation

### Security validation

Verify Figma credentials, GitHub App private keys, Vercel tokens, webhook secrets, and preview-token signing material are encrypted at rest.

Verify only preview-token hashes are stored.

Verify provider credentials are never sent to the browser, included in Step Functions input, written to S3 metadata, or logged.

Verify API and worker responses use allowlisted fields and redaction.

Verify project authorization is checked at the API boundary and again before external side effects.

Verify preview routes are rate-limited and audited.

Verify a cross-project job ID, artifact ID, repository ID, deployment ID, or preview token cannot access another project's resources.

### Reliability and idempotency validation

Verify every job has an idempotency key and durable state transition.

Verify duplicate API requests return the existing job instead of starting duplicate work.

Verify workers use checkpoints and can resume after interruption.

Verify Figma 429 responses honor `Retry-After`.

Verify GitHub and Vercel retries use bounded backoff and do not duplicate repositories, PRs, or deployments.

Verify failed jobs retain artifacts and reports for diagnosis.

Verify cancellation stops future side effects and records the terminal state.

Verify cancellation stops future side effects and records the terminal state.

Verify a worker failure.

Verify failure compensation.

Verify cancellation stops future side effects.

### Accessibility and UX validation

Verify the Prototype tab and dialogs are keyboard navigable.

Verify progress and status changes use appropriate live regions.

Verify selection trees support keyboard focus and clear selected state.

Verify error states explain the cause and next action without exposing internals.

Verify review and deployment URLs are copied through explicit user actions.

Verify color, icon, and status indicators do not rely on color alone.

### Testing matrix

Unit tests must cover inventory normalization, schema validation, token hashing, route-map generation, ownership diffs, provider adapters, and state transitions.

Integration tests must cover API authorization, Step Functions start, poll, cancel, S3 artifact reads and writes, GitHub mocked flows, and Vercel mocked flows.

Generated artifact tests must cover clean install, build, TypeScript, lint, route manifest, responsive screenshots, and interaction smoke tests.

End-to-end tests must cover Figma connect, inventory, selection, render, generation, validation, GitHub branch and PR, Vercel review deployment, preview token, promotion, and rollback.

Regression tests must cover existing Figma token import, project branches, publish and snapshot flow, connected surfaces, and project settings.

## 14. Rollout and rollback procedure

Deploy Phase 0 security and infrastructure changes first.

Run the plaintext Figma credential migration in dry-run mode, then in a controlled production migration with rollback metadata.

Enable the Prototype tab for internal users only.

Validate a small Figma file, a large Figma file, a multi-page file, an existing repository, and a new repository.

Expand access only after job, visual, security, and provider checks pass.

To roll back, disable new Prototype jobs, leave artifacts and repository history intact, stop stable URL promotion, and route the stable URL to the previous deployment.

Revoke affected preview tokens.

Do not automatically delete user repositories, branches, PRs, deployments, or manual edits during rollback.

## 15. Definition of done

Every phase exit criterion is satisfied.

Every new backend route is represented in OpenAPI and passes contract validation.

All new and modified frontend and backend tests pass.

Generated prototypes pass build, TypeScript, lint, route, responsive, and interaction checks.

Real Figma inventory includes all selected pages and nodes or reports explicit truncation.

GitHub App authentication uses short-lived installation tokens and no PAT.

Vercel review deployments and stable promotion are verified in staging.

Preview tokens are hashed, revocable, audited, and inaccessible across projects.

Existing project, Figma import, branch, snapshot, and connected-surface behavior is regression-tested.

Operational runbooks cover credentials, recovery, rollback, revocation, and deletion.

## 16. Operational prerequisites

Before implementation begins, provision or identify the following:

An AWS KMS key and Secrets Manager access for Figma, GitHub App, Vercel, and webhook secrets.

A GitHub App with repository creation, pull-request, branch, contents, and webhook permissions.

A private Strata-owned template repository and pinned template commit.

A Strata-managed Vercel team and project with an API token.

The Strata domain or gateway route that will serve stable and tokenized preview URLs.

Non-production Figma, GitHub, and Vercel fixtures for staging validation.

A staging environment where long-running workers, S3 artifacts, DynamoDB state, and CloudWatch alarms can be exercised.

## 17. Developer task handoff checklist

Before starting a task, identify its phase goal, inputs, output artifacts, dependencies, and owner role.

Before editing code, inspect the existing file and its neighboring conventions.

Before adding a backend route, update the OpenAPI source and add contract coverage.

Before adding a worker, define its input, output, retry behavior, failure behavior, and secret boundary.

Before adding UI, define every loading, success, error, empty, permission, and background state.

Before marking a task complete, run the task validation steps and record any unresolved warning in the job or report.

Before handing the task to another developer, include the files changed, the contract or schema version used, the validation commands run, and the exact state the next task should consume.
