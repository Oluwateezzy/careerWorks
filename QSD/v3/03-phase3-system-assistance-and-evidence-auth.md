# Phase 3 — System Assistance Layer & Evidence Authentication Agent

> **Document ID**: `VESC-P3-V3-005`  
> **Phase Target**: External Evidence Verification, Carrier Logistics Connectors, Legal Classification, and Structural Confidence Scoring ($C_s$)  
> **Subsystem**: `vescrow_backend` (Go/Fiber)  
> **Status**: Ready for Implementation  
> **Dependencies**: Phase 1 (Bounded Enforcement) & Phase 2 (Intake & Sanitization)

---

## 1. Executive Summary

Phase 3 implements the automated intelligence core of the Vescrow Dispute Resolution Engine. Once a dispute claim and its cryptographic evidence manifest are ingested in Phase 2, the **Evidence Authentication Agent** and **Legal Triage Engine** take over:
1. Cross-platform external connectors interface with logistics carrier APIs (GIG Logistics, DHL, FedEx) and digital code repositories (GitHub/GitLab) to extract objective ground truth.
2. The claims are cross-referenced against initial escrow parameters, delivery manifests, and historical trust profiles.
3. The engine computes a **Structural Confidence Score ($C_s \in [0.00, 1.00]$)** and assigns a **Risk Tier (`tier_1`, `tier_2`, `tier_3`)**.
4. The case is routed deterministically: high-confidence clear terms proceed toward automated settlement, moderate-ambiguity claims enter the AI mediation sandbox, and high-risk/complex disputes are dispatched to the human escalation queue.

---

## 2. Detailed Task Breakdown

### Task 3.1: Logistics Carrier Connector Framework & Adapters

- **Goal**: Build modular adapters for Nigerian and international couriers to query real-time tracking data and verify physical delivery status.
- **Files to Create**:
  - `internal/dispute/logistics/carrier.go`
  - `internal/dispute/logistics/gig.go`
  - `internal/dispute/logistics/dhl.go`
  - `internal/dispute/logistics/mock.go`
  - `internal/dispute/logistics/carrier_test.go`
- **Implementation Details**:
  1. Define the universal `CarrierAdapter` interface in `carrier.go`:
     ```go
     type TrackingDetails struct {
         CarrierCode        string
         TrackingNumber     string
         Status             CarrierDeliveryStatus // InTransit, Delivered, Returned, Failed, Unknown
         DeliveredAt        *time.Time
         RecipientName      string
         OriginCity         string
         DestinationCity    string
         WeightGrams        int
         ProofOfDeliveryURL string
         RawEvents          []TrackingEvent
     }
     type CarrierAdapter interface {
         FetchTracking(ctx context.Context, trackingNumber string) (*TrackingDetails, error)
         VerifyReturnDelivery(ctx context.Context, trackingNumber, sellerAddress string) (bool, error)
     }
     ```
  2. Implement `gig.go` (GIG Logistics API):
     - Authenticates via API key and merchant secret.
     - Parses tracking milestone codes (`ARRIVED_HUB`, `OUT_FOR_DELIVERY`, `DELIVERED_SIGNED`).
  3. Implement `dhl.go` (DHL Express API):
     - Handles OAuth2 client credentials and parses shipment status payloads.
  4. Implement `mock.go` for local testing and CI/CD pipelines, supporting pre-configured test tracking numbers (e.g. `TEST-DELIVERED-OK`, `TEST-RETURNED-SELLER`, `TEST-LOST-IN-TRANSIT`).

- **Task Validation Criteria**:
  - Unit tests verify that calling `FetchTracking("TEST-DELIVERED-OK")` returns `Status: Delivered` with accurate timestamps.
  - Network timeouts or API 500 errors from courier endpoints are handled gracefully with retries and exponential backoff, failing to `Status: Unknown` without crashing the application.

---

### Task 3.2: Digital Asset & Media Tamper Verification Agent

- **Goal**: Verify software milestones and detect forged or recycled evidence media.
- **Files to Create**:
  - `internal/dispute/agent/code_verifier.go`
  - `internal/dispute/agent/media_verifier.go`
  - `internal/dispute/agent/agent_test.go`
- **Implementation Details**:
  1. In `code_verifier.go`:
     - Interfaces with GitHub API via personal access tokens to inspect target repository.
     - Verifies:
       - Did the seller commit code prior to the milestone deadline?
       - Was the agreed pull request merged into the default branch?
       - Does the repository commit SHA match the deliverable URL submitted by the seller?
     - Performs automated HTTP health check on staging URLs to verify application responsiveness (HTTP 200 OK vs 500 error).
  2. In `media_verifier.go`:
     - Compares uploaded image SHA-256 against historical dispute evidence in `dispute_evidences` table to detect recycled stock photos or repeat scam uploads.
     - Validates EXIF timestamp: checks if the photo creation timestamp occurred **after** the package dispatch date and **before** the dispute opening date.
     - Logs verification record to `dispute_external_verifications` table.

- **Task Validation Criteria**:
  - Unit test verifies that a duplicate evidence hash from an unrelated older dispute is flagged with `duplicate_evidence_detected`.
  - Staging URL validator correctly flags an unreachable server (`dial tcp: connection refused`) as an unverified digital deliverable.

---

### Task 3.3: Legal Triage & Parameter Classification Engine

- **Goal**: Cross-reference claims against contract parameters and identify objective contradictions or anomalies.
- **Files to Create**:
  - `internal/dispute/triage/classifier.go`
  - `internal/dispute/triage/classifier_test.go`
- **Implementation Details**:
  1. Define classification rules:
     - **Rule 1 (False Non-Delivery Claim)**: Buyer claims `item_not_received`, but carrier tracking shows signed delivery to buyer's registered city with timestamp $> 24\text{h}$ prior.
     - **Rule 2 (Confirmed Non-Shipment)**: Buyer claims `item_not_received`, and seller tracking code is invalid or shows carrier has not received the package after 5 business days.
     - **Rule 3 (Carrier Confirmed Return)**: In a dispute where return was mandated, carrier API confirms return package was delivered back to seller.
     - **Rule 4 (Subjective Quality Ambiguity)**: Category is `not_as_described` for custom artisan, clothing, or creative work without objective defect proof.
  2. Map dispute category to risk classifications:
     - Physical goods with valid carrier tracking $\rightarrow$ High Structure.
     - Digital milestones with verified git commit $\rightarrow$ High Structure.
     - Subjective aesthetic claims $\rightarrow$ High Ambiguity.

- **Task Validation Criteria**:
  - Rule evaluation tests confirm that a confirmed carrier delivery triggers `Rule 1 (False Non-Delivery Claim)` anomaly flag.
  - An escrow with a non-existent tracking number is correctly classified as `Rule 2 (Confirmed Non-Shipment)`.

---

### Task 3.4: Structural Confidence Scoring Engine ($C_s$)

- **Goal**: Implement the deterministic mathematical scoring model that gauges dispute clarity.
- **Files to Create**:
  - `internal/dispute/triage/confidence.go`
  - `internal/dispute/triage/confidence_test.go`
- **Implementation Details**:
  1. Implement the weighted scoring formula:
     $$C_s = w_1 \cdot V_{\text{logistics}} + w_2 \cdot V_{\text{history}} + w_3 \cdot V_{\text{evidence}} + w_4 \cdot V_{\text{consistency}}$$
     With default weights: $w_1 = 0.35$, $w_2 = 0.20$, $w_3 = 0.25$, $w_4 = 0.20$ ($\sum w_i = 1.0$).
  2. Component Vector Calculators:
     - **$V_{\text{logistics}}$**:
       - 1.0: Carrier API officially confirms delivery or return with signed proof.
       - 0.0: Carrier confirms parcel lost or tracking invalid.
       - 0.5: Courier API unavailable or untracked local dispatch.
     - **$V_{\text{history}}$**:
       - Calculated from disputants' platform trust scores ($S_{\text{buyer}}, S_{\text{seller}} \in [0, 100]$):
         $$V_{\text{history}} = \frac{S_{\text{buyer}} + S_{\text{seller}}}{200}$$
       - Heavily penalized if either party has recent dispute losses.
     - **$V_{\text{evidence}}$**:
       - Ratio of verified evidence items (valid unboxing video, serial match, invoice) to expected evidence schema.
     - **$V_{\text{consistency}}$**:
       - Timeliness score based on how quickly the dispute was filed after shipment and consistency of logged timestamps.
  3. Risk Tier & Routing Assignment:
     - If $C_s \ge 0.90$ AND transaction amount $< \text{₦}1,000,000$ $\rightarrow$ Route to Automated Settlement Engine.
     - If $0.60 \le C_s < 0.90$ AND no fraud flag $\rightarrow$ Route to AI-Assisted Mediation Sandbox (`in_mediation`).
     - If $C_s < 0.60$ OR transaction amount $\ge \text{₦}1,000,000$ OR fraud flag $\rightarrow$ Route to Escalation Router (`escalated`).

- **Task Validation Criteria**:
  - Test vectors covering 15 synthetic dispute scenarios produce exact expected $C_s$ scores within $\pm 0.0001$.
  - Cases above ₦1,000,000 are guaranteed to be tagged `tier_2` or `tier_3` regardless of high $C_s$.

---

### Task 3.5: Asynchronous Triage Worker Pool & Routing Coordinator

- **Goal**: Execute the verification, classification, and scoring pipeline asynchronously upon dispute filing.
- **Files to Create**:
  - `internal/dispute/triage/worker.go`
- **Files to Modify**:
  - `internal/dispute/service.go`
- **Implementation Details**:
  1. Build a concurrent worker pool (`TriageWorker`) listening to dispute creation events via internal Go channel or Redis queue.
  2. Execution Lifecycle per Dispute:
     - Worker picks up dispute ID in `triage_in_progress` state.
     - Invokes Carrier Adapter (Task 3.1) and Digital Verifiers (Task 3.2).
     - Runs Anti-Bias Sanitization (from Phase 2).
     - Executes Classification Engine (Task 3.3).
     - Computes $C_s$ and Risk Tier (Task 3.4).
     - Persists results to DB: updates `confidence_score`, `triage_category`, `risk_tier`.
     - Transitions dispute status:
       - If automated rule triggered: $\rightarrow$ invokes Automated Settlement Engine (Phase 6).
       - If mediation eligible: $\rightarrow$ status becomes `in_mediation` (Phase 4).
       - If escalation required: $\rightarrow$ status becomes `escalated` (Phase 5).
     - Emits system activity event to the escrow chat log and sends email notifications.

- **Task Validation Criteria**:
  - Filing a dispute triggers the worker and completes full triage in $< 3$ seconds in test environment.
  - Database reflects populated `confidence_score`, `risk_tier`, and correct destination status.

---

## 3. Phase 3 Overall Validation & Exit Gate

The phase passes when all three routing pathways are demonstrated via automated tests:

```
[TEST SUITE: Phase 3 System Assistance & Triage Gate]
Scenario A (High Confidence Carrier Return -> Auto Route):
- Tracking confirms package returned to seller.
- Cs computed: 0.94 -> Escrow routes directly to Automated Settlement candidate.

Scenario B (Moderate Ambiguity Damaged Item -> Mediation Route):
- Photo evidence uploaded, tracking confirmed, minor defect claimed.
- Cs computed: 0.72 -> Dispute status shifts to 'in_mediation' with 48h deadline.

Scenario C (High Value / Subjective Creative Failure -> Escalation Route):
- Transaction amount: ₦3,500,000. Creative design milestone failure.
- Cs computed: 0.52 -> Tagged 'tier_3', status shifts to 'escalated', SLA set to 6 hours.
```
