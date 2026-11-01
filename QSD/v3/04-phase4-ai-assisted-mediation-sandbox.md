# Phase 4 — AI-Assisted Mediation Sandbox

> **Document ID**: `VESC-P4-V3-006`  
> **Phase Target**: Bounded Compromise Generator, Real-Time Disputant Negotiation Room, Turn-Based Timers, and Mutual Agreement Settlement  
> **Subsystems**: `vescrow_backend` (Go/Fiber) & `vescrow_frontend` (Next.js/React/TS)  
> **Status**: Ready for Implementation  
> **Dependencies**: Phase 1 (Bounded Enforcement), Phase 2 (Intake), Phase 3 (Triage & Confidence)

---

## 1. Executive Summary

Phase 4 introduces the **AI-Assisted Mediation Sandbox**, providing disputing parties with a bounded, structured negotiation space where common friction points can be resolved amicably without human arbiter intervention. When a dispute is tagged as moderate ambiguity ($0.60 \le C_s < 0.90$), the engine computes **Pareto-Optimal Compromise Proposals** (e.g., partial refund for minor defects, return-and-refund, or replacement terms).

Parties interact within a dedicated **Mediation Room** on the user frontend. If both parties cryptographically sign the same compromise terms within a strict 48-hour turn-based countdown, the dispute transitions to `mediation_resolved` and triggers instant automated ledger settlement via the Bounded Enforcement Layer. If negotiations break down or the timer expires, the case is escalated automatically to the Human Arbitration Layer.

---

## 2. Detailed Task Breakdown

### Task 4.1: Mediation Sandbox Backend Engine & Proposal Generator

- **Goal**: Implement business logic for generating algorithmic compromise proposals and managing negotiation state.
- **Files to Create**:
  - `internal/dispute/mediation/service.go`
  - `internal/dispute/mediation/service_test.go`
- **Implementation Details**:
  1. Define proposal generation heuristics:
     - **Heuristic 1 (Minor Defect / Partial Repair)**:
       - If category is `damaged_item` or `not_as_described`, propose keeping the item with a $20\% - 30\%$ refund allowance to the buyer for repair/salvage, releasing $70\% - 80\%$ to the seller.
     - **Heuristic 2 (Return for Full Refund)**:
       - Propose $100\%$ refund to the buyer conditional upon verified return shipment via platform-integrated carrier.
     - **Heuristic 3 (Milestone Partial Completion)**:
       - For freelance/service escrows with partial commits, propose split proportional to verified completed milestones.
  2. Implement `GenerateInitialProposals(ctx context.Context, dispute *Dispute) ([]DisputeMediationProposal, error)`:
     - Creates pre-computed proposal records in `dispute_mediation_proposals` table.
     - Sets `expires_at = time.Now().Add(48 * time.Hour)`.
  3. Implement `SubmitCounterOffer(ctx context.Context, disputeID, partyID uuid.UUID, buyerKobo, sellerKobo int64, description string) (*DisputeMediationProposal, error)`:
     - Validates that `buyerKobo + sellerKobo == escrow.AmountKobo`.
     - Validates that neither amount is negative.
     - Enforces a maximum of 3 counter-proposals per party to prevent infinite stalling loops.

- **Task Validation Criteria**:
  - Unit test verifies proposal generation produces valid splits summing to the exact escrow amount.
  - Submitting an invalid counter-offer (e.g. sum $\ne$ escrow total, or negative amount) is rejected with `HTTP 400 Bad Request`.
  - Submitting a 4th counter-offer by the same party is blocked with `HTTP 429 Too Many Requests: Counter-offer limit reached for this session`.

---

### Task 4.2: Mediation REST API Handlers & Routes

- **Goal**: Expose endpoints for disputants to retrieve mediation state, accept proposals, submit counter-offers, or request escalation.
- **Files to Create**:
  - `internal/dispute/mediation/handler.go`
  - `internal/dispute/mediation/routes.go`
- **Implementation Details**:
  1. Endpoints:
     - `GET /api/v1/disputes/:id/mediation`: Returns active proposals, suggested proposals, current turn, and expiration timer.
     - `POST /api/v1/disputes/:id/mediation/proposals/:proposalId/respond`:
       - Body: `{"action": "accept" | "reject"}`
       - If caller is buyer, updates `buyer_accepted = true`.
       - If caller is seller, updates `seller_accepted = true`.
     - `POST /api/v1/disputes/:id/mediation/counter`:
       - Body: `{"buyer_amount_kobo": 15000000, "seller_amount_kobo": 35000000, "terms": "..."}`
     - `POST /api/v1/disputes/:id/mediation/escalate`:
       - Disputant opts out of mediation and demands immediate human arbitration.
  2. Emit real-time WebSocket events and system chat messages (`dispute_mediation_proposal_received`, `dispute_counter_offered`).

- **Task Validation Criteria**:
  - Unauthorized user (not buyer or seller) calling mediation endpoints receives `HTTP 403 Forbidden`.
  - When one party accepts, the response indicates `buyer_accepted: true, seller_accepted: false` and notifies the counterparty.

---

### Task 4.3: Mutual Agreement Execution & Auto-Settlement Wire

- **Goal**: Detect when both parties have accepted the same proposal and execute atomic settlement via the Bounded Enforcement Layer.
- **Files to Create**:
  - `internal/dispute/mediation/settler.go`
  - `internal/dispute/mediation/settler_test.go`
- **Implementation Details**:
  1. In `RespondToProposal`:
     - If both `buyer_accepted == true` AND `seller_accepted == true`:
       - Set `proposal.Status = "accepted"`.
       - Set `dispute.Status = DisputeStatusMediationResolved`.
       - Set `dispute.Resolution = ResolutionPartialSplit` (or FullRefund/FullRelease as applicable).
       - Construct `CanonicalSettlementIntent`:
         ```json
         {
           "intent_id": "...",
           "dispute_id": "...",
           "escrow_id": "...",
           "resolution_type": "partial_split",
           "buyer_amount_kobo": 15000000,
           "seller_amount_kobo": 35000000,
           "currency": "NGN",
           "evidence_merkle_root": "...",
           "timestamp": 1792348800,
           "nonce": 1
         }
         ```
       - Co-sign intent using System Settlement KMS Key (Ed25519) with proof type `system_automated`.
       - Create `DisputeExecutionProof` record with `is_fully_executed = false`.
       - Invoke `walletService.PartialSplitFunds(ctx, escrowID, buyerKobo, sellerKobo, proof)`.
       - Transition escrow from `dispute_held` $\rightarrow$ `partial_release` or `refunded`/`completed`.
       - Record trust score updates (positive trust event for reaching mutual agreement without human cost).

- **Task Validation Criteria**:
  - End-to-end unit test: Buyer accepts $\rightarrow$ Seller accepts $\rightarrow$ System co-signs execution proof $\rightarrow$ Wallet executes split $\rightarrow$ Escrow status changes to `partial_release`.
  - Verifies that zero funds are released if the double-entry balance check fails.

---

### Task 4.4: Frontend Interactive Mediation Room & Proposal Cards

- **Goal**: Create an engaging, high-trust mediation UI in `vescrow_frontend`.
- **Files to Create**:
  - `src/components/dispute/mediation-room.tsx` (in `vescrow_frontend`)
  - `src/components/dispute/proposal-card.tsx` (in `vescrow_frontend`)
  - `src/components/dispute/counter-offer-modal.tsx` (in `vescrow_frontend`)
- **Files to Modify**:
  - `src/app/app/escrows/[id]/page.tsx`
  - `src/lib/api/disputes.ts`
- **Implementation Details**:
  1. `proposal-card.tsx`:
     - Displays proposal terms, split percentages (e.g. 30% Buyer / 70% Seller), and monetary amounts in Naira.
     - Dual acceptance indicator badges:
       - Buyer Status: ⏳ Pending / ✅ Accepted
       - Seller Status: ⏳ Pending / ✅ Accepted
     - Primary button: "Accept This Resolution" (with confirmation modal).
     - Secondary button: "Propose Counter-Offer".
  2. `mediation-room.tsx`:
     - Visual Split Ratio Bar: Interactive horizontal stacked bar showing proposed fund division.
     - Live Countdown Banner: Displays hours and minutes remaining before automatic escalation.
     - "Request Human Neutral" link for disputants who wish to bypass mediation.
  3. Integration into `escrows/[id]/page.tsx`:
     - When escrow is `dispute_held` and dispute status is `in_mediation`, automatically render the `MediationRoom` at the top of the page.

- **Task Validation Criteria**:
  - Disputant can view suggested proposals, click "Accept", and see their checkmark activate in real time.
  - Submitting a counter-offer opens the modal, validates sum in real-time, and adds the new proposal card to the negotiation feed.

---

### Task 4.5: Mediation Countdown Timer & Auto-Escalation Worker

- **Goal**: Ensure that stalled negotiations do not lock funds indefinitely by automatically escalating expired mediation sessions.
- **Files to Create**:
  - `internal/dispute/mediation/timer.go`
  - `internal/dispute/mediation/timer_test.go`
- **Implementation Details**:
  1. Implement background cron job running every 15 minutes:
     ```go
     func (t *MediationTimerWorker) ProcessExpiredSessions(ctx context.Context) error {
         var expiredDisputes []dispute.Dispute
         now := time.Now()
         // Find disputes in mediation where SLA has expired
         db.Where("status = ? AND sla_deadline < ?", DisputeStatusInMediation, now).
            Find(&expiredDisputes)
         for _, d := range expiredDisputes {
             t.escalationRouter.EscalateDispute(ctx, d.ID, "mediation_timeout_expired")
         }
     }
     ```
  2. Send 24-hour and 6-hour warning email/SMS notifications to both parties reminding them of impending escalation.

- **Task Validation Criteria**:
  - Test verifying that setting a dispute's `sla_deadline` to 1 minute in the past and executing `ProcessExpiredSessions` shifts status to `escalated` and opens an escalation record.

---

## 3. Phase 4 Overall Validation & Exit Gate

The phase is completed when the following two end-to-end user journeys pass:

```
[TEST SUITE: Phase 4 Mediation Sandbox Gate]
Journey 1 (Successful Mutual Agreement):
1. Dispute #201 enters 'in_mediation' with ₦200,000 held.
2. System suggests Proposal A: ₦50k Buyer / ₦150k Seller.
3. Buyer counters: ₦80k Buyer / ₦120k Seller.
4. Seller logs in, reviews counter-offer, and clicks "Accept".
5. Backend verifies mutual acceptance -> System KMS signs execution proof.
6. Wallet balances update: Buyer +₦80k, Seller +₦120k.
7. Escrow status updates to 'partial_release' in < 2 seconds.

Journey 2 (Mediation Timeout Escalation):
1. Dispute #202 enters 'in_mediation'.
2. Neither party takes action.
3. Timer worker advances past 48h deadline.
4. Worker transitions dispute status to 'escalated' -> Dispatches to Arbiter Queue.
```
