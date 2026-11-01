# System Architecture & Stage Contracts

## 1. End-to-End Pipeline Architecture

The Strata Prototype Generation Pipeline orchestrates the transformation of raw Figma snapshots into a production-grade, interactive React + Vite web application preview. 

### High-Level Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ 1. FIGMA PLUGIN (Unmodified)                                                │
│    • Ingests Figma canvas & generates snapshot.json                         │
│    • Uploads snapshot (nodesById, variables, tokens, imageAssetMap) to S3   │
└──────────────────────────────────────┬──────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ 2. BACKEND PREPROCESSING (prepareScreenPackages Lambda - Runs ONCE)         │
│    • Extracts & compiles tokens.css ──► s3://{bucket}/jobs/{jobId}/shared/  │
│    • Extracts, deduplicates & SVGO-optimizes vector icons ──► icons.tsx     │
│    • Prunes NodeTree per screen ──► s3://.../screens/{id}/package.json      │
│    • Generates shared prompt context (token names list, icon names list)    │
│    • Emits slim screen pointers to Step Functions Map State                 │
└──────────────────────────────────────┬──────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ 3. STEP FUNCTIONS MAP STATE (screensMap: concurrency 3-5, timeout 5m)       │
│    • Native Step Functions Retry with exponential backoff on Throttling     │
│    • ToleratedFailurePercentage ensures one failure doesn't abort workflow  │
│    │                                                                        │
│    ▼ (Iterates per screen ID)                                               │
│    ┌───────────────────────────────────────────────────────────────────┐    │
│    │ PER-SCREEN GENERATION (screenGenerator Lambda - The ONLY LLM Call)│    │
│    │ • Downloads package.json from S3 pointer                          │    │
│    │ • Constructs Prompt with Prompt Caching points                    │    │
│    │ • Bedrock Converse (Claude Haiku 5.5):                            │    │
│    │     - maxTokens: 16,384                                           │    │
│    │     - thinking: { type: "disabled" }                              │    │
│    │ • Recovery Decision Tree:                                         │    │
│    │     - Truncation ──► Assistant continuation prefill               │    │
│    │     - Missing CSS ──► CSS repair / fallback degraded CSS          │    │
│    │     - Throttled ──► Thrown to Step Functions                      │    │
│    │ • Returns: { tsx, css, status: "ok" | "degraded" | "failed" }     │    │
│    └───────────────────────────────────────────────────────────────────┘    │
└──────────────────────────────────────┬──────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ 4. PROTOTYPE ASSEMBLY (assemblePrototype Lambda - Runs ONCE)                │
│    • Injects shared tokens.css into src/design/tokens.css                   │
│    • Injects shared icons.tsx into src/components/icons.tsx                 │
│    • Verifies syntax & compiles screens into Vite prototype template        │
│    • Handles degraded/failed screen placeholders                            │
│    • Executes Vite build, packages artifacts, and deploys preview to S3     │
│    • Emits prototype-status.json with per-screen health report              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Stage Details & Responsibilities

### Stage 1: Figma Plugin (Unmodified)
* **Contract**:
  * Uploads `snapshot.json` to S3 via presigned URL: `projects/{projectId}/snapshots/{snapshotId}.json`.
  * Snapshot contains:
    * `nodesById`: Normalized Figma node records including type, bounds, fills, strokes, and child pointers.
    * `variables` / `tokens`: Color, typography, spacing variables.
    * `imageAssetMap`: S3 URLs for raster images.
    * Vector nodes: Nodes containing vector geometry paths and fills.
* **Invariant**: The Figma plugin code is **not touched**. All extraction and adaptation logic runs server-side.

---

### Stage 2: Backend Preprocessing (`prepareScreenPackages`)
*Execution context: AWS Lambda (`prepareScreenPackagesFn`), 2048 MB, 2 min timeout, runs ONCE per job.*

1. **Shared Token Extraction**:
   * Reads Figma variables and styles from snapshot.
   * Compiles complete CSS custom properties (`:root { --color-primary: #...; }`).
   * Writes `tokens.css` to `projects/{projectId}/jobs/{jobId}/shared/tokens.css`.
   * Produces a token summary list (e.g., `["--color-primary", "--spacing-md", "--font-heading"]`) for LLM prompt context.

2. **Shared Icon Extraction & Optimization**:
   * Inspects all components and vector nodes across screens.
   * Identifies icon candidates: nodes of type `VECTOR`, `BOOLEAN_OPERATION`, or components with bounds $\le 48\times 48\text{px}$.
   * Runs **SVGO** optimization:
     * `floatPrecision: 1`
     * Rewrites fill and stroke to `currentColor` for monochrome icons.
     * Computes SHA-256 hash of normalized SVG paths to deduplicate across screens.
   * Generates a single TypeScript icon registry: `icons.tsx` export containing `<IconName />` components.
   * Writes `icons.tsx` to `projects/{projectId}/jobs/{jobId}/shared/icons.tsx`.
   * Produces an icon manifest for LLM prompt context: `["IconLock", "IconClose", "IconChevronDown", "IconSearch"]`.

3. **Per-Screen NodeTree Pruning**:
   * Uses `pruneNodeTree()` with:
     * Maximum depth: 8.
     * Maximum children per node: 20.
     * Repeated sibling collapse: If $\ge 3$ structurally identical siblings exist (e.g. card lists), keep 2 and note `count: N`.
     * Strips invisible layers and internal vector path blobs (replaces vector subtrees with clean component references).
   * Writes pruned screen package to S3: `projects/{projectId}/jobs/{jobId}/screens/{screenId}/package.json`.

4. **Step Functions Map Payload Generation**:
   * Emits lightweight screen pointers (average payload $< 5\text{ KB}$):
     ```json
     [
       {
         "screenId": "54:4217",
         "screenName": "Browser window",
         "componentName": "BrowserWindow",
         "packageKey": "projects/proj_1/jobs/job_1/screens/54_4217/package.json",
         "pngKey": "projects/proj_1/snapshots/snap_1/renders/54_4217.png",
         "jobContextKey": "projects/proj_1/jobs/job_1/shared/job-context.json"
       }
     ]
     ```

---

### Stage 3: Per-Screen Generation (`screenGenerator`)
*Execution context: AWS Lambda (`screenGeneratorFn`), 2048 MB, 5 min timeout, executed concurrently in Step Functions Map.*

#### 1. Bedrock Converse Parameters
* **Model ID**: `us.anthropic.claude-haiku-5-5` (or configured via `BEDROCK_MODEL_ID`).
* **Max Tokens**: `16,384` (was 4,096).
* **Thinking Configuration**:
  ```json
  "additionalModelRequestFields": {
    "thinking": { "type": "disabled" }
  }
  ```
* **Temperature**: Omitted for Claude Haiku 5.5 / 3.7 to prevent API validation errors.

#### 2. Prompt Layout & Prompt Caching Strategy
Anthropic Prompt Caching on Amazon Bedrock allows caching prompt prefixes that are identical across invocations. Because all screens in a prototype share the exact same role, tokens list, and icon registry, we structure the messages into cached blocks:

```
┌────────────────────────────────────────────────────────────────────────┐
│ Block 1: System Prompt (CACHED)                                        │
│   • Role: Expert React TypeScript & CSS Module engineer                │
│   • Output Contract: Strict ---TSX--- ... ---CSS--- ... ---END---      │
│   • Rules: No inline SVGs (import from './icons'), use CSS variables   │
│   • Cache Point 1: ephemeral cache control                             │
├────────────────────────────────────────────────────────────────────────┤
│ Block 2: Prototype Shared Manifests (CACHED per prototype)             │
│   • Available CSS variables: var(--color-bg), var(--font-base), ...    │
│   • Available Shared Icons: <IconLock />, <IconClose />, ...           │
│   • Prototype Route Interactions: { "button_id": "/dest-route" }       │
│   • Cache Point 2: ephemeral cache control                             │
├────────────────────────────────────────────────────────────────────────┤
│ Block 3: Screen-Specific Content (NOT CACHED)                          │
│   • Screen Name & Dimensions: "BrowserWindow" (1440x900)               │
│   • Pruned NodeTree JSON for this specific screen                      │
│   • Visual screenshot image attachment (PNG bytes)                     │
└────────────────────────────────────────────────────────────────────────┘
```

#### 3. Generator Retry & Recovery Decision Flow

```
                              [Bedrock Converse Call]
                                         │
                                         ▼
                             Did Bedrock throw error?
                              /                    \
                            YES                     NO
                            /                         \
            Is ThrottlingException?             Check response.stopReason
             /                  \                       │
           YES                   NO                     ▼
           /                       \          Is stopReason == "max_tokens"?
    Rethrow to Step          Rethrow error      /                         \
    Functions Retry          to mark screen    YES                         NO
    (Exponential backoff)    failed            /                             \
                                      Truncated output                 Extract blocks:
                                             │                         ---TSX--- ... ---CSS---
                                             ▼                                   │
                                    Append partial output as                     ▼
                                    assistant prefill & ask               Are TSX & CSS valid?
                                    for continuation (max 2)              /                 \
                                                                        YES                  NO
                                                                        /                      \
                                                                   Run syntax            Is TSX ok but
                                                                   validation            CSS cut off?
                                                                        │                 /         \
                                                                        ▼               YES          NO
                                                                   Status: "ok"         /              \
                                                                                 CSS Repair Call    TSX Syntax Error
                                                                                 (CSS only)         (transpile fail)
                                                                                  /        \               │
                                                                               SUCCESS    FAIL             ▼
                                                                                 /          \        Retry prompt with
                                                                           Status: "ok"   Fallback   compiler errors
                                                                                          baseline   (max 1 retry)
                                                                                          styles
                                                                                            │
                                                                                       Status: "degraded"
```

* **Failure Mode 1: Output Truncated (`stopReason === "max_tokens"`)**:
  * Instead of discarding the output and re-running from scratch, take the partial TSX/CSS text, pass it as an assistant prefill, and prompt the model: `"Please continue directly from where this code was cut off to complete the component and CSS module."` (Capped at 2 continuations).
* **Failure Mode 2: TSX Intact, CSS Missing or Cut Off**:
  * If the TSX component is complete but `---CSS---` is missing or truncated, trigger a lightweight repair call:
    `"The TSX component was generated successfully. Given the class names used in this TSX, output ONLY the CSS module within ---CSS--- and ---END---."`
  * If the repair call fails, inject baseline fallback CSS (`display: flex; box-sizing: border-box;`) and mark the screen as `"degraded"`.
* **Failure Mode 3: TSX Fails Compilation**:
  * If the TSX contains unclosed tags or syntax errors, run one retry including the compiler error message in the prompt.
* **Failure Mode 4: Bedrock Throttled**:
  * Throw the error directly. Step Functions handles retries with backoff and jitter without consuming Lambda execution time.

---

### Stage 4: Assembly & Compilation (`assemblePrototype`)
*Execution context: AWS Lambda (`assemblePrototypeFn`), 2048 MB, 10 min timeout, runs ONCE per job.*

1. **Shared File Placement**:
   * Scaffolds the clean prototype template into `/tmp/strata-generate/{jobId}/`.
   * Places `tokens.css` into `src/design/tokens.css`.
   * Places `icons.tsx` into `src/components/icons.tsx`.
2. **Screen Placement & Transpilation Verification**:
   * Iterates through the generated screens from the Map state.
   * For each screen:
     * If `status === "ok"`: Writes `${componentName}.tsx` and `${componentName}.module.css`.
     * If `status === "degraded"`: Writes `${componentName}.tsx` and fallback `${componentName}.module.css`. Logs warning.
     * If `status === "failed"`: Generates clear placeholder component showing failure details.
   * Performs fast syntax verification on each screen file using `ts.transpileModule` (or Babel parser).
3. **Build & Bundle Execution**:
   * Uses cached `node_modules` from template layer.
   * Runs `vite build --base=/p/{projectId}/{jobId}/`.
   * Packages bundle into `dist.zip` and uploads to preview S3 bucket.
4. **Health Report Generation**:
   * Writes `prototype-status.json` recording:
     * Overall build status: `ready | degraded | failed`.
     * Total screens count, ok count, degraded count, failed count.
     * Per-screen metrics: latency, token usage, attempts, and degradation reasons.

---

## 3. Data Contracts Between Stages

| Boundary | Producer | Consumer | Payload Structure | Notes |
| :--- | :--- | :--- | :--- | :--- |
| **Plugin $\to$ S3** | Figma Plugin | Backend S3 | `{ nodesById, variables, tokens, imageAssetMap }` | Ingested as raw snapshot JSON via presigned URL. |
| **Stage 2 $\to$ S3** | `prepareScreenPackages` | Step Functions / S3 | `s3://.../shared/tokens.css`<br>`s3://.../shared/icons.tsx`<br>`s3://.../screens/{id}/package.json` | Heavy assets offloaded to S3. |
| **Stage 2 $\to$ Map** | `prepareScreenPackages` | Step Functions Map | `[{ screenId, screenName, componentName, packageKey, pngKey, jobContextKey }]` | Lightweight pointers ($< 5\text{ KB}$ per item). |
| **Map $\to$ Lambda** | Step Functions | `generateScreen` | `{ screenPackage: <pointer>, projectId, jobId, options }` | Offloaded package fetched from S3 on entry. |
| **Lambda $\to$ Bedrock** | `screenGenerator` | Amazon Bedrock | System prompt (cached) + Prototype manifest (cached) + Screen NodeTree | Converse API payload with `thinking: { type: "disabled" }`. |
| **Bedrock $\to$ Parser** | Amazon Bedrock | `screenGenerator` | `---TSX---\n[TSX code]\n---CSS---\n[CSS code]\n---END---` | Strict delimiter contract. |
| **Parser $\to$ Map** | `screenGenerator` | Step Functions Map | `{ screenId, componentName, tsx, css, status, attempts, stopReasons[], metrics }` | `status` is `"ok"`, `"degraded"`, or `"failed"`. |
| **Map $\to$ Stage 4** | Step Functions Map | `assemblePrototype` | `{ screens: [...screenResults], projectId, jobId, jobContextKey }` | Aggregated array of all screen results. |
| **Stage 4 $\to$ Preview** | `assemblePrototype` | S3 / User | `dist.zip`, deployed preview URL, `prototype-status.json` | Final runnable application. |

---

## 4. Token Budget Analysis

| Metric | Previous Architecture | Redesigned Architecture | Rationale |
| :--- | :--- | :--- | :--- |
| **Input Tokens** | ~20,000 tokens (raw tree + unoptimized vectors) | **~8,000–10,000 tokens** (pruned tree; icons & tokens referenced by name) | Aggressive subtree pruning and referencing shared tokens/icons removes repetitive JSON nodes. |
| **Input Cache Hit** | 0% (no caching) | **~70–80% cached** | System prompt and shared manifests are identical across all screens in the prototype. |
| **Output: Inline SVGs**| 3,000–6,000 tokens (thousands of raw bezier points) | **~0 tokens** | Screens simply import `<IconLock />`, `<IconClose />` from `./icons`. |
| **Output: TSX Layout** | 2,500–4,000 tokens | **2,500–4,000 tokens** | Clean React TypeScript component with full DOM layout and semantic tags. |
| **Output: CSS Module** | 1,500–3,000 tokens | **1,500–2,500 tokens** | Clean CSS module with flexbox/grid and token variable references. |
| **Output: Reasoning** | Up to 4,096 tokens (consumed 100% on retries) | **0 tokens** (`thinking: disabled`) | Eliminates reasoning token waste and eliminates 0-byte output leaks. |
| **Configured MaxTokens**| 4,096 tokens (Routinely exhausted) | **16,384 tokens** | Generous 16k ceiling provides comfortable headroom. |
| **Average Latency** | 25–40 seconds (or timeouts) | **6–12 seconds** | No reasoning overhead, fewer output tokens, fast streaming. |

---

## 5. Observability Schema

Every generation attempt emits a structured log event in CloudWatch:

```json
{
  "timestamp": "2026-10-20T14:35:00.000Z",
  "event": "screen_generation_attempt",
  "jobId": "proto-job_gen_1792497397801_igpe2",
  "screenId": "54:4217",
  "screenName": "Browser window",
  "attempt": 1,
  "modelId": "us.anthropic.claude-haiku-5-5",
  "stopReason": "end_turn",
  "inputTokens": 8420,
  "outputTokens": 3210,
  "cacheReadTokens": 7100,
  "cacheWriteTokens": 0,
  "latencyMs": 7450,
  "hasReasoningBlock": false,
  "parseResult": "ok",
  "tsxLength": 4120,
  "cssLength": 2840,
  "validationErrorsCount": 0,
  "finalStatus": "ok"
}
```

### CloudWatch Metric Alarms
1. **DegradedOrFailedScreensAlarm**: Fires if $> 5\%$ of screens finish with `status === "degraded"` or `status === "failed"` over a 15-minute window.
2. **TokenCeilingApproachingAlarm**: Fires if p95 `outputTokens / maxTokens` exceeds $70\%$ (11,400 tokens), alerting before screens start encountering truncation.
