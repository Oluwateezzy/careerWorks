# Phase 1 — Escrow Flow Redesign: Fund-on-Create & State Machine Simplification

> **Goal**: Buyer funds escrow at creation time to show intentionality. Remove the redundant "Confirm Package Delivery Received" state. Merge "Verify Product Condition" with "Approve & Release Funds" into a single atomic action. Lock completed escrows to read-only.

---

## Task 1.1 — Backend: Simplify State Machine

### Description
Update the escrow state machine to remove the `delivered_awaiting_verification` state. The `shipped` state now transitions directly to `delivery_verified`, which then transitions to `completed`.

### Files to Modify
- [`internal/escrow/state_machine.go`](file:///Users/oluwatobiloba/Desktop/qsd/vescrow/vescrow_backend/internal/escrow/state_machine.go)
- [`internal/escrow/enum.go`](file:///Users/oluwatobiloba/Desktop/qsd/vescrow/vescrow_backend/internal/escrow/enum.go)

### Implementation Details

**`state_machine.go`** — Update `ValidTransitions`:
```diff
 EscrowStatusShipped: {
-    EscrowStatusDeliveredAwaitingVerification,
+    EscrowStatusDeliveryVerified,
     EscrowStatusDisputed,
 },
-EscrowStatusDeliveredAwaitingVerification: {
-    EscrowStatusDeliveryVerified,
-    EscrowStatusDisputed,
-},
 EscrowStatusDeliveryVerified: {
     EscrowStatusCompleted,
     EscrowStatusDisputed,
 },
```

**`enum.go`** — Keep `EscrowStatusDeliveredAwaitingVerification` for backward compatibility with existing DB records, but add a deprecation comment.

### Validation Criteria
- [ ] `CanTransition("shipped", "delivery_verified")` returns `true`
- [ ] `CanTransition("shipped", "delivered_awaiting_verification")` returns `false`
- [ ] Existing tests in `state_machine_test.go` updated and passing
- [ ] Existing escrows in `delivered_awaiting_verification` state can still transition to `delivery_verified` (migration safety)

---

## Task 1.2 — Backend: Remove `ConfirmDelivery` Endpoint

### Description
Remove the `ConfirmDelivery` service method and HTTP handler. This endpoint previously transitioned from `shipped` → `delivered_awaiting_verification`, which no longer exists.

### Files to Modify
- [`internal/escrow/service.go`](file:///Users/oluwatobiloba/Desktop/qsd/vescrow/vescrow_backend/internal/escrow/service.go) — Delete `ConfirmDelivery` function (line ~648)
- [`internal/escrow/handler.go`](file:///Users/oluwatobiloba/Desktop/qsd/vescrow/vescrow_backend/internal/escrow/handler.go) — Delete `ConfirmDelivery` handler (line ~538)
- [`internal/escrow/routes.go`](file:///Users/oluwatobiloba/Desktop/qsd/vescrow/vescrow_backend/internal/escrow/routes.go) — Remove the route for confirm-delivery
- [`internal/escrow/dto.go`](file:///Users/oluwatobiloba/Desktop/qsd/vescrow/vescrow_backend/internal/escrow/dto.go) — Remove `ConfirmDeliveryDTO`

### Validation Criteria
- [ ] `POST /api/escrows/:id/confirm-delivery` returns 404
- [ ] No compilation errors
- [ ] All remaining tests pass

---

## Task 1.3 — Backend: Merge Verify + Approve + Release into Single Action

### Description
The `VerifyDelivery` service method now:
1. Validates buyer's inspection evidence (images, video, notes)
2. **Forces** `condition_accepted = true` (no longer optional)
3. Transitions escrow from `shipped` → `delivery_verified`
4. **Immediately** calls the fund release logic (same as current `ApproveDelivery`)
5. Transitions from `delivery_verified` → `completed`
6. Records trust event for successful completion

### Files to Modify
- [`internal/escrow/service.go`](file:///Users/oluwatobiloba/Desktop/qsd/vescrow/vescrow_backend/internal/escrow/service.go) — Modify `VerifyDelivery` (line ~713)

### Implementation Details
```go
func (s *EscrowService) VerifyDelivery(ctx context.Context, id uuid.UUID, userID uuid.UUID, dto VerifyDeliveryDTO) (*Escrow, error) {
    // ... existing validation ...
    
    // Force condition_accepted = true
    dto.ConditionAccepted = true
    
    // Transition: shipped → delivery_verified
    // ... existing verify logic ...
    
    // IMMEDIATELY release funds (merged from ApproveDelivery)
    // Transition: delivery_verified → completed
    if err := s.releaseFundsToSeller(ctx, escrow); err != nil {
        return nil, err
    }
    escrow.Status = EscrowStatusCompleted
    escrow.CompletedAt = timePtr(time.Now())
    
    // Record trust events
    s.trustSvc.RecordEvent(ctx, escrow.BuyerID, trust.EventCompleted, &escrow.ID, "Buyer approved delivery")
    s.trustSvc.RecordEvent(ctx, *escrow.SellerID, trust.EventCompleted, &escrow.ID, "Transaction completed")
    
    return escrow, nil
}
```

### Validation Criteria
- [ ] Calling `VerifyDelivery` on a `shipped` escrow transitions it directly to `completed`
- [ ] Funds are released to seller's wallet in the same transaction
- [ ] Trust events recorded for both buyer and seller
- [ ] `ApproveDelivery` endpoint still works for backward compatibility with existing `delivery_verified` escrows
- [ ] System message emitted: "Buyer verified delivery and released funds"

---

## Task 1.4 — Backend: Fund-on-Create Flow

### Description
When a buyer creates an escrow, the system should immediately initiate payment. The backend `CreateEscrow` endpoint returns a `payment_url` in the response that the frontend redirects to.

### Files to Modify
- [`internal/escrow/service.go`](file:///Users/oluwatobiloba/Desktop/qsd/vescrow/vescrow_backend/internal/escrow/service.go) — Modify `CreateEscrow`
- [`internal/escrow/dto.go`](file:///Users/oluwatobiloba/Desktop/qsd/vescrow/vescrow_backend/internal/escrow/dto.go) — Add `CallbackURL` to `CreateEscrowDTO`, add `PaymentURL` to `CreateEscrowResult`

### Implementation Details
The create flow becomes:
1. Validate inputs and create escrow record (status: `pending_acceptance`)
2. Initialize Paystack payment for the buyer total
3. Return `{ escrow, invite_url, payment_url }` to frontend
4. Frontend redirects buyer to `payment_url`
5. On payment callback success, the existing payment webhook handler funds the escrow

The escrow is created in `pending_acceptance` immediately. Payment is initialized but not yet confirmed. When payment succeeds (Paystack callback/webhook), the escrow's funding is recorded. The seller still needs to accept, but the money is already locked — showing buyer intentionality.

### New State Transition Rule
```
pending_acceptance (funded) → seller accepts → funded → preparing_shipment ...
```

The `funded_at` timestamp is set when payment succeeds. The `accepted` status is now optional — if the buyer has already funded and seller accepts, it skips directly to `funded`.

### Validation Criteria
- [ ] `POST /api/escrows` response includes `payment_url` field
- [ ] Paystack payment is initialized with correct amount (buyer_total_kobo)
- [ ] Payment webhook correctly marks escrow as funded
- [ ] Seller can still accept/reject a funded escrow
- [ ] If seller rejects a funded escrow, refund is triggered
- [ ] Create escrow without payment (fallback) still works for wallet-funded escrows

---

## Task 1.5 — Frontend: Update Create Escrow Page for Fund-on-Create

### Description
Modify the create escrow page to redirect the buyer to the payment gateway immediately after successful creation.

### Files to Modify
- [`src/app/app/escrows/create/page.tsx`](file:///Users/oluwatobiloba/Desktop/qsd/vescrow/vescrow_frontend/src/app/app/escrows/create/page.tsx) — Modify `handleSubmit`
- [`src/stores/escrow-store.ts`](file:///Users/oluwatobiloba/Desktop/qsd/vescrow/vescrow_frontend/src/stores/escrow-store.ts) — Modify `createEscrow` to handle payment_url
- [`src/lib/types.ts`](file:///Users/oluwatobiloba/Desktop/qsd/vescrow/vescrow_frontend/src/lib/types.ts) — Add `payment_url` to `CreateEscrowResult`

### Implementation Details
```typescript
// types.ts — Update CreateEscrowResult
export interface CreateEscrowResult {
  escrow: Escrow;
  invite_url: string;
  payment_url?: string;  // NEW: Paystack authorization URL
  seller_pending_registration: boolean;
}

// create/page.tsx — handleSubmit modification
const result = await createEscrow({ ...data, callback_url: `${window.location.origin}/app/payments/callback` });
setCreateResult(result);

// If payment_url exists, redirect immediately
if (result.payment_url && typeof window !== 'undefined') {
  window.location.href = result.payment_url;
  return;
}
// Otherwise show the existing confirmation screen
toast.success('Escrow created! Payment will be processed.');
```

### Validation Criteria
- [ ] After clicking "Confirm & Create Deal", buyer is redirected to Paystack
- [ ] After successful payment, buyer lands on payment callback page
- [ ] Payment callback page redirects to escrow detail page
- [ ] If payment fails, user can retry from escrow detail page
- [ ] Invite is sent to seller immediately (before payment completes)

---

## Task 1.6 — Frontend: Remove Confirm Delivery Modal & Merge Verify + Approve

### Description
Remove the "Confirm Package Delivery Received" modal entirely. Update the "Verify Product Condition & Inspection" modal to force approval and release funds directly.

### Files to Modify
- [`src/app/app/escrows/[id]/page.tsx`](file:///Users/oluwatobiloba/Desktop/qsd/vescrow/vescrow_frontend/src/app/app/escrows/%5Bid%5D/page.tsx)
- [`src/stores/escrow-store.ts`](file:///Users/oluwatobiloba/Desktop/qsd/vescrow/vescrow_frontend/src/stores/escrow-store.ts)
- [`src/lib/types.ts`](file:///Users/oluwatobiloba/Desktop/qsd/vescrow/vescrow_frontend/src/lib/types.ts)
- [`src/lib/api/escrows.ts`](file:///Users/oluwatobiloba/Desktop/qsd/vescrow/vescrow_frontend/src/lib/api/escrows.ts)

### Implementation Details

**Remove from escrow detail page:**
- Delete `confirmDeliveryModalOpen` state and all associated state variables (lines 97-99)
- Delete Modal 3: "Confirm Package Delivery Received" (lines 865-917)
- Remove the `shipped` status action buttons that show "Confirm Delivery Received" 
- Change `shipped` status to show "Verify & Release Funds" button instead

**Update Verify Modal:**
- Remove the `conditionAccepted` checkbox — it's always true
- Change submit button text from "Mark Delivery Verified" to "Approve & Release Funds"
- Add a clear confirmation message: "By submitting, you confirm the product matches agreed specifications and authorize immediate release of funds to the seller."
- On success, show toast: "Product verified! Funds released to seller."

**Update Desktop Action Bar:**
- `shipped` status: Show "Verify & Release Funds" (opens verify modal)
- Remove `delivered_awaiting_verification` action section entirely
- Remove `delivered` / `delivery_verified` approve button section entirely

**Update Mobile Action Bar:**
- Same changes as desktop

**Remove from types/store:**
- Delete `ConfirmDeliveryRequest` interface
- Remove `confirmDelivery` from `EscrowState` interface and store
- Remove `confirmDelivery` API call from `escrows.ts`

### Validation Criteria
- [ ] No "Confirm Delivery Received" button/modal appears anywhere
- [ ] When escrow is `shipped`, buyer sees only "Verify & Release Funds" and "Open Dispute"
- [ ] Clicking "Verify & Release Funds" opens the inspection modal
- [ ] Submitting the inspection modal transitions escrow to `completed`
- [ ] No `conditionAccepted` checkbox in the verify modal
- [ ] Toast shows "Product verified! Funds released to seller."

---

## Task 1.7 — Frontend: Completed Transaction Read-Only Summary

### Description
When an escrow reaches `completed`, `refunded`, or `cancelled` status, the detail page shows a read-only transaction summary with no action buttons.

### Files to Modify
- [`src/app/app/escrows/[id]/page.tsx`](file:///Users/oluwatobiloba/Desktop/qsd/vescrow/vescrow_frontend/src/app/app/escrows/%5Bid%5D/page.tsx)

### Implementation Details
Add a "Transaction Complete" summary card at the top of the detail page for terminal states:

```tsx
const TERMINAL_STATES = ['completed', 'refunded', 'cancelled', 'cancelled_overcapacity', 'expired', 'rejected'];
const isTerminal = TERMINAL_STATES.includes(currentEscrow.status);

{isTerminal && (
  <Card variant="elevated" padding="lg" className="...">
    <div className="flex items-center gap-3">
      <CheckCircle2 className="w-8 h-8 text-emerald-400" />
      <div>
        <h2 className="text-xl font-bold">Transaction {currentEscrow.status}</h2>
        <p className="text-sm text-slate-400">
          This escrow has been finalized. No further actions can be taken.
        </p>
      </div>
    </div>
    {/* Summary: amount, parties, dates, final status */}
  </Card>
)}
```

Ensure `mobileActions` returns `null` for terminal states (it already does implicitly, but verify).

### Validation Criteria
- [ ] Completed escrows show a summary banner
- [ ] No action buttons appear on completed escrows
- [ ] Summary shows: title, amount, parties, funded date, completed date, status
- [ ] Chat is still viewable but not sendable for completed escrows
- [ ] Dispute tab shows resolution if applicable

---

## Phase 1 Validation Criteria

- [ ] The complete new escrow lifecycle works end-to-end: Create+Fund → Seller Accepts → Prepare → Ship → Buyer Verify+Release → Completed
- [ ] Old escrows in `delivered_awaiting_verification` can still be progressed (backward compat)
- [ ] All backend tests pass with updated state machine
- [ ] No TypeScript compilation errors
- [ ] Payment flow (Paystack redirect → callback → funded) works correctly
- [ ] Completed escrows are fully locked to read-only
