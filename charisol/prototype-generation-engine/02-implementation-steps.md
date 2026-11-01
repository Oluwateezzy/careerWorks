# Step-by-Step Implementation Guide

## Implementation Roadmap Overview

The refactor is organized into six sequential phases to ensure continuous testability and zero downtime:

```
┌────────────────────────────────────────────────────────────────────────┐
│ Phase 1: Engine Parameters & Logging (Immediate Unblocker)            │
│   • Raise maxTokens to 16,384                                          │
│   • Disable Claude thinking in Bedrock Converse                        │
│   • Add structured observability logging                               │
├────────────────────────────────────────────────────────────────────────┤
│ Phase 2: Generator Retry & Recovery Matrix                             │
│   • Continuation on truncation (stopReason: "max_tokens")              │
│   • CSS repair & degraded fallback                                     │
│   • Screen status contract ("ok" | "degraded" | "failed")              │
├────────────────────────────────────────────────────────────────────────┤
│ Phase 3: Step Functions & CDK Infrastructure Updates                   │
│   • Raise Lambda and Step Functions task timeouts to 5 minutes         │
│   • Configure Step Functions retry on ThrottlingException              │
├────────────────────────────────────────────────────────────────────────┤
│ Phase 4: Shared Token & Icon Extraction (Preprocessing)                 │
│   • Compile shared tokens.css in prepareScreenPackages                 │
│   • Deduplicate & SVGO-optimize vector icons into icons.tsx            │
│   • Update prompt templates to reference shared icons & tokens         │
├────────────────────────────────────────────────────────────────────────┤
│ Phase 5: Prototype Assembly & Transpilation Verification               │
│   • Inject shared tokens.css & icons.tsx into Vite template            │
│   • Transpile verification & degraded screen handling                  │
│   • Emit prototype-status.json                                         │
├────────────────────────────────────────────────────────────────────────┤
│ Phase 6: Verification & Test Execution                                 │
│   • Run backend unit test suite (679+ tests)                           │
│   • End-to-end dry run with compiler snapshots (snap_f1, snap_f2, etc)│
└────────────────────────────────────────────────────────────────────────┘
```

---

## Phase 1: Engine Parameters & Logging (Immediate Unblocker)

### Step 1.1: Update `services/bedrockService.js`
Modify the Bedrock service to:
1. Increase default `maxTokens` from `4096` to `16384`.
2. Explicitly disable thinking for Claude 3.7 and Claude Haiku 5.5 models via `additionalModelRequestFields`.
3. Support continuation prefill calls (accepting `assistantPrefill`).
4. Emit structured attempt logs.

#### File: `charisol-design-system-be/services/bedrockService.js`
```javascript
// Change DEFAULT_MAX_TOKENS
const DEFAULT_MAX_TOKENS = Number(process.env.BEDROCK_MAX_TOKENS) || 16384;

// Inside generateScreenCode():
// 1. Configure thinking disabled for Claude 3.7 / Haiku 5.5
const additionalModelRequestFields = {};
const isThinkingCapable =
  effectiveModelId.includes("haiku-5-5") ||
  effectiveModelId.includes("claude-3-7");

if (isThinkingCapable) {
  additionalModelRequestFields.thinking = { type: "disabled" };
}

// 2. Build messages array with optional assistant prefill for continuation
const messages = [
  { role: "user", content: buildContent(userMessage, imageBase64) },
];

if (assistantPrefill) {
  messages.push({
    role: "assistant",
    content: [{ text: assistantPrefill }],
  });
}

// 3. Send ConverseCommand
const commandInput = {
  modelId: effectiveModelId,
  system: systemPrompt ? [{ text: String(systemPrompt) }] : undefined,
  messages,
  inferenceConfig,
};

if (Object.keys(additionalModelRequestFields).length > 0) {
  commandInput.additionalModelRequestFields = additionalModelRequestFields;
}

const response = await client.send(new ConverseCommand(commandInput));
```

---

## Phase 2: Generator Retry & Recovery Matrix

### Step 2.1: Update `workers/generate/screenGenerator.js`
Refactor `screenGenerator.js` to implement the multi-tiered recovery decision tree.

#### Key Enhancements:
1. **Continuation on Truncation**:
   When `bedrockResponse.stopReason === "max_tokens"`, capture `partialContent`, pass it as `assistantPrefill`, and request completion without re-sending the whole generation prompt.
2. **Targeted CSS Repair Call**:
   If the TSX block was parsed cleanly but the CSS block is missing or cut off, invoke a lightweight repair call with the component's class names:
   ```javascript
   const cssRepairPrompt = `The following React component was generated. Generate ONLY the corresponding CSS Module styles within ---CSS--- and ---END--- delimiters.\n\nComponent:\n${tsx}`;
   ```
3. **Degraded State Fallback**:
   If CSS repair fails, inject baseline fallback styles:
   ```css
   /* Auto-generated baseline styles for degraded screen */
   .screen {
     display: flex;
     flex-direction: column;
     width: 100%;
     min-height: 100vh;
     background: var(--color-background, #ffffff);
     color: var(--color-text, #111827);
   }
   ```
   Mark status as `"degraded"`.
4. **Structured Return Object**:
   ```javascript
   return {
     success: true,
     screenId: screenPackage.screenId,
     screenName: screenPackage.screenName,
     componentName,
     tsx,
     css,
     status: isDegraded ? "degraded" : "ok",
     degradationReason: isDegraded ? degradationReason : null,
     metrics: {
       inputTokens: totalInputTokens,
       outputTokens: totalOutputTokens,
       latencyMs: totalLatencyMs,
       attempts,
       stopReasons,
     },
   };
   ```

### Step 2.2: Update `workers/generate/lambda/generateScreen.js`
Update the Lambda handler to:
1. Preserve and forward `status: "ok" | "degraded" | "failed"`.
2. If `err.name === "ThrottlingException"` or `err.statusCode === 429`, **rethrow the error directly** so AWS Step Functions can execute native exponential backoff instead of terminating the screen.

---

## Phase 3: Step Functions & Infrastructure Updates

### Step 3.1: Update `cdk/lib/generate-state-machine.ts`
1. **Raise Lambda Task Timeout**: Change task timeout from 120s to 300s (5 minutes).
2. **Step Functions Native Throttling Retry**:
   Add specific retry handling for `ThrottlingException` with exponential backoff and jitter.

#### File: `cdk/lib/generate-state-machine.ts`
```typescript
// Inside createGenerateStateMachineChain():
const generateScreenTask = new tasks.LambdaInvoke(scope, `${stackId}-sfn-generate-screen`, {
  lambdaFunction: screenGeneratorFn,
  payloadResponseOnly: true,
  taskTimeout: sfn.Timeout.duration(cdk.Duration.minutes(5)),
  comment: "Generate React TSX and CSS for a screen via Bedrock",
});

// Configure Bedrock throttling retry with jitter
generateScreenTask.addRetry({
  errors: ["ThrottlingException", "BedrockThrottled", "TooManyRequestsException"],
  interval: cdk.Duration.seconds(5),
  maxAttempts: 4,
  backoffRate: 2.0,
  jitterFull: true,
});

// General transient retry
generateScreenTask.addRetry({
  errors: ["States.Timeout", "States.TaskFailed"],
  interval: cdk.Duration.seconds(3),
  maxAttempts: 2,
  backoffRate: 2.0,
});
```

### Step 3.2: Update `cdk/lib/charisol-design-system-stack.ts`
Update the `screenGeneratorFn` function definition:
```typescript
const screenGeneratorFn = new lambda.Function(this, `${stackId}-screen-generator-fn`, {
  // ...
  timeout: cdk.Duration.minutes(5), // Raised from 2 minutes
  memorySize: 2048,
});
```

---

## Phase 4: Shared Token & Icon Extraction (Preprocessing)

### Step 4.1: Icon Extraction & Deduplication Utility
Create `workers/generate/iconExtractor.js` to extract vector graphics from the snapshot without touching the Figma plugin.

#### File: `workers/generate/iconExtractor.js`
```javascript
"use strict";

const crypto = require("crypto");

/**
 * Normalizes and extracts vector icons from Figma snapshot nodes.
 * Deduplicates by path geometry hash and rewrites fill to currentColor.
 *
 * @param {Object} nodesById
 * @returns {{ iconsTsx: string, iconNames: string[] }}
 */
function extractSharedIcons(nodesById = {}) {
  const iconMap = new Map(); // hash -> { name, svgContent }

  for (const [nodeId, node] of Object.entries(nodesById)) {
    if (!node || node.visible === false) continue;

    // Check if node is an icon candidate (VECTOR node or component under 48x48)
    const isSmall = node.bounds && node.bounds.width <= 48 && node.bounds.height <= 48;
    const isVectorLike = node.type === "VECTOR" || node.type === "BOOLEAN_OPERATION" || node.vectorPaths;

    if (isSmall && (isVectorLike || /icon/i.test(node.name || ""))) {
      const cleanName = sanitizeIconName(node.name || `Icon_${nodeId}`);
      const svgPath = extractPathData(node);
      if (!svgPath) continue;

      const pathHash = crypto.createHash("sha256").update(svgPath).digest("hex").slice(0, 8);
      if (!iconMap.has(pathHash)) {
        iconMap.set(pathHash, {
          name: cleanName,
          width: Math.round(node.bounds?.width || 24),
          height: Math.round(node.bounds?.height || 24),
          path: svgPath,
        });
      }
    }
  }

  // Generate icons.tsx
  const iconNames = [];
  const componentDecls = [];

  for (const { name, width, height, path } of iconMap.values()) {
    iconNames.push(name);
    componentDecls.push(`
export function ${name}(props: React.SVGProps<SVGSVGElement>) {
  return (
    <svg width="${width}" height="${height}" viewBox="0 0 ${width} ${height}" fill="none" stroke="currentColor" {...props}>
      <path d="${path}" strokeLinecap="round" strokeLinejoin="round" />
    </svg>
  );
};`);
  }

  const iconsTsx = `import React from 'react';\n\n${componentDecls.join("\n")}\n`;
  return { iconsTsx, iconNames };
}

function sanitizeIconName(name) {
  const clean = name
    .replace(/[^a-zA-Z0-9]/g, " ")
    .split(/\s+/)
    .map((w) => w.charAt(0).toUpperCase() + w.slice(1))
    .join("");
  return clean.startsWith("Icon") ? clean : `Icon${clean}`;
}

function extractPathData(node) {
  if (node.vectorPaths && Array.isArray(node.vectorPaths) && node.vectorPaths[0]?.data) {
    return node.vectorPaths[0].data;
  }
  return null;
}

module.exports = {
  extractSharedIcons,
  sanitizeIconName,
};
```

### Step 4.2: Update `workers/generate/lambda/prepareScreenPackages.js`
1. Call `extractSharedIcons(source.nodesById)`.
2. Compile `tokens.css` from `source.variables` / `source.tokens`.
3. Upload `shared/tokens.css` and `shared/icons.tsx` to S3.
4. Pass `tokenNames` and `iconNames` in `jobContextKey` for use in the prompt.

### Step 4.3: Update Prompts for Prompt Caching
In `workers/generate/prompts/user-template.md`:
* Replace instruction asking for inline `<svg>` with:
  ```markdown
  ### Available Shared Icons
  Import needed icons directly from `./icons`:
  {{AVAILABLE_ICONS}}

  Example:
  import { IconLock, IconClose } from './icons';

  DO NOT write raw inline <svg> path coordinates. Always use the available icon components.
  ```

---

## Phase 5: Prototype Assembly & Transpilation Verification

### Step 5.1: Update `workers/generate/assembler.js`
1. **Place Shared Assets**:
   * Write `src/design/tokens.css` from preprocessed tokens.
   * Write `src/components/icons.tsx` from preprocessed icons.
2. **Verify TSX Syntax**:
   * Before writing to disk, run a regex/syntax check on the TSX.
   * Normalize any residual `react-router-dom` imports to `react-router`.
3. **Handle Screen Statuses**:
   * If `status === "ok"`: write screen TSX and CSS module.
   * If `status === "degraded"`: write screen TSX and fallback CSS module; add to assembly warnings.
   * If `status === "failed"`: generate failed screen placeholder component.
4. **Emit Status Report**:
   * Generate `prototype-status.json` and upload to S3:
     ```json
     {
       "projectId": "...",
       "jobId": "...",
       "status": "ready",
       "summary": { "total": 4, "ok": 3, "degraded": 1, "failed": 0 },
       "screens": [
         { "screenId": "54:4217", "name": "BrowserWindow", "status": "ok", "latencyMs": 8200 }
       ]
     }
     ```

---

## Phase 6: Verification & Test Execution

### Step 6.1: Run Backend Test Suites
Run the full unit test suite to ensure zero regressions:
```bash
cd /Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-be
npm run test:unit
```

### Step 6.2: Validate with Compiler Fixtures
Execute the test runner on the compiler snapshots (`snap_f1`, `snap_f2`, `snap_f5`, `snap_f6`):
```bash
npx jest tests/unit/workers/generate/screenGenerator.test.js
npx jest tests/unit/workers/generate/assembler.test.js
npx jest tests/unit/workers/generate/prepareScreenPackages.test.js
```

### Step 6.3: PR Deployment & Live Generation Test
1. Push branch `Prototype-generation` to GitHub.
2. Trigger deployment workflow.
3. Open Strata Studio on the deployed PR URL.
4. Select screen `54:4217 (Browser window)` and click **Generate Prototype**.
5. Verify in CloudWatch logs:
   * `BEDROCK_MODEL_ID="us.anthropic.claude-haiku-5-5"`
   * `StopReason: "end_turn"`
   * `ContentLength > 4000` (both TSX and CSS extracted cleanly)
   * Prototype builds and renders with full interactive preview.
