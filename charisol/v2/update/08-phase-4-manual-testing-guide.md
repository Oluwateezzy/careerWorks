# Phase 4 Manual Testing & Verification Guide

This guide provides end-to-end instructions for testing and validating **Phase 4: Frontend Integration**.

---

## Overview of Phase 4 Deliverables

| Task | Scope | Primary Files | Verification Command |
|---|---|---|---|
| **4.1** | Generation Progress UI & Screen Status | `charisol-design-system-fe/components/sections/project/prototype/GenerationProgress.tsx`<br>`charisol-design-system-fe/app/api/projects/[id]/prototype/jobs/[jobId]/screens/route.ts`<br>`charisol-design-system-fe/services/prototype.service.ts`<br>`charisol-design-system-fe/types/prototype.ts` | `npm test __tests__/generation-progress.test.tsx` |
| **4.2** | Screen Selection & Warnings | `charisol-design-system-fe/components/sections/project/prototype/explorer/ScreensTab.tsx`<br>`charisol-design-system-fe/components/sections/project/prototype/PrototypeActionFooter.tsx` | `npm test __tests__/screen-selection.test.tsx` |
| **4.3** | Preview, Routes, & Runnable Zip | `charisol-design-system-fe/components/sections/project/prototype/PreviewFrame.tsx`<br>`charisol-design-system-fe/components/sections/project/PrototypeSection.tsx`<br>`charisol-design-system-be/workers/generate/assembler.js` | `npm test __tests__/preview-download.test.tsx`<br>`node --test charisol-design-system-be/tests/unit/assembler-readme.test.js` |
| **4.4** | Generation Mode Toggle | `charisol-design-system-fe/components/sections/project/prototype/PrototypeActionFooter.tsx` | `npm test __tests__/generation-modes.test.tsx` |
| **Gate**| Phase 4 Gate End-to-End | Full frontend test suite & typecheck | `npm test`<br>`npm run typecheck` |

---

## Task 4.1: Generation Progress UI & Screen Status API

### Objective
Verify that the generation progress UI displays granular, per-screen status updates during prototype generation instead of a monolithic loading indicator:
1. Displays overall duration timer running from `startedAt`.
2. Lists individual screens with real-time status:
   - `succeeded`: Green checkmark (`✅`) with latency in seconds (e.g., `(12.3s)`).
   - `generating`: Animated spinner / refresh icon (`🔄`) with `(generating...)`.
   - `pending`: Hourglass / clock icon (`⏳`) with `(pending)`.
   - `failed`: Red alert / error icon (`❌`) with hover tooltip revealing error details.
3. Steps progress through "Preparing screen packages...", "Generating screens (X/Y)", "Assembling prototype...", "Building...", and "Uploading...".
4. Overall progress bar tracks percentage accurately (0% to 100%).
5. Polling occurs every 3 seconds while job is active (`queued` or `running`).
6. Cancel button stops running job; Retry button restarts failed/cancelled jobs.

### Automated Checks
```bash
cd charisol-design-system-fe
npm test __tests__/generation-progress.test.tsx
```

### Manual Verification Steps
1. **Start Development Servers**:
   - Backend: In `charisol-design-system-be`, run `npm run dev`.
   - Frontend: In `charisol-design-system-fe`, run `npm run dev`.
2. **Navigate to Prototype Studio**:
   - Open browser to `http://localhost:3000/projects/[projectId]`.
   - Click on the **Prototype** tab.
3. **Trigger Prototype Generation**:
   - Select 3–5 screens.
   - Click **Generate Prototype**.
4. **Observe Live Generation Progress**:
   - Verify the "Generation" card appears with a live elapsed duration counter.
   - Verify "Preparing screen packages... ✅" marks complete.
   - Verify screen list shows individual screens transitioning from `pending` -> `generating` -> `succeeded`.
   - Verify latencies display upon completion (e.g. `(14.2s)`).
   - Verify subsequent steps ("Assembling prototype...", "Building...", "Uploading...") transition to complete.
5. **Simulate Screen Error & Tooltip**:
   - Trigger a generation with an invalid node or mock error.
   - Verify the failed screen displays an alert icon with hover tooltip showing error description.
   - Verify the overall generation completes with fallback placeholder rather than crashing.

---

## Task 4.2: Screen Selection Grid & Guardrails

### Objective
Verify that screen selection is intuitive, displays thumbnails, persists across navigation, and warns against excessive screen counts:
1. Each screen card displays thumbnail rendered from the Figma snapshot assets.
2. Checkboxes allow selecting or deselecting individual screens.
3. "Select All" selects all top-level screens across all pages.
4. "Clear" / "Deselect All" deselects all screens.
5. Screen card displays dimensions (`width × height px`) and page name.
6. Selected count appears in the floating action footer (e.g., `"5 screens selected"`).
7. If `0 screens selected`, "Generate Prototype" button is disabled.
8. If `> 20 screens selected`, a prominent warning appears:
   `"⚠️ More than 20 screens selected. Generating many screens simultaneously may take several minutes and incur higher Bedrock token costs."`
9. Selection state persists in `localStorage` when reloading or switching tabs.

### Automated Checks
```bash
cd charisol-design-system-fe
npm test __tests__/screen-selection.test.tsx
```

### Manual Verification Steps
1. **Open Screens Tab**:
   - In Prototype Studio, view the **Screens** tab.
   - Verify screen thumbnails load from the snapshot asset URLs.
2. **Test Selection Controls**:
   - Click individual screen cards/checkboxes; verify border highlights with magenta accent and selection counter updates.
   - Click **Select All**; verify all screens check.
   - Click **Clear**; verify all screens uncheck and "Generate Prototype" button disables.
3. **Test Selection Persistence**:
   - Select 3 screens.
   - Navigate to another page (e.g. "Components" or "Tokens") and return to "Screens".
   - Verify the same 3 screens remain selected.
4. **Test Guardrail (>20 Screens)**:
   - Select 21+ screens (or mock snapshot with 25 screens).
   - Verify the warning box displays in the action footer and options modal before generating.

---

## Task 4.3: Preview Frame, Route Navigation & Runnable Source Zip

### Objective
Verify that the preview iframe loads generated screens, allows route navigation, displays fidelity score & report warnings, and produces a runnable zip archive with a comprehensive README:
1. Preview loads prototype via signed CloudFront URL.
2. Interactive route links at the top of the preview display all generated screens (e.g. `/`, `/about`, `/pricing`).
3. Clicking a route updates the preview iframe to that screen.
4. Fidelity score badge renders clearly (e.g. `Fidelity: 94%`).
5. Generation warnings (e.g. fallback components, missing assets) display in an alert badge list.
6. "Download source (.zip)" downloads a complete Vite+React project archive.
7. Downloaded zip contains `README.md` formatted with setup instructions and route list:
   ```markdown
   # Prototype: [Project Name]
   Generated by Strata on [Date]

   ## Setup
   npm install
   npm run dev

   ## Routes
   / — Hero Section
   /about — About Page
   /pricing — Pricing
   ```
8. Extracted zip runs locally without errors (`npm install && npm run dev`).

### Automated Checks
```bash
cd charisol-design-system-fe
npm test __tests__/preview-download.test.tsx
cd ../charisol-design-system-be
node --test tests/unit/assembler-readme.test.js
```

### Manual Verification Steps
1. **Preview Generated Prototype**:
   - Complete a generation run or open a completed job from Generation History.
   - Verify Preview Frame renders the prototype.
   - Test viewport switcher buttons ("Desktop", "Tablet", "Mobile").
2. **Test Route Navigation**:
   - Click on different route tabs at the top of the preview.
   - Verify iframe updates to render the selected screen.
3. **Verify Report & Warnings**:
   - If warnings exist, click the warnings dropdown/drawer to inspect warning codes and messages.
4. **Download and Run Zip**:
   - Click **Download source (.zip)**.
   - Save and extract `source.zip` into a test directory:
     ```bash
     unzip source.zip -d /tmp/test-prototype
     cd /tmp/test-prototype
     cat README.md
     npm install
     npm run dev
     ```
   - Open browser to Vite local server URL (e.g. `http://localhost:5173`); verify app loads and routes function.

---

## Task 4.4: Generation Mode Toggle (LLM vs Deterministic)

### Objective
Allow users to select between high-fidelity LLM generation (default) and quick deterministic generation:
1. Default mode is LLM generation ("✨ Generate Prototype").
2. Advanced / quick option is available: "⚡ Quick Generate (lower quality)" — deterministic mode.
3. Popover/tooltip explains difference:
   - LLM mode: High-fidelity components, semantic Tailwind/CSS, responsive layouts.
   - Quick mode: Instant compilation without AI inference.
4. Confirmation modal allows toggling options before starting.
5. Starting generation passes `{ mode: "llm" }` or `{ mode: "deterministic" }` in the generation request payload.
6. Both modes trigger successfully in the backend.

### Automated Checks
```bash
cd charisol-design-system-fe
npm test __tests__/generation-modes.test.tsx
```

### Manual Verification Steps
1. **Open Generation Options Modal**:
   - In Prototype Studio, click the generation trigger button.
   - Verify "LLM Generation (Default)" is selected.
   - Switch to "Quick Generate (Deterministic)" mode.
   - Verify tooltip displays explanatory text.
2. **Execute Both Modes**:
   - Trigger generation in Quick Mode: verify generation completes rapidly using deterministic compiler.
   - Trigger generation in LLM Mode: verify Step Functions / Bedrock workflow executes with full AI synthesis.

---

## Phase 4 Sign-Off Checklist

- [ ] **Task 4.1**: `GenerationProgress.tsx` displays per-screen progress, latencies, icons, and duration timer.
- [ ] **Task 4.1**: BFF route `/api/projects/[id]/prototype/jobs/[jobId]/screens` proxies to backend correctly.
- [ ] **Task 4.2**: Screen grid shows thumbnails, dimensions, select all / clear, and warns if >20 screens selected.
- [ ] **Task 4.3**: Preview frame displays route navigation tabs, fidelity score, and warning list.
- [ ] **Task 4.3**: Downloaded zip includes formatted `README.md` and runs cleanly with `npm install && npm run dev`.
- [ ] **Task 4.4**: Generation mode toggle allows selecting between LLM mode and Deterministic mode.
- [ ] **Gate**: Frontend test suite passes (`npm test`), and typecheck passes (`npm run typecheck`).
