# Prototype Generation v2 Update — LLM-Powered Architecture

## Problem

The current deterministic Figma-to-code compiler produces **broken, non-runnable prototypes**. Elements overlap, images are missing, typography is hardcoded, and the downloaded zip does not produce a working application. See [01-problem-analysis.md](./01-problem-analysis.md) for the full root cause analysis.

## Solution

Replace the mechanical div-translator with an **LLM-powered, screen-by-screen generation pipeline** using Amazon Bedrock (Claude). Each screen is independently converted to high-quality React code by an LLM that receives both a visual snapshot (PNG) and structured design data (JSON). Generated screens are assembled into a complete, runnable Vite + React app.

## Documents

| Document | Contents |
|---|---|
| [01-problem-analysis.md](./01-problem-analysis.md) | Deep analysis of why current output is poor. Root causes in the compiler, style module, and node data. |
| [02-architecture.md](./02-architecture.md) | New system architecture: LLM Screen Generator, Assembly Worker, Step Functions orchestration, Bedrock integration, data contracts. |
| [03-implementation-plan.md](./03-implementation-plan.md) | 5-phase implementation plan with 18 tasks, validation criteria, dependencies, and timeline. |
| [04-phase-0-manual-testing-guide.md](./04-phase-0-manual-testing-guide.md) | Comprehensive step-by-step manual testing and verification guide for Phase 0 tasks. |
| [05-phase-1-manual-testing-guide.md](./05-phase-1-manual-testing-guide.md) | Comprehensive step-by-step manual testing and verification guide for Phase 1 tasks. |
| [06-phase-2-manual-testing-guide.md](./06-phase-2-manual-testing-guide.md) | Comprehensive step-by-step manual testing and verification guide for Phase 2 tasks. |
| [07-phase-3-manual-testing-guide.md](./07-phase-3-manual-testing-guide.md) | Comprehensive step-by-step manual testing and verification guide for Phase 3 tasks. |

## Key Architectural Decisions

1. **LLM per screen, not per project** — Each screen gets its own LLM call with focused context (PNG + pruned node tree). This produces higher quality than dumping all 27 screens into one prompt.

2. **Amazon Bedrock with Claude** — AWS-native, IAM-based auth, vision support, strong React/TypeScript code generation. ~$0.02–0.04 per screen.

3. **Parallel via Step Functions** — Up to 5 screens generate concurrently. A 27-screen project completes in ~3 minutes instead of 13+ minutes.

4. **Deterministic fallback preserved** — The existing compiler becomes a fallback mode for when the LLM is unavailable or when users want fast/free generation.

5. **Visual + structured input** — The LLM receives BOTH the screen's PNG render AND the structured node tree JSON. The PNG gives visual context; the JSON gives exact values (colors, fonts, dimensions).

## Phases

| Phase | Duration | Summary |
|---|---|---|
| **Phase 0: Foundation** | ~1 week | Expand node data in plugin + snapshot loader, set up Bedrock, create prompt templates |
| **Phase 1: Screen Generation** | ~1.5 weeks | Build screen package builder, LLM screen generator, code validator, progress tracking |
| **Phase 2: Assembly & Build** | ~1 week | Assembly worker, build error recovery, integrate into generation pipeline |
| **Phase 3: Step Functions** | ~1 week | State machine, Lambda packaging, API wiring, cost controls |
| **Phase 4: Frontend** | ~1 week | Per-screen progress UI, screen selection, preview updates, mode toggle |
| **Phase 5: Quality** | ~1 week | Visual fidelity scoring, prompt iteration, observability, integration tests |

**Total estimated duration: ~6.5 weeks**
