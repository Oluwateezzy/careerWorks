# Prototype Builder — incremental, chat-driven prototype generation

## Goal

Make generating a prototype feel like building an app with a teammate, not like submitting a batch job:

1. **Generate the first pages**, then come back and **add more pages** until the app is done.
2. **Never rebuild a page that was already generated** unless its design changed.
3. **Change the app without regenerating it**: swap an image or asset, change a font or colour, fix copy. This is the designer → developer handoff problem we want to solve.
4. **Figma is optional**: users can upload **screenshots + assets** instead of using the plugin.
5. Every generation or update returns **what changed and how to test it**.
6. **GitHub handoff**: each version can open a pull request containing the prototype source.

The frontend redesign is done, in `charisol-design-system-fe` on branch `Prototype-generation`. It is built against the contracts in this folder. When an endpoint is missing it degrades gracefully: a 404/405/501 or `NOT_IMPLEMENTED` shows "isn't enabled on this server yet". This document set covers the **backend work** needed to light those features up.

## Directory contents

| Document | Focus |
| :--- | :--- |
| [01-architecture.md](./01-architecture.md) | The new frontend structure, and the backend architecture: per-screen manifest and reuse, append and edit strategies, change reports, image sources, GitHub push. |
| [02-api-contracts.md](./02-api-contracts.md) | Exact request and response shapes the FE already sends and reads: new fields on existing endpoints, plus four new endpoints. |
| [03-implementation-plan.md](./03-implementation-plan.md) | Phased, file-by-file backend steps with tests and exit criteria. |

## What already exists (backend)

The backend scan was done on 2026-10-22.

| Capability | Status |
| :--- | :--- |
| Figma snapshot → per-screen LLM generation → Vite build → signed preview | ✅ (`workers/generate/**`, Step Functions) |
| Job list across snapshots (`GET /prototype/jobs?workflowType=generation`) | ✅ (`snapshotId` is optional) |
| Routes manifest with `nodeIds` per route | ✅ (`workers/generate/assembler.js:504`), which the FE uses for "already in app" |
| Assets registry `assets.ts` + `public/figma` | ✅ from figma-asset-export Phase 4, and the basis for asset swaps |
| GitHub App connect, repo picker, branch + PR publish | ✅, but `publish` commits placeholder design-system files, **not the prototype** |
| Incremental / append generation, screen reuse | ❌ every job is a full fresh build |
| Targeted edits without regeneration | ❌ |
| Change report / test steps | ❌ |
| Screenshot (non-Figma) source | ❌ the vision model only sees Figma renders |
| Instructions / notes passed to the generator | ❌ |

## Priority order

**Phase 1 (serialization) → Phase 2 (screen manifest + reuse + append + change report) → Phase 3 (GitHub push) → Phase 4 (targeted edits) → Phase 5 (screenshot sources).**

Phase 2 is the core of the product idea. The FE already sends `strategy: "append"` and the union of the existing and new node IDs, so today's backend still produces a correct, complete app. It is just slower and regenerates pages it should reuse. Once Phase 2 lands, the same request reuses the existing pages.
