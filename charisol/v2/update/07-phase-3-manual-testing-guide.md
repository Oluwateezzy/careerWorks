# Phase 3 Manual Testing & Verification Guide

This guide provides end-to-end instructions for testing and validating **Phase 3: Step Functions Orchestration & Parallel Generation**.

---

## Overview of Phase 3 Deliverables

| Task | Scope | Primary Files | Verification Command |
|---|---|---|---|
| **3.1** | Step Functions State Machine | `cdk/lib/generate-state-machine.ts`<br>`cdk/lib/charisol-design-system-stack.ts` | `npx cdk synth --no-staging` |
| **3.2** | Generation Lambda Handlers | `workers/generate/lambda/prepareScreenPackages.js`<br>`workers/generate/lambda/generateScreen.js`<br>`workers/generate/lambda/assemblePrototype.js` | `node --test tests/unit/generation-lambdas.test.js` |
| **3.3** | API Controller & SFN Wiring | `controllers/prototypeController.js`<br>`services/stepFunctionsService.js` | `node --test tests/unit/prototype-step-functions.test.js` |
| **3.4** | Cost Tracking & Daily Limits | `services/llmCostTracker.js` | `node --test tests/unit/llm-cost-tracker.test.js` |
| **Gate**| Phase 3 Gate End-to-End | Integration suite | `node --test tests/integration/phase3-step-functions-e2e.test.js` |

---

## Task 3.1: Step Functions State Machine (`generate-state-machine.ts`)

### Objective
Verify that the Step Functions state machine:
1. Coordinates the three pipeline steps: `PrepareScreenPackages` -> `GenerateScreensMap` (Map state, max concurrency 5) -> `AssemblePrototype`.
2. Employs exponential backoff retry on `ThrottlingException` for the `GenerateScreen` task.
3. Catches individual screen failures with `ScreenFailed` pass state so a single screen error does not abort the entire execution.
4. Correctly routes to `Succeeded` on completion or `Failed` on unrecoverable errors.

### Automated Checks
```bash
cd charisol-design-system-be/cdk
npm run build
npx cdk synth --no-staging
```

### Manual Verification Steps
1. **Inspect CloudFormation Synthesis**:
   - Inspect the synthesized state machine definition in `cdk/cdk.out/dev-charisol-design-system-be.template.json`.
   - Verify `Type: "AWS::StepFunctions::StateMachine"` has `GenerateScreensMap` with `"MaxConcurrency": 5`.
   - Verify `Iterator.States.GenerateScreen.Catch` transitions to `ScreenFailed`.
2. **AWS Console Visual Graph Inspection (in deployed environments)**:
   - Navigate to AWS Step Functions in the AWS Management Console.
   - Open the state machine `*-prototype-workflow`.
   - Review the Workflow Studio / Graph Inspector:
     - `PrepareScreenPackages` connects to `GenerateScreensMap`.
     - `GenerateScreensMap` displays parallel sub-execution branches.
     - `AssemblePrototype` collects results and transitions to `Succeeded`.

---

## Task 3.2: Generation Lambda Handlers (`workers/generate/lambda/`)

### Objective
Verify that the three modular Lambda handlers execute independently and handle inputs/outputs as expected by the state machine:
1. `prepareScreenPackages.handler(event)`: Loads snapshot, builds pruned screen packages, initializes DynamoDB screen statuses, and returns `{ screenPackages, totalScreens, ... }`.
2. `generateScreen.handler(event)`: Updates screen status to `"generating"`, invokes Bedrock (or mock), tracks token metrics, and returns `{ componentName, tsx, css, status, ... }`.
3. `assemblePrototype.handler(event)`: Assembles project directory, writes screens, tokens, and assets, builds Vite app, and uploads `source.zip` and reports to S3.

### Automated Checks
```bash
cd charisol-design-system-be
node --test tests/unit/generation-lambdas.test.js
```

### Manual Verification Steps
1. **Prepare Handler Local Test**:
   ```javascript
   const { handler: prepareHandler } = require("./workers/generate/lambda/prepareScreenPackages");
   const result = await prepareHandler({
     projectId: "proj_test_prepare",
     jobId: "job_prepare_01",
     snapshotId: "snap_f1",
     selectedNodeIds: ["1:1"],
   });
   console.log("Prepared screens:", result.screenPackages.length);
   ```
2. **Screen Generator Handler Local Test**:
   ```javascript
   const { handler: screenHandler } = require("./workers/generate/lambda/generateScreen");
   const result = await screenHandler({
     projectId: "proj_test_prepare",
     jobId: "job_prepare_01",
     screenPackage: { screenId: "1:1", screenName: "Hero", nodeTree: { id: "1:1" } },
   });
   console.log("Screen result status:", result.status);
   ```

---

## Task 3.3: API Controller & Cancellation Wiring (`prototypeController.js`)

### Objective
Verify that `POST /v1/projects/:projectId/prototype/generate`:
1. Starts the Step Functions execution with `{ projectId, jobId, snapshotId, selectedNodeIds }`.
2. Returns HTTP 202 Accepted with `{ jobId, executionArn }`.
3. Records `PROTOTYPE_GENERATE_STARTED` audit event.
4. Allows mid-flight cancellation: `POST /v1/projects/:projectId/prototype/jobs/:jobId/cancel` stops the Step Functions execution and marks the job `CANCELLED`.

### Automated Checks
```bash
cd charisol-design-system-be
node --test tests/unit/prototype-step-functions.test.js
```

### Manual Verification Steps
1. **Trigger Generation via API**:
   ```bash
   curl -X POST http://localhost:3000/v1/projects/proj_demo/prototype/generate \
     -H "Authorization: Bearer <TOKEN>" \
     -H "Content-Type: application/json" \
     -d '{"snapshotId": "snap_123", "options": { "mode": "llm" }}'
   ```
   - Expect HTTP 202 Accepted with `{ "status": true, "data": { "jobId": "job_...", "executionArn": "arn:..." } }`.
2. **Poll Status**:
   ```bash
   curl http://localhost:3000/v1/projects/proj_demo/prototype/jobs/<JOB_ID> \
     -H "Authorization: Bearer <TOKEN>"
   ```
   - Expect job state `queued` -> `running` with phase updates.
3. **Cancel Mid-Flight**:
   ```bash
   curl -X POST http://localhost:3000/v1/projects/proj_demo/prototype/jobs/<JOB_ID>/cancel \
     -H "Authorization: Bearer <TOKEN>"
   ```
   - Expect HTTP 200 with job marked `cancelled`.
   - In AWS Console, verify Step Functions execution is aborted with `USER_CANCELLED`.

---

## Task 3.4: Cost Tracking & Daily Project Limits (`llmCostTracker.js`)

### Objective
Verify cost tracking and guardrails:
1. Calculates costs for Claude 3.5 Sonnet / Bedrock pricing ($0.003/1K input, $0.015/1K output).
2. Computes and records token metrics in job summaries.
3. Enforces daily limits: when a project exceeds `PROTOTYPE_DAILY_LIMIT_PER_PROJECT` (default 10), subsequent requests return HTTP 429 Too Many Requests.
4. Per-screen token budget: validates screen package payload size and flags oversized requests (> 15,000 tokens).

### Automated Checks
```bash
cd charisol-design-system-be
node --test tests/unit/llm-cost-tracker.test.js
```

### Manual Verification Steps
1. **Cost Calculation**:
   - Input: 5,000 input tokens, 1,000 output tokens.
   - Cost: `(5000 / 1000) * 0.003 + (1000 / 1000) * 0.015` = `$0.015 + $0.015` = `$0.030`.
2. **Rate Limit / Daily Cap Test**:
   - Send 10 generation requests for the same `projectId`.
   - Send an 11th generation request on the same calendar day.
   - Verify the 11th request returns HTTP 429:
     ```json
     {
       "status": false,
       "error": "DAILY_LIMIT_EXCEEDED",
       "message": "Daily generation limit of 10 reached for project."
     }
     ```

---

## Phase 3 Gate Acceptance Checklist

| Criteria | Verification Step | Pass Criteria |
|---|---|---|
| **Parallel screen generation** | Run Map state with 5 test screens | Concurrency reaches 5 simultaneous screen generations |
| **End-to-end API pipeline** | `POST /generate` -> poll -> completion | Job finishes `succeeded` and outputs valid `source.zip` |
| **Resilience to partial failures** | 1 screen out of 5 fails LLM | Remaining 4 generate normally; failed screen shows placeholder |
| **Cost tracking in job record** | Check `job.tokenSummary` & `job.cost` | Total input/output tokens and cost in USD recorded |
| **Cancellation halts execution** | Call `POST /jobs/:id/cancel` while running | Execution stops immediately; DynamoDB records `cancelled` |
| **Generation speedup** | 10 screens generation benchmark | Parallel Map completes in under 5 minutes (< 300s) |
