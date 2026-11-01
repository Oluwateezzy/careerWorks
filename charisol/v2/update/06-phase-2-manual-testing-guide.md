# Phase 2 Manual Testing & Verification Guide

This guide provides end-to-end instructions for testing each component of **Phase 2: Assembly Worker & Build Pipeline**.

---

## Overview of Phase 2 Deliverables

| Task | Scope | Primary Files | Verification Command |
|---|---|---|---|
| **2.1** | Assembly Worker (`AssemblyWorkerFn`) | `charisol-design-system-be/workers/generate/assembler.js` | `node --test tests/unit/assembler.test.js` |
| **2.2** | Automated Build Error Recovery | `charisol-design-system-be/workers/generate/buildFixer.js` | `node --test tests/unit/build-fixer.test.js` |
| **2.3** | Build Pipeline in `runGenerationJob` | `charisol-design-system-be/workers/generate/index.js` | `node --test tests/unit/generation-llm-pipeline.test.js` |
| **Gate**| Phase 2 Gate End-to-End Verification | Multi-module integration | `npm test` & local Vite build execution |

---

## Task 2.1: Assembly Worker (`assembler.js`)

### Objective
Verify that `assembleProject({ workDir, screens, designTokens, brandContext, imageAssets, ... })`:
1. Scaffolds a complete Vite + React project from `templates/prototype/`.
2. Emits `src/screens/{ComponentName}.tsx` and `src/screens/{ComponentName}.module.css` for each screen.
3. Performs import analysis on screens and writes shared components into `src/components/{Name}.tsx`.
4. Generates a valid TypeScript `src/routes.tsx` mapping URL paths to screen components.
5. Emits `src/design/tokens.css` with CSS custom properties and links it in `src/main.tsx`.
6. Downloads referenced image assets to `public/assets/{hash}.{ext}` (or writes transparent fallbacks).
7. Injects Google Fonts `<link>` tags and updates Content-Security-Policy in `index.html`.
8. Generates accessible placeholder components for screens that failed LLM generation.

### Automated Checks
```bash
cd charisol-design-system-be
node --test tests/unit/assembler.test.js
```

### Manual Verification Steps
1. **Scaffolding and File Structure Inspection**:
   - Run a test assembly into a temporary folder:
     ```javascript
     const { assembleProject } = require("./workers/generate/assembler");
     await assembleProject({
       workDir: "/tmp/test-strata-app",
       screens: [
         { screenId: "1:1", name: "Landing Page", componentName: "LandingPage", tsx: "...", css: "..." },
         { screenId: "1:2", name: "Pricing Table", componentName: "PricingTable", tsx: "...", css: "..." },
         { screenId: "1:3", name: "Contact Us", componentName: "ContactUs", failed: true, reason: "Model timeout" }
       ],
       brandContext: { fonts: ["Inter", "Space Grotesk"] },
       designTokensCss: ":root { --color-primary: #2563EB; }",
     });
     ```
   - Check folder contents:
     - `src/screens/LandingPage.tsx` and `LandingPage.module.css` exist.
     - `src/screens/PricingTable.tsx` and `PricingTable.module.css` exist.
     - `src/screens/ContactUs.tsx` contains the failure notification:
       `⚠️ This screen could not be generated` with `Reason: Model timeout`.
     - `src/routes.tsx` imports all three screens and assigns `/`, `/pricing-table`, and `/contact-us`.
     - `src/design/tokens.css` contains `--color-primary: #2563EB;`.
     - `index.html` has Google Fonts links for `family=Inter` and `family=Space+Grotesk` and allows `fonts.googleapis.com` in CSP.

2. **Shared Component Auto-Detection**:
   - If a screen imports `import Header from "../components/Header"`, verify that `src/components/Header.tsx` exists (either user-provided or generated as a safe fallback component).

3. **Asset Resolution Verification**:
   - Check that images referenced by screens are located in `public/assets/` and can be resolved at `/assets/{hash}.{ext}`.

---

## Task 2.2: Build Error Recovery (`buildFixer.js`)

### Objective
Verify that `attemptBuildFix({ projectDir, buildError, buildOutput })` parses compiler error outputs from `vite build` or `tsc --noEmit` and applies targeted automated patches:
1. **Missing Imports (TS2304)**: Adds `import React from "react"`, React hooks (`useState`, `useEffect`, etc.), and router exports (`Link`, `useNavigate`, etc.).
2. **Duplicate Identifiers (TS2300)**: Renames duplicate identifiers with a `_dup` suffix on line collision.
3. **Type Errors on Props (TS2322 / TS2339 / TS7006)**: Applies `// @ts-ignore` to prevent compiler crashes on mismatched props.
4. **CSS Module Class Mismatch**: Appends empty rule `.className {}` to the corresponding CSS module.
5. **Missing Asset**: Writes a 1x1 transparent placeholder image to `public/assets/{filename}` to satisfy Vite bundler resolution.
6. **Limit Enforcement**: Returns `fixed: false` when an error is unfixable, allowing the pipeline to fail cleanly rather than looping.

### Automated Checks
```bash
cd charisol-design-system-be
node --test tests/unit/build-fixer.test.js
```

### Manual Verification Steps
1. **Missing Hook Test**:
   - Create a screen file using `useState(0)` without importing `useState`.
   - Run `attemptBuildFix` with output: `error TS2304: Cannot find name 'useState'`.
   - Confirm file is modified to include `import { useState } from "react";`.
2. **Missing Asset Test**:
   - Simulate error: `Could not resolve "/assets/hero_bg.png" from "src/screens/Hero.tsx"`.
   - Run `attemptBuildFix`.
   - Confirm `public/assets/hero_bg.png` is generated as a placeholder and `fixed: true` is returned.
3. **Unfixable Error Test**:
   - Pass an unparseable or circular dependency error: `fatal: Circular dependency detected`.
   - Confirm `fixed: false` and `fixCount: 0`.

---

## Task 2.3: Build Pipeline Integration (`workers/generate/index.js`)

### Objective
Verify that `runGenerationJob` coordinates the end-to-end LLM generation pipeline:
- Supports `options.mode: "llm"` (default) and `options.mode: "deterministic"` (fallback).
- Follows the phase lifecycle:
  - **PREPARING**: Loads snapshot and constructs `screenPackage` for selected screens.
  - **GENERATING_SCREENS**: Generates TSX/CSS per screen and updates screen progress.
  - **ASSEMBLING**: Invokes `assembleProject` to build the full Vite application.
  - **BUILDING**: Executes `npm ci` and `npx vite build --base=/p/{projectId}/{jobId}/` (with 1 auto-fix retry).
  - **VALIDATING**: Runs `npm run typecheck`, `npm run lint`, and generates preview screenshots.
  - **UPLOADING**: Bundles `source.zip`, generates `report.json`, `strata.manifest.json`, `ownership.json`, and records fidelity score.

### Automated Checks
```bash
cd charisol-design-system-be
node --test tests/unit/generation-llm-pipeline.test.js
```

### Manual Verification Steps (End-to-End Local Execution)
1. **Execute Mock LLM Pipeline**:
   ```javascript
   const { runGenerationJob } = require("./workers/generate");
   const { F1 } = require("./fixtures/figma/nodes/compilerSnapshots");

   const mockBedrockService = {
     async generateScreenCode() {
       return {
         text: `---TSX---
   import React from "react";
   import styles from "./Screen1.module.css";
   export default function Screen1() { return <div className={styles.hero}>Hello World</div>; }
   ---CSS---
   .hero { color: #2563EB; padding: 2rem; }
   ---END---`,
         inputTokens: 500,
         outputTokens: 200,
       };
     }
   };

   const res = await runGenerationJob({
     projectId: "proj_manual_test",
     jobId: "job_manual_test_01",
     snapshot: F1,
     options: { mode: "llm", bedrockService: mockBedrockService, selectedNodeIds: ["1:1"] },
     skipShell: true,
     skipWrites: true,
   });
   console.log("Status:", res.report.status);
   console.log("Directory:", res.directory);
   ```

2. **Inspect and Run Generated Vite App**:
   - Navigate to `res.directory`.
   - Extract `source.zip` if testing standalone archive:
     ```bash
     mkdir -p /tmp/extracted-app && unzip /path/to/source.zip -d /tmp/extracted-app
     cd /tmp/extracted-app
     npm install
     npm run dev
     ```
   - Open the displayed local URL (e.g., `http://localhost:5173/`).
   - Verify that:
     - The page renders without React runtime errors.
     - Design tokens from `src/design/tokens.css` apply styles.
     - Routes can be navigated.
     - Any intentionally failed screen displays the fallback card.

---

## Phase 2 Gate Acceptance Checklist

| Criteria | Verification Step | Pass Criteria |
|---|---|---|
| **Assembly file structure** | `node --test tests/unit/assembler.test.js` | All screens, routes, tokens, and assets exist |
| **Vite build succeeds** | `npm run build` in assembled project | Exit code 0, files emitted to `dist/` |
| **TypeScript check passes** | `npm run typecheck` in assembled project | Exit code 0, 0 compiler errors |
| **Runnable standalone app** | Extract `source.zip` → `npm install && npm run dev` | Dev server boots and renders in browser |
| **Image asset rendering** | Inspect network tab in preview | Image fills load with 200 OK from `/assets/` |
| **Route navigation** | Click between routes defined in `routes.tsx` | URL changes and corresponding screen mounts |
| **Failed screen resilience** | Test with a screen where LLM failed | Renders fallback `⚠️ This screen could not be generated` |
| **Deterministic compatibility** | Run tests with `mode: "deterministic"` | Existing deterministic compiler tests pass |
