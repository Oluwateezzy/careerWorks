# Strata Prototype Generation: Master Testing & Verification Guide (Phases 0 – 4)

This comprehensive guide walks you through testing the entire prototype generation pipeline **from Phase 0 through Phase 4**, step by step in chronological order.

---

## Table of Contents
1. [Prerequisites & Environment Setup](#1-prerequisites--environment-setup)
2. [Phase 0: Foundation — Enhanced Node Data & Bedrock Integration](#2-phase-0-foundation--enhanced-node-data--bedrock-integration)
3. [Phase 1: Screen Package Builder & Per-Screen LLM Generation](#3-phase-1-screen-package-builder--per-screen-llm-generation)
4. [Phase 2: Assembly Worker & Build Pipeline](#4-phase-2-assembly-worker--build-pipeline)
5. [Phase 3: Step Functions Orchestration & Parallel Generation](#5-phase-3-step-functions-orchestration--parallel-generation)
6. [Phase 4: Frontend Integration & Prototype Studio UX](#6-phase-4-frontend-integration--prototype-studio-ux)
7. [End-to-End Live Walkthrough (Figma to Browser)](#7-end-to-end-live-walkthrough-figma-to-browser)
8. [Summary Checklist & Quick Reference](#8-summary-checklist--quick-reference)

---

## 1. Prerequisites & Environment Setup

### 1.1 Requirements
- **Node.js**: `v20.x` or `v22.x`
- **Yarn**: `v1.22.x` (or npm)
- **AWS Credentials**: Configured with Bedrock (`anthropic.claude-3-5-sonnet-20241022-v2:0` access), DynamoDB, and S3 permissions.
- **Git Repository**: Branch `Prototype-generation`

### 1.2 Directory Structure
```
/Users/oluwatobiloba/Desktop/charisol/design_system/
├── charisol-figma-plugin/       # Phase 0.1 plugin walker & node serializer
├── charisol-design-system-be/   # Backend services, workers, Lambdas, & CDK
└── charisol-design-system-fe/   # Next.js frontend, BFF routes, & Studio UI
```

### 1.3 Launch Development Servers
Open two terminal tabs:
```bash
# Terminal 1: Backend API (Port 5001)
cd /Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-be
npm run dev

# Terminal 2: Frontend App (Port 3000)
cd /Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-fe
npm run dev
```

---

## 2. Phase 0: Foundation — Enhanced Node Data & Bedrock Integration

**Goal**: Verify rich Figma node serialization, snapshot normalization, Amazon Bedrock Converse integration, and prompting templates.

### Step 0.1: Figma Plugin Node Serializer
Ensure the plugin serializes typography, multi-type fills (gradients, images), strokes, shadows/blurs, corner radii, and auto-layout flex wrapping:
```bash
cd /Users/oluwatobiloba/Desktop/charisol/design_system/charisol-figma-plugin
npm run typecheck
npm run build
npm test
npm run sim
```
- **What to verify**:
  - `npm test` passes without TypeScript or lint errors.
  - `npm run sim` outputs serialized JSON containing `fontFamily`, `fontSize`, `fontWeight`, `letterSpacing`, `lineHeight`, `fills` (with gradients and image hashes), `strokes`, and `layoutWrap`.

### Step 0.2: Snapshot Loader Normalization
Verify that raw snapshots (either v1 or v2 format) are safely normalized:
```bash
cd /Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-be
node --test tests/unit/generate-snapshot-loader.test.js
```
- **What to verify**: `loadSnapshotData()` resolves `nodesById`, resolves design token collections, and maps asset hashes.

### Step 0.3: Bedrock Converse API & IAM Configuration
Verify the Bedrock client communicates using the Amazon Bedrock Converse API with Claude 3.5 Sonnet:
```bash
cd /Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-be
node --test tests/unit/bedrock-service.test.js
```
- **What to verify**:
  - `invokeBedrockConverse()` handles message formatting, system prompts, image attachments, and returns token metrics (`inputTokens`, `outputTokens`).
  - Gracefully maps `ThrottlingException` and `ModelTimeoutException`.

### Step 0.4: LLM Prompt Templates
Verify prompt rendering with design tokens, layout instructions, and strict delimiters:
```bash
cd /Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-be
node --test tests/unit/prompt-templates.test.js
```
- **What to verify**: System and user prompts enforce `---TSX---` and `---CSS---` separation and inject component library rules.

---

## 3. Phase 1: Screen Package Builder & Per-Screen LLM Generation

**Goal**: Verify that screens are isolated into self-contained packages under 5,000 tokens, generated individually via Bedrock, and validated for syntax and safety.

### Step 1.1: Screen Package Builder & Tree Pruner
Verify subtree pruning and token budgeting:
```bash
cd /Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-be
node --test tests/unit/screen-package.test.js
```
- **What to verify**:
  - `pruneNodeTree()` strips invisible nodes (`visible === false`), curves/SVG vector blobs, rounds coordinates, and collapses repeated sibling rows.
  - Large trees (>500 nodes) are pruned down to <200 nodes and remain comfortably under the 5,000 token limit.
  - Image assets in the screen subtree resolve to presigned S3 URLs.

### Step 1.2: Screen Generator Service
Verify that Bedrock responses are parsed into valid TSX and CSS modules:
```bash
cd /Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-be
node --test tests/unit/screen-generator.test.js
```
- **What to verify**:
  - Extracts clean `tsx` and `css` blocks from Bedrock delimiters.
  - Records input/output tokens and latency.
  - Gracefully returns failed status with fallback info if model throttles or errors out.

### Step 1.3: Code Validator
Verify that generated React code is checked for syntax, valid React imports, CSS module adherence, and security:
```bash
cd /Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-be
node --test tests/unit/code-validator.test.js
```
- **What to verify**: Rejects malformed JSX, dangerous evaluations (`eval`, `dangerouslySetInnerHTML`), and missing default exports.

### Step 1.4: Per-Screen Status Tracking API
Verify that the backend tracks per-screen statuses in DynamoDB and exposes the API endpoint:
```bash
cd /Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-be
node --test tests/unit/screen-progress.test.js
```
- **What to verify**:
  - `GET /projects/:projectId/prototype/jobs/:jobId/screens` returns `200` with total screen count, completed count, and array of screen records (`screenId`, `status`, `latencyMs`, `tokenUsage`, `error`).

---

## 4. Phase 2: Assembly Worker & Build Pipeline

**Goal**: Verify that individual screens are assembled into a runnable Vite + React project, auto-fixed if build errors occur, and packaged as a `.zip` file with a `README.md`.

### Step 2.1: Assembly Worker
Verify project scaffolding, routing, token CSS, Google Fonts, and placeholders:
```bash
cd /Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-be
node --test tests/unit/assembler.test.js
```
- **What to verify**:
  - Scaffolds from `templates/prototype/` to work directory.
  - Writes screens to `src/screens/{ComponentName}.tsx` and `.module.css`.
  - Detects shared component imports and writes stubs to `src/components/{Name}.tsx`.
  - Synthesizes `src/routes.tsx` mapping URL paths to screen components.
  - Links `src/design/tokens.css` in `src/main.tsx`.
  - Generates accessible fallback cards for any screens that failed generation.

### Step 2.2: Automated Build Error Recovery
Verify auto-fixing of common compiler errors:
```bash
cd /Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-be
node --test tests/unit/build-fixer.test.js
```
- **What to verify**: Detects missing package imports (e.g., `lucide-react`, `clsx`), syntax glitches, or undefined variables, and patches files automatically before aborting.

### Step 2.3: Readme Synthesis & Package Packaging
Verify that the assembled prototype includes setup instructions and the route list:
```bash
cd /Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-be
node --test tests/unit/assembler-readme.test.js
```
- **What to verify**: Produces `README.md` containing `# Prototype: [Project Name]`, `npm install`, `npm run dev`, and all screen routes (e.g., `/ — Hero Section`, `/about — About Page`).

### Step 2.4: Dual-Mode Pipeline Integration Test
Verify both LLM mode and deterministic compilation modes:
```bash
cd /Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-be
node --test tests/unit/generation-llm-pipeline.test.js
```
- **What to verify**: Both `mode: "llm"` and `mode: "deterministic"` compile clean, runnable output.

---

## 5. Phase 3: Step Functions Orchestration & Parallel Generation

**Goal**: Verify AWS Step Functions state machine orchestration with parallel Map state execution (`maxConcurrency: 5`), error isolation, cost tracking, and job cancellation.

### Step 3.1: CloudFormation & Step Functions Synthesis
Verify CDK state machine definition:
```bash
cd /Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-be/cdk
npm run build
npx cdk synth --no-staging
```
- **What to verify**:
  - CloudFormation template synthesizes clean `AWS::StepFunctions::StateMachine`.
  - `GenerateScreensMap` is configured with `"MaxConcurrency": 5`.
  - Retry policy includes backoff on `ThrottlingException`.
  - Catch policy routes failures to `ScreenFailed` pass state so a single screen error does not abort the whole execution.

### Step 3.2: Modular Lambda Handlers
Verify the three decoupled Lambda handlers:
```bash
cd /Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-be
node --test tests/unit/generation-lambdas.test.js
```
- **What to verify**:
  - `prepareScreenPackages.handler`: Generates packages, sets 15,000 token budget check, returns `{ screenPackages }`.
  - `generateScreen.handler`: Generates TSX/CSS, updates screen status, records token counts & cost.
  - `assemblePrototype.handler`: Assembles directory, builds Vite project, creates `source.zip` and manifest.

### Step 3.3: API Controller & Step Functions Service
Verify API controller interaction with Step Functions:
```bash
cd /Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-be
node --test tests/unit/prototype-step-functions.test.js
```
- **What to verify**:
  - `POST /prototype/generate` starts state machine execution and returns `202 Accepted` with `executionArn`.
  - `POST /prototype/jobs/:jobId/cancel` stops the Step Functions execution via `StopExecution`.
  - Enforces daily generation limits (11th request returns `429 DAILY_LIMIT_EXCEEDED`).

### Step 3.4: Cost & Token Tracker
Verify LLM pricing and budget checks:
```bash
cd /Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-be
node --test tests/unit/llm-cost-tracker.test.js
```
- **What to verify**:
  - Calculates Claude 3.5 Sonnet pricing ($3.00/M input, $15.00/M output).
  - Enforces 15,000 token screen budget.
  - Enforces 10 generations/day project limit.

### Step 3.5: Phase 3 Gate Integration Suite
Run the full Phase 3 integration test simulating 5+ concurrent screens, 1 failing screen recovering with fallback, cost tracking, and cancellation:
```bash
cd /Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-be
node --test tests/integration/phase3-step-functions-e2e.test.js
```
- **Expected Output**: `✔ Phase 3 Gate: Parallel Screen Generation, Step Functions Orchestration, and Cost Control` (all 4 gate criteria pass).

---

## 6. Phase 4: Frontend Integration & Prototype Studio UX

**Goal**: Verify granular per-screen generation progress, screen selection controls and guardrails, preview frame navigation, source zip download, and generation mode controls.

### Step 4.1: Generation Progress UI & Screen Status Polling
Verify the granular progress card:
```bash
cd /Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-fe
npm test __tests__/generation-progress.test.tsx
```
- **What to verify**:
  - Live duration timer formatting (e.g. `Duration 2m 8s`).
  - Active 3-second polling of `getJobScreens`.
  - Status display for individual screens:
    - Succeeded: green checkmark (`✅`) with latency (e.g., `(12.3s)`).
    - Generating: animated spinner (`🔄`) with `(generating...)`.
    - Pending: clock (`⏳`) with `(pending)`.
    - Failed: red alert (`❌`) with hover tooltip revealing error details.
  - Accurate progress bar percentage (0% to 100%).
  - Cancel and Retry button click handlers.

### Step 4.2: Screen Selection Grid & Guardrails
Verify selection actions and high-count warnings:
```bash
cd /Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-fe
npm test __tests__/screen-selection.test.tsx
```
- **What to verify**:
  - Screen thumbnails load from snapshot asset URLs.
  - "Select All" selects all screens; "Deselect All" clears selection.
  - When `0` screens are selected, "Generate Prototype" button is disabled.
  - When `> 20` screens are selected, warning appears:
    `High Screen Count (X screens selected): Generating more than 20 screens simultaneously may increase Step Functions execution time and incur higher LLM token usage.`

### Step 4.3: Generation Mode Toggle (LLM vs Deterministic)
Verify switching between AI synthesis and quick compiler mode:
```bash
cd /Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-fe
npm test __tests__/generation-modes.test.tsx
```
- **What to verify**:
  - Default mode is `"llm"`.
  - Options modal permits switching to `"deterministic"`.
  - Clicking "⚡ Quick Generate" in the footer directly initiates deterministic mode.

### Step 4.4: Preview Frame & Route Navigation
Verify route switching, fidelity chip, and report warnings:
```bash
cd /Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-fe
npm test __tests__/preview-download.test.tsx
```
- **What to verify**:
  - Route navigation pills at the top switch iframe URL on click.
  - Fidelity score badge renders with appropriate color coding (`>=90%` emerald, `75–89%` amber, `<75%` red).
  - Warnings drawer toggles open/closed to show generation report warnings.

### Step 4.5: Full Frontend Suite & TypeScript Typecheck
Run the complete frontend test suite and typechecker:
```bash
cd /Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-fe
npm test __tests__/generation-progress.test.tsx __tests__/screen-selection.test.tsx __tests__/generation-modes.test.tsx __tests__/preview-download.test.tsx __tests__/generation-panel.test.tsx __tests__/PrototypeSection.test.tsx
npm run typecheck
```
- **Expected Output**:
  - `Test Suites: 6 passed, 6 total` (38/38 passing).
  - `npm run typecheck`: clean with 0 errors.

---

## 7. End-to-End Live Walkthrough (Figma to Browser)

Follow these manual steps to test the real workflow from Figma to a running prototype:

### Step A: Push Snapshot from Figma
1. In Figma, open your design file.
2. Launch **Plugins** -> **Development** -> **Strata Design System Plugin**.
3. Authenticate with your project ID (or select your project from the dropdown).
4. Click **Export Snapshot to Strata**.
5. Confirm the upload succeeds (nodes, typography, fills, and thumbnails are stored).

### Step B: Open Prototype Studio in Browser
1. In your browser, navigate to:
   ```
   http://localhost:3000/projects/[yourProjectId]
   ```
2. Click on the **Prototype** tab.
3. Verify that the snapshot digest displays:
   - Screen cards show image thumbnails and dimensions (`1440 × 900 px`).
   - Sticky footer displays: `"X screens selected"`.

### Step C: Test Selection & Guardrails
1. Click **Deselect All** -> verify selected count drops to 0 and "Generate Prototype" button is disabled.
2. Click individual screen cards to select 2 to 4 screens -> verify selected count updates and buttons re-enable.
3. If testing with >20 screens, select all -> verify the amber warning banner appears:
   `⚠️ High count (X screens). May increase build time.`

### Step D: Trigger Generation & Observe Live Progress
1. Click **✨ Generate Prototype**.
2. In the modal, verify:
   - Mode is set to **AI-Powered LLM Synthesis (Default)**.
   - Click **Start Generation**.
3. Observe the **Generation** progress card:
   - Live elapsed duration timer ticks up (e.g. `Duration 12s`, `Duration 45s`).
   - "Preparing screen packages... ✅" completes.
   - Screen cards display their individual progress:
     - Transition from `(pending)` -> `(generating...)` -> `(12.4s) ✅`.
   - "Assembling prototype..." transitions to complete.
   - "Building..." transitions to complete.
   - "Uploading..." transitions to complete.
   - Overall progress bar reaches `100%`.

### Step E: Inspect Prototype in Preview Frame
1. Once generation finishes, the **Preview Frame** renders above the snapshot explorer.
2. Test the **Route Navigation Pills**:
   - Click `/` -> verify home screen displays.
   - Click `/about` (or other screen routes) -> verify iframe switches route.
3. Test **Viewport Switcher**:
   - Click **Desktop** (`1440px`), **Tablet** (`834px`), and **Mobile** (`390px`) -> verify iframe width adjusts.
4. Check **Fidelity Score & Warnings**:
   - Verify fidelity score chip (e.g. `Fidelity: 94%`).
   - If warnings exist, click the warning button to inspect the generation report drawer.

### Step F: Download and Run Prototype Locally
1. In Prototype Studio, click **Download source (.zip)**.
2. Save the `.zip` archive to your machine and extract it:
   ```bash
   mkdir /tmp/my-test-prototype
   unzip ~/Downloads/source.zip -d /tmp/my-test-prototype
   cd /tmp/my-test-prototype
   ```
3. Inspect `README.md`:
   ```bash
   cat README.md
   ```
   - Verify setup instructions and route list are clearly formatted:
     ```markdown
     # Prototype: [Project Name]
     Generated by Strata on [Date]

     ## Setup
     npm install
     npm run dev

     ## Routes
     / — Hero Section
     /about — About Page
     ...
     ```
4. Install dependencies and start the Vite dev server:
   ```bash
   npm install
   npm run dev
   ```
5. Open `http://localhost:5173` in your browser.
   - Verify the application runs cleanly without build errors.
   - Click routes or navigate between screens.

---

## 8. Summary Checklist & Quick Reference

Run this single consolidated sequence of commands to verify all phases in under 3 minutes:

```bash
# ==============================================================================
# PHASE 0 & 1: Foundation, Screen Packages, Prompts & Bedrock
# ==============================================================================
cd /Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-be
node --test tests/unit/generate-snapshot-loader.test.js
node --test tests/unit/bedrock-service.test.js
node --test tests/unit/prompt-templates.test.js
node --test tests/unit/screen-package.test.js
node --test tests/unit/screen-generator.test.js
node --test tests/unit/code-validator.test.js
node --test tests/unit/screen-progress.test.js

# ==============================================================================
# PHASE 2: Assembly Worker, Auto-Fixer & Readme
# ==============================================================================
node --test tests/unit/assembler.test.js
node --test tests/unit/build-fixer.test.js
node --test tests/unit/assembler-readme.test.js
node --test tests/unit/generation-llm-pipeline.test.js

# ==============================================================================
# PHASE 3: Step Functions, Lambda Handlers & Cost Tracker
# ==============================================================================
node --test tests/unit/llm-cost-tracker.test.js
node --test tests/unit/generation-lambdas.test.js
node --test tests/unit/prototype-step-functions.test.js
node --test tests/integration/phase3-step-functions-e2e.test.js

# CDK State Machine Synthesis
cd cdk && npm run build && npx cdk synth --no-staging && cd ..

# Backend Build & OpenAPI Sync Check
npm run build

# ==============================================================================
# PHASE 4: Frontend UI, Selection, Mode Toggle & Preview
# ==============================================================================
cd /Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-fe
npm test __tests__/generation-progress.test.tsx \
         __tests__/screen-selection.test.tsx \
         __tests__/generation-modes.test.tsx \
         __tests__/preview-download.test.tsx \
         __tests__/generation-panel.test.tsx \
         __tests__/PrototypeSection.test.tsx

# TypeScript Typecheck
npm run typecheck
```

✅ **All checks green**: The full pipeline from Figma ingestion to browser preview and runnable zip download is verified and production-ready!
