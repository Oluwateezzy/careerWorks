# Strata v2 Plan: Figma Plugin → Prototype, Brand Context, GitHub Handoff

| Doc | What it covers |
|---|---|
| [`01-architecture.md`](./01-architecture.md) | Goals, the current-state audit with file and line evidence, key decisions, data contracts (snapshot schema v2, DynamoDB, API), sequence diagrams, renderer and compiler design, security, infrastructure, and risks |
| [`02-implementation-plan.md`](./02-implementation-plan.md) | Phases 0–7 with tasks, task validation, phase exit criteria, parallel tracks, fixtures, and requirement traceability |

Supersedes the Figma-REST parts of [`../1789987065356-strata-live-prototype-plan.md`](../1789987065356-strata-live-prototype-plan.md). See architecture §1, "Relationship to the v1 plan".

## The workflow in one paragraph

The user **connects the Strata Figma plugin** once using a device code approved in the web app. In Figma they click **Push to Strata**. The plugin streams every page, node, variable (local and library), style, component, prototype interaction, and asset to S3 in chunks. Strata ingests the push into an immutable **Figma snapshot** with read-optimized digests. The **Prototype page** renders everything in the snapshot (screens, flows, components, tokens, assets, layers, and warnings) and offers **Generate Prototype**, which deterministically builds a React and Vite app that you can preview inside Strata and download. The **Brand Context Engine**'s Figma source reads the same snapshot, with no URL or token. **Push to GitHub** sends the generated prototype to a branch and PR, using a fixed OAuth flow.

## Why the screenshots look wrong today

| Symptom | Root cause | Fixed in |
|---|---|---|
| Total Nodes: 10,000 | Hard cap `maxNodes \|\| 10000` in `charisol-figma-plugin/src/walker/treeWalker.ts` L43, and truncation is always reported as `false` | Tasks 0.3, 1.4 |
| Variables 0 / Styles 0 | Reads only *local* variables and styles, swallows errors silently, and never captures node `boundVariables` or style IDs | Tasks 0.3, 1.5 |
| "Prototype Job ID" after export | The upload creates a fake generation job that is already `ready-for-review` | Tasks 2.2, 1.3 |
| "No GitHub OAuth token found for user" | `GET /github/oauth/authorize` is unauthenticated, so `userId` is empty, the callback uses `"anonymous"`, and the token is **never saved**. Success is shown anyway. | Tasks 6.1, 6.2, 6.4 |

## Recommended order

1. **Phase 0** (foundation and honesty fixes). Ship **Task 0.3** right away.
2. In parallel: **Track A** (Phases 1 → 2 → 3 → 4), **Track B** (Phase 6 A, the GitHub OAuth fix), and **Track C** (Phase 5, after Phase 2).
3. **Phase 6 B** (push the generated prototype) after Phase 4.
4. **Phase 7** (hardening and rollout).
