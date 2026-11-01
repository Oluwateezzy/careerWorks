# Phase 0 Manual Testing & Verification Guide

This guide provides end-to-end instructions for testing each component of **Phase 0: Foundation — Enhanced Node Data & Bedrock Integration**.

---

## Overview of Phase 0 Deliverables

| Task | Scope | Primary Files | Verification Command |
|---|---|---|---|
| **0.1** | Figma Plugin Node Serialization | `charisol-figma-plugin/src/walker/nodeSerializer.ts`<br>`charisol-figma-plugin/src/types/messages.ts` | `npm run typecheck && npm run build`<br>`npm test`<br>`npm run sim` |
| **0.2** | Snapshot Loader Normalization | `charisol-design-system-be/workers/generate/snapshotLoader.js` | `node --test tests/unit/generate-snapshot-loader.test.js` |
| **0.3** | Bedrock Converse API & IAM | `charisol-design-system-be/services/bedrockService.js`<br>`charisol-design-system-be/cdk/lib/charisol-design-system-stack.ts` | `node --test tests/unit/bedrock-service.test.js`<br>`npm run build` in `cdk` |
| **0.4** | LLM Prompt Templates | `charisol-design-system-be/workers/generate/prompts/` | `node --test tests/unit/prompt-templates.test.js` |

---

## Task 0.1: Figma Plugin Node Serialization

### Objective
Ensure the Figma plugin exports the complete visual fidelity of every node, including typography, multi-type fills, strokes with alignment, shadow/blur effects, corner radii, opacity, layout constraints, auto-layout flex wrapping, and overflow clipping.

### Automated Checks
Run the following commands inside `charisol-figma-plugin/`:
```bash
cd charisol-figma-plugin
npm run typecheck
npm run build
npm test
npm run sim
```

### Manual Testing in Figma Desktop / Web App
1. **Load Unpacked Plugin in Figma**:
   - Open Figma Desktop or Figma in Google Chrome.
   - Go to **Plugins** -> **Development** -> **Import plugin from manifest...**.
   - Select `/Users/oluwatobiloba/Desktop/charisol/design_system/charisol-figma-plugin/manifest.json`.

2. **Prepare a Test Canvas with Varied Styles**:
   Create a frame containing:
   - **Text Node**: Custom font (e.g. Inter 600 SemiBold), 24px, 32px line height, 1.5px letter spacing, uppercase/lowercase, center-aligned, text decoration (underline).
   - **Solid Fill Rectangle**: `#2563EB` with 80% opacity.
   - **Gradient Fill Frame**: Linear gradient with 2+ stops and custom angle.
   - **Image Fill Rectangle**: A rectangle filled with an image fill.
   - **Strokes**: 2px stroke with `INSIDE` or `OUTSIDE` alignment and custom color.
   - **Effects**: Drop shadow (offset X: 0, Y: 4, blur: 12, spread: 0, color: rgba(0,0,0,0.15)) and Background Blur.
   - **Corner Radii**: Uniform corner radius (e.g., 12px) and independent corner radii (e.g., 16px, 16px, 0, 0).
   - **Auto-layout Frame**: Horizontal layout with `layoutWrap: "WRAP"`, `counterAxisAlignItems: "CENTER"`, and custom gap/padding.

3. **Export and Inspect**:
   - Run the Strata plugin.
   - Click **Export Snapshot**.
   - Inspect the emitted manifest or check the simulated export output via:
     ```bash
     npm run sim
     ```
   - Verify that:
     - `node.fontFamily`, `node.fontSize`, `node.fontWeight`, `node.letterSpacing`, `node.lineHeight` are present.
     - `node.fills` contains all stops for gradients and `imageRef` hash for images.
     - `node.strokes` includes color, weight, and align.
     - `node.effects` includes offset, radius, spread, color, and blur types.
     - `node.cornerRadius` or `node.cornerRadii` are captured.
     - `node.clipsContent` and `node.overflow` match Figma frame settings.

---

## Task 0.2: Snapshot Loader Normalization

### Objective
Verify that `snapshotLoader.js` no longer strips visual styling properties and generates a valid `imageAssetMap` mapping imageRef hashes to presigned S3 URLs.

### Automated Checks
Run the snapshot loader test suite inside `charisol-design-system-be/`:
```bash
cd charisol-design-system-be
node --test tests/unit/generate-snapshot-loader.test.js
```

### Manual Verification
1. **Create/Inspect a Rich Snapshot Fixture**:
   - In a scratch script or test file, call `buildCompilerSnapshot` with a raw node containing:
     - Solid, gradient, and image fills.
     - Text typography (`fontFamily: "Inter"`, `fontSize: 16`, `fontWeight: 600`).
     - Strokes with `weight: 2` and `align: "INSIDE"`.
     - Drop shadow effect.
     - `cornerRadii: { topLeft: 12, topRight: 12, bottomRight: 0, bottomLeft: 0 }`.
   - Assert that the output in `snapshot.nodesById[id]` preserves all of these properties verbatim.

2. **Verify `imageAssetMap`**:
   - For any node with an image fill (`imageRef: "hash_abc123"`), verify `snapshot.imageAssetMap["hash_abc123"]` is resolved to a valid presigned S3 URL or asset path.

---

## Task 0.3: Amazon Bedrock Access & SDK Wrapper

### Objective
Verify that `charisol-design-system-be/services/bedrockService.js` correctly calls Amazon Bedrock's Converse API, passes system prompts and image attachments, tracks token metrics and latency, and handles errors gracefully.

### Automated Checks
Run the dedicated Bedrock service test:
```bash
cd charisol-design-system-be
node --test tests/unit/bedrock-service.test.js
```

### CDK IAM Policy Verification
Synthesize and check the CDK stack:
```bash
cd charisol-design-system-be/cdk
npm run build
```
Verify that the `generateTaskRole` contains the policy statement:
```json
{
  "Effect": "Allow",
  "Action": [
    "bedrock:InvokeModel",
    "bedrock:InvokeModelWithResponseStream"
  ],
  "Resource": "arn:aws:bedrock:*::foundation-model/anthropic.claude-*"
}
```

### Manual Live Bedrock Test (With Active AWS Credentials)
When AWS credentials and Amazon Bedrock model access (Claude 3.5 Sonnet) are configured in the environment:
1. Run a test invocation script:
   ```bash
   node -e "
     const { generateScreenCode } = require('./services/bedrockService');
     (async () => {
       const res = await generateScreenCode({
         systemPrompt: 'You are a test assistant. Reply with OK.',
         userMessage: 'Test prompt',
         maxTokens: 50,
       });
       console.log('Result:', res);
     })().catch(console.error);
   "
   ```
2. Verify the output:
   - `content` contains text response from Claude.
   - `inputTokens > 0` and `outputTokens > 0`.
   - `latencyMs` is a positive number (typically 800ms - 2500ms).

3. **Error Handling Test**:
   - Provide an invalid model ID (e.g. `anthropic.non-existent-model`).
   - Confirm the function throws a normalized error with `statusCode: 404` or `isNotFound: true` rather than crashing unhandled.

---

## Task 0.4: LLM Prompt Templates

### Objective
Ensure prompt template files exist, load correctly, and reliably substitute all template placeholders without exceeding token limits.

### Automated Checks
Run the prompt templates test:
```bash
cd charisol-design-system-be
node --test tests/unit/prompt-templates.test.js
```

### Manual Inspection Checklist
1. **System Prompt (`workers/generate/prompts/system.md`)**:
   - Role clearly specified: "senior React/TypeScript UI developer".
   - Output delimiters defined: `---TSX---`, `---CSS---`, `---END---`.
   - Architecture constraints enforced: React 18, TypeScript, CSS Modules, no inline styles, no Tailwind, no arbitrary external libraries.
   - 2 concrete before/after examples mapping Figma node JSON to React + CSS Module code.

2. **User Message Template (`workers/generate/prompts/user-template.md`)**:
   - Placeholders present:
     - `{{SCREEN_NAME}}`
     - `{{NODE_TREE}}`
     - `{{TOKENS_CSS}}`
     - `{{ASSET_MANIFEST}}`
     - `{{INTERACTIONS}}`

3. **Template Renderer (`workers/generate/prompts/index.js`)**:
   - Call `renderPrompt({ screenName: 'Home', nodeTree: {...}, tokensCss: '...', assetManifest: {...}, interactions: [] })`.
   - Confirm that all placeholders are replaced (no unrendered `{{...}}` brackets remaining).
   - Estimate token count (rough rule of thumb: ~4 characters per token):
     - Confirm baseline prompt size without large subtree is under 2,500 tokens.
     - With a standard 100-node screen subtree, total input remains under 8,000 tokens.

---

## Phase 0 Gate Checklist

Before proceeding to Phase 1, verify each gate condition:

- [ ] **Plugin Typecheck & Build**: `npm run typecheck && npm run build` succeeds in `charisol-figma-plugin`.
- [ ] **Plugin Tests**: All unit tests in `charisol-figma-plugin` pass (`npm test`).
- [ ] **Snapshot Normalization**: Unit tests pass in backend (`node --test tests/unit/generate-snapshot-loader.test.js`).
- [ ] **Bedrock Service**: Unit tests pass in backend (`node --test tests/unit/bedrock-service.test.js`).
- [ ] **Prompt Templates**: Unit tests pass in backend (`node --test tests/unit/prompt-templates.test.js`).
- [ ] **CDK Build**: `npm run build` succeeds in `charisol-design-system-be/cdk`.
- [ ] **All Backend Tests**: Existing test suite continues to pass (`node --test tests/unit/*.test.js`).
