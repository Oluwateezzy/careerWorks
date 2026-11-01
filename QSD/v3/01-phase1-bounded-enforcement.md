# Phase 1 — Bounded Enforcement & State Locking Engine

> **Document ID**: `VESC-P1-V3-003`  
> **Phase Target**: Core Cryptographic Foundations, Immutable Escrow State Locking, and Gated Ledger Rails  
> **Subsystem**: `vescrow_backend` (Go/Fiber/GORM)  
> **Status**: Ready for Implementation  
> **Dependencies**: None (Foundational Phase)

---

## 1. Executive Summary

Phase 1 constructs the trustless backbone of the Vescrow V3 Dispute Engine. Under this phase:
1. The escrow state machine is modified to enforce an immediate, immutable **"Held" Lock** (`EscrowStatusDisputeHeld`) the exact millisecond a dispute is filed. This freezes all funds and prohibits any standard release, withdrawal, cancellation, or timer-based auto-release.
2. A **Bounded Enforcement Engine** is introduced. Neither internal administrators nor automated background workers can disburse funds via simple database updates. All monetary movements require a cryptographically valid **`ExecutionProof`** containing Ed25519 or ECDSA digital signatures matching either the platform's System Settlement KMS key or an M-of-N quorum of authorized human arbiters.
3. The double-entry bookkeeping ledger (`internal/wallet/service.go`) is refactored to require proof verification before executing debit/credit ledger records.

---

## 2. Detailed Task Breakdown

### Task 1.1: Database Schema Migrations & GORM Models

- **Goal**: Provision the database tables and model structures for cryptographic arbiter keys, execution proofs, and dispute state extensions.
- **Files to Create**:
  - `internal/dispute/enforcement/model.go`
- **Files to Modify**:
  - `internal/escrow/enum.go`
  - `internal/dispute/enum.go`
  - `internal/dispute/model.go`
- **Implementation Details**:
  1. Add `EscrowStatusDisputeHeld EscrowStatus = "dispute_held"` in `internal/escrow/enum.go`.
  2. Add new dispute status constants in `internal/dispute/enum.go`:
     - `DisputeStatusTriageInProgress DisputeStatus = "triage_in_progress"`
     - `DisputeStatusInMediation DisputeStatus = "in_mediation"`
     - `DisputeStatusMediationResolved DisputeStatus = "mediation_resolved"`
     - `DisputeStatusEscalated DisputeStatus = "escalated"`
     - `DisputeStatusProofPending DisputeStatus = "proof_pending"`
  3. Define in `internal/dispute/enforcement/model.go`:
     - `ArbiterSigningKey`: Stores public keys, algorithm (`Ed25519` / `ECDSA_P256`), authorized risk tier (`tier_1`, `tier_2`, `tier_3`), and revocation status.
     - `DisputeExecutionProof`: Stores canonical intent JSON, SHA-256 intent hash, proof type (`system_automated` vs `human_multisig`), required vs collected signatures, JSONB array of signatures, and execution timestamp.
     - `CanonicalSettlementIntent`: Strongly typed Go struct for deterministic JSON serialization.
  4. Extend `Dispute` model in `internal/dispute/model.go` with:
     - `ConfidenceScore float64` (`gorm:"type:numeric(5,4);default:0.0000"`)
     - `TriageCategory string` (`gorm:"type:varchar(50)"`)
     - `RiskTier string` (`gorm:"type:varchar(20);default:'tier_1'"`)
     - `MediationStatus string` (`gorm:"type:varchar(30);default:'not_started'"`)
     - `EvidenceMerkleRoot string` (`gorm:"type:varchar(64)"`)
     - `ExecutionProofID *uuid.UUID` (`gorm:"type:uuid"` foreign key to `dispute_execution_proofs`)
     - `AutoResolved bool` (`gorm:"default:false"`)
     - `SLADeadline *time.Time`
     - `EscalatedAt *time.Time`
  5. Add AutoMigrate entries in `internal/database/database.go`.

- **Task Validation Criteria**:
  - Executing `go test ./internal/dispute/...` runs AutoMigrate cleanly on PostgreSQL without schema syntax errors.
  - Foreign key constraints between `disputes`, `dispute_execution_proofs`, and `escrows` enforce referential integrity.
  - Unique index on `arbiter_signing_keys.key_fingerprint` and `dispute_execution_proofs.intent_hash` blocks duplicates.

---

### Task 1.2: Escrow State Machine Hardening to `dispute_held`

- **Goal**: Guarantee that an escrow under dispute cannot have funds released, cancelled, or modified by any standard workflow.
- **Files to Modify**:
  - `internal/escrow/state_machine.go`
  - `internal/escrow/service.go`
  - `internal/escrow/worker.go`
- **Implementation Details**:
  1. In `internal/escrow/state_machine.go`:
     - Register `EscrowStatusDisputeHeld` as a valid transition target from `EscrowStatusShipped`, `EscrowStatusDeliveredAwaitingVerification`, `EscrowStatusDeliveryVerified`, and `EscrowStatusInProgress`.
     - Explicitly define outgoing transitions from `EscrowStatusDisputeHeld`: only allowed destinations are `EscrowStatusRefunded`, `EscrowStatusCompleted`, and `EscrowStatusPartialRelease`.
     - Block all user transitions (e.g. `VerifyDelivery`, `CancelEscrow`, `ApproveDelivery`) when current status is `EscrowStatusDisputeHeld`, returning `common.NewAppError(409, "Escrow is held under active dispute")`.
  2. In `internal/escrow/worker.go`:
     - Update inspection period auto-release query: `db.Where("status = ? AND inspection_period_ends_at < ?", EscrowStatusShipped, now)`. Ensure it explicitly excludes `status = 'dispute_held'`.
  3. In `internal/escrow/service.go`:
     - Add `HoldForDispute(ctx context.Context, tx *gorm.DB, escrowID uuid.UUID) error` helper executed within the dispute creation transaction.

- **Task Validation Criteria**:
  - Unit test in `state_machine_test.go` verifies that attempting to call `VerifyDelivery()` or `CancelEscrow()` on an escrow in `dispute_held` returns `HTTP 409 Conflict`.
  - Worker unit test confirms that an expired inspection timer does **not** auto-release funds if the escrow is in `dispute_held`.

---

### Task 1.3: Canonical Intent Serializer & Deterministic Hasher

- **Goal**: Implement deterministic, canonical hashing of settlement intents to ensure identical hash representation across Go backend and browser WebCrypto clients.
- **Files to Create**:
  - `internal/dispute/enforcement/intent.go`
  - `internal/dispute/enforcement/intent_test.go`
- **Implementation Details**:
  1. Define canonical settlement intent struct:
     ```go
     type CanonicalSettlementIntent struct {
         IntentID            uuid.UUID `json:"intent_id"`
         DisputeID           uuid.UUID `json:"dispute_id"`
         EscrowID            uuid.UUID `json:"escrow_id"`
         ResolutionType      string    `json:"resolution_type"` // "full_refund", "full_release", "partial_split"
         BuyerAmountKobo     int64     `json:"buyer_amount_kobo"`
         SellerAmountKobo    int64     `json:"seller_amount_kobo"`
         Currency            string    `json:"currency"`
         EvidenceMerkleRoot  string    `json:"evidence_merkle_root"`
         Timestamp           int64     `json:"timestamp"`
         Nonce               int64     `json:"nonce"`
     }
     ```
  2. Implement `Canonicalize() ([]byte, error)`:
     - Sorts JSON dictionary keys alphabetically.
     - Strips unescaped whitespace and newline characters.
     - Formats numbers as standard decimal integers without exponent notation.
  3. Implement `ComputeIntentHash(intent *CanonicalSettlementIntent) (string, error)`:
     - Returns lowercase hexadecimal SHA-256 hash of the canonical bytes.

- **Task Validation Criteria**:
  - Table-driven unit test in `intent_test.go` tests 10 varied payloads with scrambled key orders and verifies that `ComputeIntentHash()` consistently produces the identical SHA-256 hash.
  - Cross-language test vector: Verifies that Go `ComputeIntentHash` output matches the output of JavaScript `crypto.subtle.digest('SHA-256', ...)` on identical test vectors.

---

### Task 1.4: Cryptographic Execution Proof Verifier

- **Goal**: Build the cryptographic signature verification engine for Ed25519 and ECDSA P-256 signatures.
- **Files to Create**:
  - `internal/dispute/enforcement/verifier.go`
  - `internal/dispute/enforcement/verifier_test.go`
- **Implementation Details**:
  1. Implement `ProofVerifier` struct with dependencies on DB and system KMS config.
  2. Implement `VerifyExecutionProof(ctx context.Context, proof *DisputeExecutionProof) error`:
     - Recalculates `ComputeIntentHash` over `proof.CanonicalIntent` and asserts equality with `proof.IntentHash`.
     - Validates that `BuyerAmountKobo + SellerAmountKobo == Escrow.AmountKobo`.
     - Branch on `ProofType`:
       - **`system_automated`**:
         - Requires exactly 1 signature matching `config.SystemSettlementPubKey`.
         - Verifies Ed25519 signature over `intent_hash`.
       - **`human_multisig`**:
         - Checks `Dispute.RiskTier`:
           - `tier_1`: Requires at least 1 valid signature from an active Tier 1+ arbiter.
           - `tier_2`: Requires at least 2 distinct valid signatures from active Tier 2+ arbiters.
           - `tier_3`: Requires at least 2 distinct valid signatures from Tier 3 arbiters, including at least one designated compliance key.
         - For each signature:
           - Fetches registered public key from `arbiter_signing_keys` by `key_fingerprint`.
           - Asserts `is_active == true` and `revoked_at IS NULL`.
           - Verifies cryptographic signature using `crypto/ed25519` or `crypto/ecdsa`.
           - Ensures no duplicate signatures from the same arbiter.
  3. Mark proof as verified and return nil, or reject with descriptive validation error.

- **Task Validation Criteria**:
  - Unit tests verify:
    - Valid Ed25519 system signature passes verification.
    - Valid 2-of-3 human arbiter signatures pass verification for Tier 2.
    - Reused signature from the same arbiter is rejected.
    - Revoked arbiter key signature is rejected with `ErrKeyRevoked`.
    - Corrupted or tampered intent hash fails signature check with `ErrInvalidSignature`.

---

### Task 1.5: Double-Entry Ledger Gating & Safe Fund Execution

- **Goal**: Rewire wallet settlement so that no funds can move unless accompanied by a verified `ExecutionProof`.
- **Files to Modify**:
  - `internal/wallet/service.go`
  - `internal/dispute/service.go`
- **Implementation Details**:
  1. In `internal/wallet/service.go`:
     - Refactor `ReleaseToSeller`, `RefundToBuyer`, and `PartialSplitFunds` to accept an additional parameter: `proof *enforcement.DisputeExecutionProof`.
     - In each method, call `s.proofVerifier.VerifyExecutionProof(ctx, proof)`. If verification fails, immediately abort the transaction.
     - Record the `proof.ID` in the ledger transaction reference column for audit reconciliation.
  2. In `internal/dispute/service.go`:
     - Update `ResolveDispute` to construct an `ExecutionProof`, invoke the verifier, and pass the verified proof to the wallet service.
     - Transition escrow status from `dispute_held` to final status (`refunded`, `completed`, or `partial_release`) within the same database transaction.
     - Mark `DisputeExecutionProof.IsFullyExecuted = true` and record `ExecutedAt = time.Now()`.

- **Task Validation Criteria**:
  - Direct call to `walletService.ReleaseToSeller(ctx, escrowID, nil)` fails with `ErrMissingExecutionProof`.
  - A validly signed proof executes atomic ledger transfers with exact zero-sum ledger balance.
  - Escrow status cleanly transitions from `dispute_held` $\rightarrow$ `refunded` or `completed`.

---

## 3. Phase 1 Overall Validation & Exit Gate

Before Phase 1 is marked complete and Phase 2 begins, the following comprehensive integration test must pass:

```
[TEST SUITE: Phase 1 Bounded Enforcement Gate]
1. Create and fund test escrow (Amount: ₦500,000 / 50,000,000 kobo).
2. Open dispute -> Verify escrow status shifts to 'dispute_held'.
3. Attempt concurrent buyer approval -> Verify HTTP 409 Conflict.
4. Attempt seller cancellation -> Verify HTTP 409 Conflict.
5. Attempt manual wallet debit without proof -> Verify rejected.
6. Create canonical intent for 70/30 split (₦350k buyer / ₦150k seller).
7. Sign with 1 arbiter key on Tier 2 case -> Verify verifier rejects (requires 2 signatures).
8. Add second arbiter key signature -> Verify verifier accepts.
9. Execute settlement -> Verify:
   - System escrow wallet debited: 50,000,000 kobo
   - Buyer wallet credited: 35,000,000 kobo
   - Seller wallet credited: 15,000,000 kobo (less fee)
   - Escrow status = 'partial_release'
   - Proof marked 'is_fully_executed' = true
```
