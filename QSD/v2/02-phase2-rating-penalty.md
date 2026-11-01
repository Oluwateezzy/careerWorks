# Phase 2 — Rating Penalty for Non-Approval & Trust System Enhancement

> **Goal**: If a buyer fails to approve and release funds after the inspection period expires, their trust score takes a negative hit. This leverages the existing trust/rating system.

---

## Task 2.1 — Backend: Add New Trust Event Type for Buyer Verification Timeout

### Description
Add a new event type `EventBuyerVerificationTimeout` that is recorded when the auto-release worker releases funds because the buyer didn't verify within the inspection period.

### Files to Modify
- [`internal/trust/enum.go`](file:///Users/oluwatobiloba/Desktop/qsd/vescrow/vescrow_backend/internal/trust/enum.go)
- [`internal/trust/service.go`](file:///Users/oluwatobiloba/Desktop/qsd/vescrow/vescrow_backend/internal/trust/service.go)

### Implementation Details

**`enum.go`** — Add new event type:
```diff
 const (
     EventCompleted        EventType = "completed"
     EventDeliveryOnTime   EventType = "delivery_on_time"
     EventDisputeLost      EventType = "dispute_lost"
     EventDeliveryDefault  EventType = "delivery_default"
     EventChargeback       EventType = "chargeback"
     EventAdminAdjust      EventType = "admin_adjust"
+    EventBuyerVerificationTimeout EventType = "buyer_verification_timeout"
 )
```

**`service.go`** — Add delta for the new event:
```diff
 var eventDeltas = map[EventType]int{
     EventCompleted:       2,
     EventDeliveryOnTime:  1,
     EventDisputeLost:     -15,
     EventDeliveryDefault: -20,
     EventChargeback:      -30,
+    EventBuyerVerificationTimeout: -10,
 }
```

The `-10` penalty is moderate — less severe than a lost dispute (`-15`) or delivery default (`-20`), but significant enough to incentivize timely verification. Repeated timeouts will compound.

### Validation Criteria
- [ ] `EventBuyerVerificationTimeout` is defined in the enum
- [ ] Delta of `-10` is assigned in `eventDeltas`
- [ ] `RecordEvent` correctly applies the delta when called with this event type
- [ ] Trust score calculation: a buyer starting at 100 drops to 90 after one timeout

---

## Task 2.2 — Backend: Auto-Release Worker Records Trust Penalty

### Description
Modify the auto-release worker (`ProcessAutoReleases`) to record a `EventBuyerVerificationTimeout` trust event against the buyer when funds are auto-released.

### Files to Modify
- [`internal/escrow/worker.go`](file:///Users/oluwatobiloba/Desktop/qsd/vescrow/vescrow_backend/internal/escrow/worker.go)

### Implementation Details

In the `autoReleaseEscrow` function (line ~61), after successfully releasing funds and transitioning to `completed`, add:

```go
func (s *EscrowService) autoReleaseEscrow(ctx context.Context, escrow Escrow) error {
    // ... existing release logic ...
    
    // NEW: Record trust penalty for buyer not verifying
    if err := s.trustSvc.RecordEvent(
        ctx, 
        escrow.BuyerID, 
        trust.EventBuyerVerificationTimeout,
        &escrow.ID, 
        "Buyer did not verify delivery within inspection period — funds auto-released",
    ); err != nil {
        log.Printf("[AutoReleaseWorker] Error recording trust penalty for buyer %s: %v", escrow.BuyerID, err)
        // Don't fail the auto-release for this
    }
    
    // ... existing notification logic ...
}
```

### Validation Criteria
- [ ] When auto-release runs, buyer's trust score decreases by 10
- [ ] Trust event has correct `escrow_id` reference
- [ ] Trust event reason includes explanation about inspection period expiry
- [ ] Auto-release still succeeds even if trust recording fails (non-blocking)
- [ ] Worker test `TestWorkerAutoRelease` updated to verify trust event creation

---

## Task 2.3 — Backend: Auto-Release Reminder Includes Rating Warning

### Description
Update the auto-release reminder notifications (24h and 12h before auto-release) to include a warning about the trust score penalty.

### Files to Modify
- [`internal/escrow/worker.go`](file:///Users/oluwatobiloba/Desktop/qsd/vescrow/vescrow_backend/internal/escrow/worker.go) — Update notification message templates

### Implementation Details
Update the system message and notification content for the 24h and 12h reminders:

```go
// 24h reminder
"Your inspection period expires in 24 hours. If you do not verify the delivery, funds will be auto-released to the seller and your trust rating will be negatively affected (-10 points)."

// 12h reminder  
"URGENT: Your inspection period expires in 12 hours. Failure to verify will result in auto-release of funds AND a trust score penalty. Please verify now to avoid a rating impact."
```

### Validation Criteria
- [ ] 24h reminder message includes trust penalty warning
- [ ] 12h reminder message includes urgent trust penalty warning
- [ ] Worker tests for reminders updated with new message content

---

## Task 2.4 — Frontend: Display Trust Warning in Verify Modal

### Description
In the "Verify Product Condition & Inspection" modal, display a notice about the trust rating consequence of not verifying.

### Files to Modify
- [`src/app/app/escrows/[id]/page.tsx`](file:///Users/oluwatobiloba/Desktop/qsd/vescrow/vescrow_frontend/src/app/app/escrows/%5Bid%5D/page.tsx)

### Implementation Details
Add a warning banner inside the verify modal:

```tsx
<div className="p-3 bg-amber-500/10 border border-amber-500/30 rounded-xl text-xs text-amber-300 flex items-center gap-2">
  <AlertTriangle className="w-4 h-4 shrink-0" />
  <span>
    <strong>Trust Rating Notice:</strong> Failing to verify delivery before the inspection 
    period expires will result in automatic fund release and a <strong>-10 point</strong> impact 
    to your trust score.
  </span>
</div>
```

Also show the inspection period deadline prominently:
```tsx
<div className="flex justify-between text-xs text-slate-300">
  <span>Inspection deadline:</span>
  <span className="font-bold text-white">
    {formatDate(inspectionDeadline)} ({daysRemaining} days remaining)
  </span>
</div>
```

### Validation Criteria
- [ ] Warning banner visible in verify modal
- [ ] Inspection deadline shown with countdown
- [ ] No warning shown when escrow is not in `shipped` state

---

## Phase 2 Validation Criteria

- [ ] New trust event type exists and has correct delta (-10)
- [ ] Auto-release worker records trust penalty against buyer
- [ ] Reminder notifications include trust penalty warnings
- [ ] Frontend verify modal shows trust warning
- [ ] A buyer going through 10 auto-releases drops from 100 → 0 score
- [ ] Trust status transitions correctly: good (≥70) → warning (≥40) → red (≥20) → blocked (<20)
- [ ] All existing trust system tests still pass
