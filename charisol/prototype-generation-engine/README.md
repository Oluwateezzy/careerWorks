# Prototype Generation Engine Architecture & Implementation Plan

## Executive Summary

This documentation outlines the complete, production-grade architectural redesign and step-by-step implementation plan for the **Strata Prototype Generation Engine** in `charisol-design-system-be`.

The core architectural shift ensures that **only one box in the entire pipeline is an LLM call** (generating the React layout and screen component structure). All other tasks—token extraction, SVG icon deduplication and optimization, state machine orchestration, compilation validation, and build assembly—are handled by deterministic, zero-token backend code that runs once per prototype or once per screen.

---

## Directory Contents

| Document | Focus | Description |
| :--- | :--- | :--- |
| [01-architecture-and-contracts.md](file:///Users/oluwatobiloba/Desktop/charisol/design_system/docsplan/prototype-generation-engine/01-architecture-and-contracts.md) | **System Architecture & Data Contracts** | End-to-end pipeline architecture, data contracts between stages, token budget models, prompt caching layout, and the generator retry/recovery decision tree. |
| [02-implementation-steps.md](file:///Users/oluwatobiloba/Desktop/charisol/design_system/docsplan/prototype-generation-engine/02-implementation-steps.md) | **Step-by-Step Implementation Guide** | Phase-by-phase implementation instructions, file-by-file changes, exact code diffs, testing checklists, and rollout verification steps. |

---

## Architectural Principles

1. **Zero Changes to the Figma Plugin**:
   The Figma plugin continues to export the snapshot payload (containing `nodesById`, `variables`, `tokens`, `imageAssetMap`, and vector geometries) to S3. The backend ingests the snapshot as-is.

2. **Deterministic Preprocessing Over LLM Token Generation**:
   - Design tokens (`tokens.css`) and common vector icons (`icons.tsx`) are generated **once** during preprocessing and placed into a shared S3 location.
   - The LLM never invents CSS variables or hand-codes hundreds of lines of inline SVG bezier coordinates. It simply imports `<IconName />` and references `var(--token-name)`.

3. **High Token Ceiling with Thinking Disabled**:
   - `maxTokens` raised from `4096` to `16384`.
   - Claude Haiku 5.5 / Claude 3.7 **Adaptive Thinking is explicitly disabled** (`thinking: { type: "disabled" }`), preventing internal reasoning loops from consuming the output token budget and eliminating the 0-byte output trap.

4. **Multi-Tiered Recovery and Degraded States**:
   - If an output is truncated at the token limit, the generator uses **assistant continuation prefill** to complete the code.
   - If the TSX is valid but the CSS is truncated, a fast repair call is triggered or baseline styles are injected, marking the screen as `"degraded"` instead of failing the prototype.
   - Bedrock throttling is caught and retried natively in AWS Step Functions with exponential backoff and jitter.

5. **Prompt Caching Efficiency**:
   - Prompts are structured with stable system instructions and shared prototype manifests first (marked with ephemeral cache points), followed by the screen's pruned node tree last.
   - Reduces effective input token latency and costs by 60–80%.
