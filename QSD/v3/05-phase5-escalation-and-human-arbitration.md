# Phase 5 — Escalation Router & Human Arbitration Layer

> **Document ID**: `VESC-P5-V3-007`  
> **Phase Target**: Multi-Tier Escalation Routing, Automated Case Dossier Generator, Blind Arbiter Workspace, and WebCrypto M-of-N Verdict Signing  
> **Subsystems**: `vescrow_backend` (Go/Fiber) & `vescrow_admin` (Next.js/React/TS)  
> **Status**: Ready for Implementation  
> **Dependencies**: Phase 1 (Bounded Enforcement), Phase 2 (Intake & Anti-Bias), Phase 3 (Triage & Confidence)

---

## 1. Executive Summary

Phase 5 implements the **Human Arbitration Layer**—the critical "in-the-loop" fail-safe of the Vescrow dispute ecosystem. When automated triage detects subjective terms, high financial value, or anomalous/fraudulent signals, the **Escalation Router** dispatches the ticket to a tiered queue of vetted human neutrals.

To maximize arbiter efficiency and eliminate cognitive bias, the system generates an executive **Case Dossier** with automated discrepancy detection and enforces a **Blind Review Mode** by default. Finally, to eliminate single points of operational failure, arbiters render verdicts using **WebCrypto M-of-N Cryptographic Signing Panels**, collecting non-repudiable digital signatures before the Bounded Enforcement Layer permits fund disbursement.

---

## 2. Detailed Task Breakdown

### Task 5.1: Escalation Router, SLA Manager & Tiered Queue Backend

- **Goal**: Deterministically assign escalated disputes to appropriate arbiter pools with strict SLA enforcement.
- **Files to Create**:
  - `internal/dispute/escalation/router.go`
  - `internal/dispute/escalation/router_test.go`
- **Implementation Details**:
  1. Define Tier Assignment Rules:
     - **Tier 1 (Standard Operations)**:
       - Escrow Amount $< \text{₦}250,000$.
       - Standard product return / condition dispute with $C_s \ge 0.50$.
       - Quorum requirement: 1 authorized Tier 1+ arbiter. SLA: 24 hours.
     - **Tier 2 (High-Value / Complex)**:
       - Escrow Amount between $\text{₦}250,000$ and $\text{₦}2,000,000$, OR milestone software deliverable, OR $C_s < 0.50$.
       - Quorum requirement: 2 distinct signatures from Tier 2+ arbiters. SLA: 12 hours.
     - **Tier 3 (Fraud & Critical Legal Risk)**:
       - Escrow Amount $> \text{₦}2,000,000$, OR duplicate evidence detected, OR account flagged for chargeback abuse.
       - Quorum requirement: 2-of-3 signatures including Platform Compliance Officer. SLA: 6 hours.
  2. Implement Arbiter Ticket Lifecycle:
     - `ClaimDispute(ctx context.Context, disputeID, arbiterID uuid.UUID) error`:
       - Locks ticket to the claiming arbiter for a 2-hour exclusive evaluation window.
     - Auto-Reassignment Cron: If an arbiter claims a ticket but does not submit a proposed verdict within 2 hours, releases the lock back to the active pool and logs an arbiter inactivity penalty.

- **Task Validation Criteria**:
  - A ₦1,500,000 dispute is automatically assigned `tier_2` with `required_signatures = 2` and a 12-hour deadline.
  - A Tier 1 arbiter attempting to claim a Tier 2 ticket receives `HTTP 403 Forbidden: Insufficient arbiter tier authorization`.

---

### Task 5.2: Case Dossier & Discrepancy Generator

- **Goal**: Synthesize transaction history, carrier events, and evidence into an actionable executive summary for arbiters.
- **Files to Create**:
  - `internal/dispute/dossier/generator.go`
  - `internal/dispute/dossier/generator_test.go`
- **Implementation Details**:
  1. Define `CaseDossier`:
     ```go
     type CaseDossier struct {
         DisputeID          uuid.UUID
         EscrowSummary      EscrowSummaryDTO
         ConfidenceScore    float64
         RiskTier           string
         RequiredSignatures int
         BlindTokens        map[string]string // "buyer": "Disputant Alpha", "seller": "Disputant Beta"
         Discrepancies      []DiscrepancyItem
         Timeline           []TimelineEvent
         ExternalFindings   []VerificationFinding
         AIRecommendation   ProposedOutcome
     }
     ```
  2. Automated Discrepancy Extractor:
     - Evaluates discrepancies between party claims and verified logs:
       - *Weight Mismatch*: Carrier depot logged 1,200g, while buyer claims empty box weighing 120g.
       - *Timeline Deviation*: Buyer reported damage 5 days after delivery was logged by carrier.
       - *Identity Inconsistency*: Waybill recipient signature does not match registered buyer initials.
  3. AI Recommendation Engine:
     - Synthesizes all data points into a proposed distribution with explainability rationale (e.g. *"Suggest 70% refund to buyer due to verified carrier transit damage; 30% to seller as salvage value"*).

- **Task Validation Criteria**:
  - Test case with contradictory carrier weight correctly populates the `Discrepancies` array with exact weight figures.
  - Output contains zero unmasked PII when retrieved under blind mode.

---

### Task 5.3: WebCrypto Key Management & Client Signer in Admin Portal

- **Goal**: Enable arbiters to generate and manage Ed25519 or ECDSA P-256 signing keys locally in browser secure storage.
- **Files to Create**:
  - `src/lib/crypto/webcrypto-signer.ts` (in `vescrow_admin`)
  - `src/lib/crypto/key-manager.ts` (in `vescrow_admin`)
- **Implementation Details**:
  1. In `key-manager.ts`:
     - Generates non-extractable signing key pair using `window.crypto.subtle.generateKey()` with algorithm `Ed25519` (or `ECDSA` with P-256 curve where Ed25519 is unsupported).
     - Exports public key in SPKI PEM format; computes SHA-256 public key fingerprint.
     - Registers public key with backend: `POST /api/v1/admin/signing-keys/register`.
     - Persists private key handle in browser `IndexedDB` protected by browser-level PIN / WebAuthn biometric prompt.
  2. In `webcrypto-signer.ts`:
     - Implements `signSettlementIntent(intent: CanonicalSettlementIntent): Promise<ArbiterSignature>`:
       - Serializes canonical JSON and computes SHA-256 hash.
       - Signs the hash with the stored private key.
       - Returns signature in hexadecimal format along with the arbiter's key fingerprint.

- **Task Validation Criteria**:
  - Key generation in Chrome and Safari produces a valid public key and fingerprint registered cleanly with the backend.
  - Signing an intent produces a signature that successfully passes Go `ed25519.Verify()` on the backend.

---

### Task 5.4: Arbiter Workspace UI: Dossier, Timeline & Blind Review Mode

- **Goal**: Build a modern, high-efficiency adjudication interface in `vescrow_admin`.
- **Files to Create**:
  - `src/components/dispute/case-dossier-card.tsx` (in `vescrow_admin`)
  - `src/components/dispute/discrepancy-feed.tsx` (in `vescrow_admin`)
  - `src/components/dispute/timeline-scrubber.tsx` (in `vescrow_admin`)
  - `src/components/dispute/evidence-lightbox.tsx` (in `vescrow_admin`)
- **Files to Modify**:
  - `src/app/(portal)/disputes/[id]/page.tsx`
- **Implementation Details**:
  1. Top Section: Dossier Summary Card:
     - Displays Dispute ID, Amount in Naira, Risk Tier badge, SLA countdown, and AI Confidence Score meter.
     - Blind Mode Switch: Enabled by default. Toggling off requires entering a mandatory justification reason and logs an audit event.
  2. Middle Section: Discrepancy & Evidence Comparison Grid:
     - Left column: Disputant Alpha (Buyer) claims & verified evidence.
     - Right column: Disputant Beta (Seller) claims & verified evidence.
     - Embedded `evidence-lightbox.tsx`: Allows deep image zoom, color inversion, EXIF inspection, and side-by-side photo comparison.
     - `discrepancy-feed.tsx`: Callout cards highlighting detected contradictions (e.g. weight delta, tracking discrepancies).
  3. Bottom Section: Chronological Timeline Scrubber:
     - Visual vertical timeline combining platform events, carrier tracking logs, and dispute comments.

- **Task Validation Criteria**:
  - Navigating to `/disputes/[id]` loads the dossier in $< 1$ second.
  - Blind mode reliably replaces names with `Disputant Alpha` and `Disputant Beta` and masks phone numbers.
  - Clicking an evidence image opens the lightbox with full zoom and EXIF metadata panel.

---

### Task 5.5: Multi-Sig Verdict Co-Signing Panel in Admin Portal

- **Goal**: Implement the multi-sig verdict drafting and co-signing interface in `vescrow_admin`.
- **Files to Create**:
  - `src/components/dispute/multisig-signing-panel.tsx` (in `vescrow_admin`)
- **Files to Modify**:
  - `src/lib/api/admin.ts`
- **Implementation Details**:
  1. Verdict Formulation Form:
     - Radio selection: `Full Refund to Buyer` | `Full Release to Seller` | `Custom Split`.
     - Interactive split slider: real-time Naira and percentage updates. Sum must equal escrow amount.
     - Mandatory Adjudication Rationale input (minimum 50 characters).
  2. Signing Quorum Tracker:
     - Visual multi-sig progress indicator:
       - *Tier 1*: `[ ( ) Signature 1 ]`
       - *Tier 2*: `[ (✓) Arbiter #1 (Signed) ] ── [ ( ) Arbiter #2 (Pending) ]`
       - *Tier 3*: `[ (✓) Arbiter #1 ] ── [ (✓) Compliance Lead ] ── [ ( ) Legal Dir ]`
  3. Signing Actions:
     - "Sign & Propose Verdict": Arbiter #1 signs via WebCrypto. Creates pending `DisputeExecutionProof`.
     - "Co-Sign Verdict": Arbiter #2 reviews proposed breakdown, inspects diff, and adds their signature.
     - Final signature automatically submits proof to the Bounded Enforcement Layer.

- **Task Validation Criteria**:
  - Attempting to submit a split that doesn't sum to the total disables the sign button with a real-time warning.
  - Signing prompts the WebCrypto key and updates the progress indicator from 1/2 to 2/2.

---

### Task 5.6: Backend Verdict Submission & Multi-Sig Quorum Controller

- **Goal**: Receive arbiter signatures, enforce quorum constraints, and trigger atomic ledger settlement upon quorum completion.
- **Files to Create**:
  - `internal/dispute/enforcement/controller.go`
  - `internal/dispute/enforcement/controller_test.go`
- **Implementation Details**:
  1. `POST /api/v1/admin/disputes/:id/verdict/propose`:
     - Validates claiming arbiter's key.
     - Creates `DisputeExecutionProof` with 1 signature.
     - If `required_signatures == 1`: directly invokes `verifier.VerifyExecutionProof()` and executes settlement.
     - If `required_signatures > 1`: marks proof `pending_co_signatures` and alerts the Tier 2/3 arbiter pool.
  2. `POST /api/v1/admin/disputes/:id/verdict/co-sign`:
     - Verifies second arbiter is not the same person as Arbiter #1.
     - Validates second arbiter signature over canonical intent hash.
     - Appends signature to JSONB array.
     - When `signatures_collected == required_signatures`: executes settlement and marks dispute resolved.

- **Task Validation Criteria**:
  - Arbiter attempting to co-sign their own proposed verdict receives `HTTP 409 Conflict: Co-signer must be a distinct authorized arbiter`.
  - When second signature is posted on Tier 2 case, system verifies both signatures and successfully executes wallet fund disbursement.

---

## 3. Phase 5 Overall Validation & Exit Gate

The phase is completed when the following full escalation cycle passes:

```
[TEST SUITE: Phase 5 Human Arbitration Gate]
1. Escrow #301 (₦1,200,000) escalated as Tier 2 case.
2. Arbiter Alice logs into vescrow_admin -> Claims ticket.
3. Alice inspects dossier in Blind Mode -> Reviews discrepancy feed.
4. Alice proposes 60/40 split with rationale -> Signs with WebCrypto Key Alice.
5. System records Proof #PRF-01 with 1/2 signatures (status: pending_co_signatures).
6. Arbiter Bob logs in -> Reviews Alice's proposed verdict and dossier.
7. Bob clicks "Co-Sign Verdict" -> Signs with WebCrypto Key Bob.
8. System detects 2/2 signatures -> ProofVerifier validates both keys.
9. Wallet executes: ₦720,000 to Buyer, ₦480,000 to Seller.
10. Escrow status updates to 'partial_release' and dispute marked 'resolved'.
```
