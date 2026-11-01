# Escrow Transaction Chat Activity Log — Implementation Plan

**Task ID**: `VESC-CHAT-2026-001`  
**Created**: 2026-09-27  
**Status**: Draft — Awaiting Approval  
**Goal**: Every escrow action (from both buyer and seller) should automatically appear as a system message in the escrow transaction chat

---

## Executive Summary

Currently, the escrow chat (`Messages` model) is only used for free-text messages typed by buyer/seller. All lifecycle actions (accept, fund, ship, deliver, dispute, etc.) are tracked only via status changes, audit logs, and email notifications — **but none of them appear in the chat timeline**.

This plan will make every escrow action emit an automatic **system message** into the chat thread, giving both parties a complete, transparent activity log inside the escrow conversation. Think of it like how WhatsApp shows "You changed the group description" — but for escrow lifecycle events.

---

## Architecture Design Decisions

### 1. Message Type Differentiation

> [!IMPORTANT]
> The `Message` model needs a new `message_type` field to distinguish **user-typed chat messages** from **system-generated activity messages**. This is the core architectural change.

```
message_type = "user"    → Regular chat message from buyer/seller
message_type = "system"  → Auto-generated activity log entry
```

### 2. System Sender Strategy

For system messages, `sender_id` will be set to the **actor who performed the action** (buyer or seller UUID), not a special "system" user. This way:
- The chat thread shows "Buyer funded the escrow" attributed to the buyer
- The chat thread shows "Seller marked as shipped" attributed to the seller
- Worker/system actions (auto-release, expiry) will use the **escrow buyer's ID** as sender with a `"system"` message type so they still display correctly

### 3. Metadata for Rich Messages

System messages will carry optional `metadata` (JSONB) for embedding evidence links, tracking numbers, amounts, and other structured data alongside the text content. This enables the frontend to render rich activity cards instead of plain text.

---

## Complete Action → Chat Message Mapping

| # | Action | Actor | Chat Message Content | Metadata |
|---|--------|-------|---------------------|----------|
| 1 | **Create Escrow** | Buyer | `"created this escrow transaction for {amount}"` | `{event: "escrow_created", amount_kobo}` |
| 2 | **Submit to Seller** | Buyer | `"submitted the escrow terms to the seller for review"` | `{event: "escrow_submitted"}` |
| 3 | **Accept Terms** | Seller | `"accepted the escrow terms"` | `{event: "escrow_accepted"}` |
| 4 | **Reject Terms** | Seller | `"rejected the escrow terms"` | `{event: "escrow_rejected"}` |
| 5 | **Fund Escrow** | Buyer | `"funded the escrow with {amount}"` | `{event: "escrow_funded", amount_kobo}` |
| 6 | **Submit Shipment Evidence** | Seller | `"submitted pre-shipment evidence and photos"` | `{event: "shipment_evidence", image_count, has_video}` |
| 7 | **Mark as Shipped** | Seller | `"marked item as shipped via {courier}, tracking: {tracking_number}"` | `{event: "escrow_shipped", tracking_number, courier}` |
| 8 | **Confirm Delivery** | Buyer | `"confirmed delivery was received"` | `{event: "delivery_confirmed", image_count}` |
| 9 | **Verify Delivery** | Buyer | `"verified product condition — accepted"` | `{event: "delivery_verified", condition_accepted}` |
| 10 | **Approve & Release** | Buyer | `"approved delivery and released funds to seller"` | `{event: "escrow_completed"}` |
| 11 | **Mark Delivered** (legacy) | Seller | `"marked the delivery as complete"` | `{event: "escrow_delivered"}` |
| 12 | **Cancel Escrow** | Buyer | `"cancelled this escrow transaction"` | `{event: "escrow_cancelled"}` |
| 13 | **Open Dispute** | Buyer | `"opened a dispute: {reason}"` | `{event: "dispute_opened", category, reason}` |
| 14 | **Dispute Resolved** | System | `"Dispute resolved by admin — {resolution_type}"` | `{event: "dispute_resolved", resolution}` |
| 15 | **Extend Inspection** | Buyer | `"extended inspection period by {days} additional days"` | `{event: "inspection_extended", additional_days}` |
| 16 | **Auto-Release** | System | `"Inspection period expired — funds auto-released to seller"` | `{event: "auto_released"}` |
| 17 | **Escrow Expired** | System | `"Escrow expired — seller did not respond in time"` | `{event: "escrow_expired"}` |
| 18 | **Submit Product Details** | Seller | `"added product photos and description"` | `{event: "product_details_submitted", image_count}` |
| 19 | **Complete Milestone** | Seller | `"marked milestone \"{title}\" as complete"` | `{event: "milestone_completed", milestone_id, title}` |
| 20 | **Approve Milestone** | Buyer | `"approved milestone \"{title}\""` | `{event: "milestone_approved", milestone_id, title}` |
| 21 | **Seller Response (Dispute)** | Seller | `"responded to the dispute"` | `{event: "dispute_seller_responded"}` |
| 22 | **Upload Dispute Evidence** | Both | `"uploaded evidence for the dispute"` | `{event: "dispute_evidence_uploaded", file_type}` |
| 23 | **Payment Webhook Success** | System | `"Payment confirmed by Paystack — escrow is now funded"` | `{event: "payment_confirmed", reference}` |

---

## Phase 1: Backend — Data Model & Helper

**Objective**: Extend the `Message` model with type/metadata fields and create a reusable helper function for emitting system messages.

### Task 1.1 — Extend `Message` Model

**File**: `internal/escrow/model.go`

**Changes**:
```diff
 type Message struct {
   ID            uuid.UUID `gorm:"type:uuid;primaryKey;default:uuid_generate_v4()" json:"id"`
   EscrowID      uuid.UUID `gorm:"type:uuid;not null;index" json:"escrow_id"`
   SenderID      uuid.UUID `gorm:"type:uuid;not null;index" json:"sender_id"`
   Sender        auth.User `gorm:"foreignKey:SenderID" json:"sender,omitempty"`
   Content       string    `gorm:"type:text;not null" json:"content"`
+  MessageType   string    `gorm:"type:varchar(10);default:'user';not null" json:"message_type"`
+  EventType     *string   `gorm:"type:varchar(50)" json:"event_type,omitempty"`
+  Metadata      datatypes.JSON `gorm:"type:jsonb" json:"metadata,omitempty"`
   AttachmentURL *string   `gorm:"type:varchar(500)" json:"attachment_url,omitempty"`
   CreatedAt     time.Time `json:"created_at"`
 }
```

**Validation Criteria**:
- [ ] Database migration runs without errors (GORM AutoMigrate adds columns)
- [ ] Existing messages default to `message_type = "user"` with NULL `event_type` and `metadata`
- [ ] New columns are nullable-safe — no breaking change to existing data
- [ ] JSON serialization includes new fields

---

### Task 1.2 — Create System Message Helper

**File**: `internal/escrow/service.go` (add new method)

**New Method**: `emitSystemMessage`

```go
// emitSystemMessage inserts an automatic activity-log message into the escrow chat thread.
// It is fire-and-forget: errors are logged but never block the parent action.
func (s *EscrowService) emitSystemMessage(ctx context.Context, escrowID, actorID uuid.UUID, event, content string, metadata map[string]interface{}) {
    metaJSON, _ := json.Marshal(metadata)
    msg := &Message{
        ID:          uuid.New(),
        EscrowID:    escrowID,
        SenderID:    actorID,
        Content:     content,
        MessageType: "system",
        EventType:   &event,
        Metadata:    datatypes.JSON(metaJSON),
    }
    if err := s.repo.CreateMessage(ctx, msg); err != nil {
        log.Printf("[SystemMessage] failed to emit %s for escrow %s: %v", event, escrowID, err)
    }
}
```

**Design Rationale**:
- Fire-and-forget pattern: system messages should never cause a transaction to fail
- Actor attribution: the `actorID` is always the user performing the action
- Structured metadata enables rich frontend rendering

**Validation Criteria**:
- [ ] Helper compiles without errors
- [ ] Calling `emitSystemMessage` inserts a row with `message_type = "system"`
- [ ] JSON metadata is correctly serialized and stored
- [ ] Errors in message insertion are logged but do not propagate

---

### Task 1.3 — Expose Helper to Dispute Service

**File**: `internal/escrow/service.go`

**New Exported Method**:

```go
// EmitSystemMessage is the exported version for cross-package use (e.g., dispute service).
func (s *EscrowService) EmitSystemMessage(ctx context.Context, escrowID, actorID uuid.UUID, event, content string, metadata map[string]interface{}) {
    s.emitSystemMessage(ctx, escrowID, actorID, event, content, metadata)
}
```

**Validation Criteria**:
- [ ] `EmitSystemMessage` is accessible from `dispute` and `payment` packages
- [ ] No import cycle introduced

---

### Phase 1 Validation

- [ ] `go build ./...` succeeds
- [ ] Database migration adds 3 new columns to `messages` table: `message_type`, `event_type`, `metadata`
- [ ] Existing messages remain intact with default `message_type = 'user'`
- [ ] Unit test: `emitSystemMessage` creates a message with correct fields

---

## Phase 2: Backend — Wire System Messages into All Escrow Actions

**Objective**: Add `emitSystemMessage` calls to every service method that performs a lifecycle action.

### Task 2.1 — CreateEscrow

**File**: `internal/escrow/service.go` → `CreateEscrow()` method

**Insert after** the escrow record is created (after the transaction block, ~line 173):

```go
s.emitSystemMessage(ctx, escrowRecord.ID, buyerID, "escrow_created",
    fmt.Sprintf("created this escrow transaction for %s", notificationFormatKobo(escrowRecord.AmountKobo)),
    map[string]interface{}{"event": "escrow_created", "amount_kobo": escrowRecord.AmountKobo},
)
```

**Validation Criteria**:
- [ ] Creating an escrow produces a system message in the chat thread
- [ ] Message content includes formatted amount
- [ ] Metadata contains `event` and `amount_kobo`

---

### Task 2.2 — SubmitToSeller

**File**: `internal/escrow/service.go` → `SubmitToSeller()` method

**Insert after** status update:

```go
s.emitSystemMessage(ctx, escrow.ID, userID, "escrow_submitted",
    "submitted the escrow terms to the seller for review",
    map[string]interface{}{"event": "escrow_submitted"},
)
```

**Validation Criteria**:
- [ ] Submitting escrow to seller creates a system chat message
- [ ] Message attributed to buyer

---

### Task 2.3 — AcceptEscrow (acceptEscrowRecord)

**File**: `internal/escrow/invite.go` → `acceptEscrowRecord()` method

**Insert after** status update:

```go
s.emitSystemMessage(ctx, escrow.ID, userID, "escrow_accepted",
    "accepted the escrow terms",
    map[string]interface{}{"event": "escrow_accepted"},
)
```

**Validation Criteria**:
- [ ] Seller accepting escrow creates a system message
- [ ] Message attributed to seller

---

### Task 2.4 — RejectEscrow (rejectEscrowRecord)

**File**: `internal/escrow/invite.go` → `rejectEscrowRecord()` method

**Insert after** status update:

```go
s.emitSystemMessage(ctx, escrow.ID, userID, "escrow_rejected",
    "rejected the escrow terms",
    map[string]interface{}{"event": "escrow_rejected"},
)
```

**Validation Criteria**:
- [ ] Seller rejecting escrow creates a system message

---

### Task 2.5 — FundEscrow

**File**: `internal/escrow/service.go` → `FundEscrow()` method

**Insert after** status update:

```go
s.emitSystemMessage(ctx, escrow.ID, userID, "escrow_funded",
    fmt.Sprintf("funded the escrow with %s", notificationFormatKobo(escrow.AmountKobo)),
    map[string]interface{}{"event": "escrow_funded", "amount_kobo": escrow.AmountKobo},
)
```

**Validation Criteria**:
- [ ] Funding escrow creates a system message with amount

---

### Task 2.6 — SubmitShipmentEvidence

**File**: `internal/escrow/service.go` → `SubmitShipmentEvidence()` method

**Insert after** transaction commits:

```go
imageCount := len(dto.ProductImageURLs) + len(dto.PackagingImageURLs) + len(dto.ConditionImageURLs) + len(dto.AccessoriesImageURLs)
s.emitSystemMessage(ctx, escrow.ID, userID, "shipment_evidence_submitted",
    "submitted pre-shipment evidence and photos",
    map[string]interface{}{"event": "shipment_evidence", "image_count": imageCount, "has_video": dto.VideoURL != ""},
)
```

**Validation Criteria**:
- [ ] Submitting shipment evidence creates a system message
- [ ] Metadata includes image count and video flag

---

### Task 2.7 — MarkAsShipped

**File**: `internal/escrow/service.go` → `MarkAsShipped()` method

**Insert after** transaction commits:

```go
s.emitSystemMessage(ctx, escrow.ID, userID, "escrow_shipped",
    fmt.Sprintf("marked item as shipped via %s, tracking: %s", dto.Courier, dto.TrackingNumber),
    map[string]interface{}{"event": "escrow_shipped", "tracking_number": dto.TrackingNumber, "courier": dto.Courier},
)
```

**Validation Criteria**:
- [ ] Marking as shipped creates a system message with courier and tracking info
- [ ] Metadata contains tracking details

---

### Task 2.8 — ConfirmDelivery

**File**: `internal/escrow/service.go` → `ConfirmDelivery()` method

**Insert after** transaction commits:

```go
s.emitSystemMessage(ctx, escrow.ID, userID, "delivery_confirmed",
    "confirmed delivery was received",
    map[string]interface{}{"event": "delivery_confirmed", "image_count": len(dto.ImageURLs)},
)
```

**Validation Criteria**:
- [ ] Confirming delivery creates a system message

---

### Task 2.9 — VerifyDelivery

**File**: `internal/escrow/service.go` → `VerifyDelivery()` method

**Insert after** transaction commits:

```go
s.emitSystemMessage(ctx, escrow.ID, userID, "delivery_verified",
    "verified product condition — accepted",
    map[string]interface{}{"event": "delivery_verified", "condition_accepted": dto.ConditionAccepted},
)
```

**Validation Criteria**:
- [ ] Verifying delivery creates a system message

---

### Task 2.10 — ApproveDelivery

**File**: `internal/escrow/service.go` → `ApproveDelivery()` method

**Insert after** status update and fund release:

```go
s.emitSystemMessage(ctx, escrow.ID, userID, "escrow_completed",
    "approved delivery and released funds to seller",
    map[string]interface{}{"event": "escrow_completed"},
)
```

**Validation Criteria**:
- [ ] Approving delivery creates a system message

---

### Task 2.11 — MarkDelivered (legacy flow)

**File**: `internal/escrow/service.go` → `MarkDelivered()` method

**Insert after** status update:

```go
s.emitSystemMessage(ctx, escrow.ID, userID, "escrow_delivered",
    "marked the delivery as complete",
    map[string]interface{}{"event": "escrow_delivered"},
)
```

**Validation Criteria**:
- [ ] Legacy delivery flow creates a system message

---

### Task 2.12 — CancelEscrowDraft

**File**: `internal/escrow/service.go` → `CancelEscrowDraft()` method

**Insert after** status update:

```go
s.emitSystemMessage(ctx, escrow.ID, userID, "escrow_cancelled",
    "cancelled this escrow transaction",
    map[string]interface{}{"event": "escrow_cancelled"},
)
```

**Validation Criteria**:
- [ ] Cancelling escrow creates a system message

---

### Task 2.13 — DisputeEscrow

**File**: `internal/escrow/service.go` → `DisputeEscrow()` method

**Insert after** status update:

```go
s.emitSystemMessage(ctx, escrow.ID, userID, "dispute_opened",
    "opened a dispute on this transaction",
    map[string]interface{}{"event": "dispute_opened"},
)
```

**Validation Criteria**:
- [ ] Opening dispute from escrow service creates a system message

---

### Task 2.14 — ExtendInspection

**File**: `internal/escrow/service.go` → `ExtendInspection()` method

**Insert after** update:

```go
s.emitSystemMessage(ctx, escrow.ID, userID, "inspection_extended",
    fmt.Sprintf("extended inspection period by %d additional days (now %d total)", additionalDays, totalDays),
    map[string]interface{}{"event": "inspection_extended", "additional_days": additionalDays, "total_days": totalDays},
)
```

**Validation Criteria**:
- [ ] Extending inspection creates a system message with day counts

---

### Task 2.15 — CompleteMilestone

**File**: `internal/escrow/service.go` → `CompleteMilestone()` method

**Insert after** milestone save:

```go
s.emitSystemMessage(ctx, id, userID, "milestone_completed",
    fmt.Sprintf("marked milestone \"%s\" as complete", milestone.Title),
    map[string]interface{}{"event": "milestone_completed", "milestone_id": milestone.ID, "title": milestone.Title},
)
```

**Validation Criteria**:
- [ ] Completing milestone creates a system message with milestone title

---

### Task 2.16 — ApproveMilestone

**File**: `internal/escrow/service.go` → `ApproveMilestone()` method

**Insert after** milestone save:

```go
s.emitSystemMessage(ctx, id, userID, "milestone_approved",
    fmt.Sprintf("approved milestone \"%s\"", milestone.Title),
    map[string]interface{}{"event": "milestone_approved", "milestone_id": milestone.ID, "title": milestone.Title},
)
```

**Validation Criteria**:
- [ ] Approving milestone creates a system message

---

### Phase 2 Validation

- [ ] `go build ./...` succeeds
- [ ] All 16 escrow service methods emit system messages
- [ ] System messages are correctly attributed to the acting user
- [ ] No existing tests break (system messages are additive)
- [ ] Manual test: perform full escrow lifecycle and verify chat thread contains all activity messages
- [ ] Existing user-typed messages continue to work unchanged with `message_type = "user"`

---

## Phase 3: Backend — Wire System Messages into Worker & Cross-Module Actions

**Objective**: Add system messages for automated worker actions, dispute service actions, and payment webhook events.

### Task 3.1 — Auto-Release Worker

**File**: `internal/escrow/worker.go` → `autoReleaseEscrow()` method

**Insert after** completion update:

```go
s.emitSystemMessage(ctx, escrow.ID, escrow.BuyerID, "auto_released",
    "Inspection period expired — funds automatically released to seller",
    map[string]interface{}{"event": "auto_released"},
)
```

**Validation Criteria**:
- [ ] Auto-release emits a system message attributed to the buyer ID
- [ ] Message is clearly marked as a system/automated action

---

### Task 3.2 — Expiry Worker

**File**: `internal/escrow/worker.go` → `expireEscrow()` method

**Insert after** status update:

```go
s.emitSystemMessage(ctx, escrow.ID, escrow.BuyerID, "escrow_expired",
    "Escrow expired — seller did not respond in time",
    map[string]interface{}{"event": "escrow_expired"},
)
```

**Validation Criteria**:
- [ ] Expired escrows emit a system message

---

### Task 3.3 — Dispute Service: OpenDispute

**File**: `internal/dispute/service.go` → `OpenDispute()` method

**Requires**: Access to `EscrowService.EmitSystemMessage` — pass escrow service as a dependency or use a shared interface.

**Approach**: Add `EscrowSystemMessenger` interface to dispute service:

```go
// In dispute/service.go
type EscrowSystemMessenger interface {
    EmitSystemMessage(ctx context.Context, escrowID, actorID uuid.UUID, event, content string, metadata map[string]interface{})
}
```

Add field to `DisputeService`:
```go
escrowMessenger EscrowSystemMessenger
```

**Insert after** dispute creation:

```go
if s.escrowMessenger != nil {
    s.escrowMessenger.EmitSystemMessage(ctx, escrowID, buyerID, "dispute_opened",
        fmt.Sprintf("opened a dispute: %s — %s", dto.Category, dto.Reason),
        map[string]interface{}{"event": "dispute_opened", "category": dto.Category, "reason": dto.Reason},
    )
}
```

**Validation Criteria**:
- [ ] Opening dispute from dispute service emits a system chat message
- [ ] No import cycle between `dispute` and `escrow` packages

---

### Task 3.4 — Dispute Service: SubmitSellerResponse

**File**: `internal/dispute/service.go` → `SubmitSellerResponse()` method

**Insert after** response saved:

```go
if s.escrowMessenger != nil {
    s.escrowMessenger.EmitSystemMessage(ctx, dispute.EscrowID, sellerID, "dispute_seller_responded",
        "responded to the dispute",
        map[string]interface{}{"event": "dispute_seller_responded"},
    )
}
```

**Validation Criteria**:
- [ ] Seller responding to dispute emits a system message

---

### Task 3.5 — Dispute Service: UploadEvidence

**File**: `internal/dispute/service.go` → `UploadEvidence()` method

**Insert after** evidence saved:

```go
if s.escrowMessenger != nil {
    s.escrowMessenger.EmitSystemMessage(ctx, dispute.EscrowID, userID, "dispute_evidence_uploaded",
        "uploaded evidence for the dispute",
        map[string]interface{}{"event": "dispute_evidence_uploaded", "file_type": dto.FileType},
    )
}
```

**Validation Criteria**:
- [ ] Uploading dispute evidence emits a system message

---

### Task 3.6 — Dispute Service: ResolveDispute

**File**: `internal/dispute/service.go` → `ResolveDispute()` method

**Insert after** resolution:

```go
if s.escrowMessenger != nil {
    resolutionLabel := strings.ReplaceAll(string(resolution), "_", " ")
    s.escrowMessenger.EmitSystemMessage(ctx, dispute.EscrowID, adminID, "dispute_resolved",
        fmt.Sprintf("Dispute resolved by admin — %s", resolutionLabel),
        map[string]interface{}{"event": "dispute_resolved", "resolution": resolution, "notes": dto.Notes},
    )
}
```

**Validation Criteria**:
- [ ] Resolving dispute emits a system message with resolution type

---

### Task 3.7 — Payment Webhook: Fund Confirmation

**File**: `internal/payment/service.go` → The method that handles successful Paystack payment webhook

**Approach**: After the webhook handler transitions escrow to `funded`, emit a system message. Requires passing escrow messenger interface into payment service during wiring.

```go
if s.escrowMessenger != nil {
    s.escrowMessenger.EmitSystemMessage(ctx, escrow.ID, escrow.BuyerID, "payment_confirmed",
        fmt.Sprintf("Payment confirmed — escrow is now funded with %s", formattedAmount),
        map[string]interface{}{"event": "payment_confirmed", "reference": paymentReference},
    )
}
```

**Validation Criteria**:
- [ ] Successful payment webhook emits a system message
- [ ] Message includes payment reference

---

### Task 3.8 — Product Details Submission

**File**: `internal/escrow/product_details.go`

**Insert after** product details are saved:

```go
s.emitSystemMessage(ctx, escrow.ID, userID, "product_details_submitted",
    "added product photos and description",
    map[string]interface{}{"event": "product_details_submitted", "image_count": len(dto.Images)},
)
```

**Validation Criteria**:
- [ ] Submitting product details creates a system message

---

### Task 3.9 — Wire Dependencies in main.go

**File**: `cmd/server/main.go`

**Changes**:
- Pass `escrowService` as `EscrowSystemMessenger` to `dispute.NewDisputeService()`
- Pass `escrowService` as `EscrowSystemMessenger` to `payment.Service` (if applicable)

```go
disputeService.SetEscrowMessenger(escrowService)
// or via constructor if refactored
```

**Validation Criteria**:
- [ ] Application starts without nil pointer panics
- [ ] All messenger dependencies are correctly wired
- [ ] `go build ./...` passes

---

### Phase 3 Validation

- [ ] All worker/cron actions emit system messages
- [ ] All dispute lifecycle actions emit system messages
- [ ] Payment webhook emits system message on successful funding
- [ ] No import cycles introduced
- [ ] Full integration: run complete escrow + dispute lifecycle, verify chat thread has complete history
- [ ] System messages from workers are correctly attributed

---

## Phase 4: Frontend — Display System Messages in Chat

**Objective**: Update the frontend chat component to render system messages as distinct activity cards alongside regular messages.

### Task 4.1 — Update TypeScript Types

**File**: `src/lib/types.ts`

**Add new fields to Message type**:

```typescript
interface Message {
  id: string;
  escrow_id: string;
  sender_id: string;
  sender?: User;
  sender_name?: string;
  content: string;
  message_type: 'user' | 'system';      // NEW
  event_type?: string;                    // NEW
  metadata?: Record<string, any>;         // NEW
  attachment_url?: string;
  created_at: string;
}
```

**Validation Criteria**:
- [ ] Types compile without errors
- [ ] New fields are optional to maintain backward compatibility

---

### Task 4.2 — Create System Message Component

**File**: `src/components/escrow/system-message.tsx` (NEW)

**Description**: A visually distinct activity log entry for the chat timeline. Renders as a centered, muted card (not aligned left/right like user messages).

**Design**:
- Centered horizontal layout
- Muted background (`bg-slate-800/40 border border-white/5`)
- Small icon matching the event type (scale for funded, truck for shipped, check for verified, etc.)
- Event-specific text with timestamp
- Optional expandable metadata (tracking number, amount, etc.)

```tsx
function SystemMessage({ message }: { message: Message }) {
  const icon = getEventIcon(message.event_type);
  return (
    <div className="flex justify-center my-2">
      <div className="flex items-center gap-2 px-4 py-2 rounded-full bg-slate-800/50 border border-white/5 max-w-md">
        {icon}
        <span className="text-[11px] text-slate-400">{message.content}</span>
        <span className="text-[10px] text-slate-500">{formatTime(message.created_at)}</span>
      </div>
    </div>
  );
}
```

**Validation Criteria**:
- [ ] System messages render as centered, muted pill/badge elements
- [ ] Visually distinct from user chat bubbles
- [ ] Correct icon per event type
- [ ] Timestamp displayed

---

### Task 4.3 — Update Chat Thread Rendering

**File**: `src/app/app/escrows/[id]/page.tsx` → Messages tab (line ~527-565)

**Changes**: Update the message rendering loop to differentiate between user and system messages:

```tsx
{chatThread.map((msg) => {
  if (msg.message_type === 'system') {
    return <SystemMessage key={msg.id} message={msg} />;
  }
  
  const isMe = msg.sender_id === user?.id;
  return (
    // ... existing user message bubble rendering
  );
})}
```

**Validation Criteria**:
- [ ] User messages continue rendering as left/right aligned bubbles
- [ ] System messages render as centered activity cards
- [ ] Mixed timeline (user + system messages) displays correctly in chronological order
- [ ] Scrolling behavior works with mixed message types

---

### Task 4.4 — Event Icon Mapping

**File**: `src/components/escrow/system-message.tsx` (within the new component)

**Map event types to Lucide icons**:

| Event Type | Icon | Color |
|-----------|------|-------|
| `escrow_created` | `Plus` | emerald |
| `escrow_submitted` | `Send` | blue |
| `escrow_accepted` | `CheckCircle` | emerald |
| `escrow_rejected` | `XCircle` | rose |
| `escrow_funded` | `DollarSign` | emerald |
| `shipment_evidence` | `Camera` | blue |
| `escrow_shipped` | `Truck` | blue |
| `delivery_confirmed` | `Package` | emerald |
| `delivery_verified` | `ShieldCheck` | emerald |
| `escrow_completed` | `CheckCheck` | emerald |
| `escrow_cancelled` | `Ban` | rose |
| `dispute_opened` | `AlertTriangle` | amber |
| `dispute_resolved` | `Scale` | emerald |
| `auto_released` | `Clock` | amber |
| `escrow_expired` | `Timer` | rose |
| `milestone_completed` | `Flag` | blue |
| `milestone_approved` | `ThumbsUp` | emerald |
| `payment_confirmed` | `CreditCard` | emerald |

**Validation Criteria**:
- [ ] Every event type has a mapped icon
- [ ] Icons render correctly with appropriate colors
- [ ] Unknown event types fall back to a generic info icon

---

### Task 4.5 — Messages Tab Rename

**File**: `src/app/app/escrows/[id]/page.tsx` → Tabs definition (line ~335-345)

**Change**: Update the "Messages" tab label to "Activity" to reflect the broader purpose:

```tsx
{
  id: 'messages',
  label: 'Activity',
  icon: <MessageSquare className="w-4 h-4" />,
  count: chatThread.length
}
```

**Validation Criteria**:
- [ ] Tab label reflects the broader scope (activity log + chat)
- [ ] Badge count includes both user and system messages

---

### Phase 4 Validation

- [ ] Chat thread shows user messages as styled left/right bubbles
- [ ] System messages display as centered muted activity entries
- [ ] Full escrow lifecycle produces a complete activity timeline in the chat
- [ ] Icons match event types correctly
- [ ] Responsive design works on mobile (375px) and desktop (1440px)
- [ ] No TypeScript compilation errors
- [ ] `npm run build` succeeds

---

## Phase 5: Frontend — Polish & Edge Cases

**Objective**: Handle edge cases, add micro-animations, ensure backward compatibility, and optimize performance.

### Task 5.1 — Backward Compatibility

**File**: `src/components/escrow/system-message.tsx` and `page.tsx`

**Handle**: Messages from API responses that don't have the `message_type` field (older data):

```typescript
const messageType = msg.message_type ?? 'user';
```

**Validation Criteria**:
- [ ] Older messages without `message_type` default to `"user"` rendering
- [ ] No crashes on null/undefined `message_type`, `event_type`, or `metadata`

---

### Task 5.2 — Rich Metadata Rendering

**File**: `src/components/escrow/system-message.tsx`

**Enhancements**: For specific event types, expand metadata into richer cards:

- **`escrow_shipped`**: Show tracking number and courier in a mini-card below the text
- **`escrow_funded`**: Show the formatted amount
- **`dispute_opened`**: Show the dispute category badge
- **`milestone_completed` / `milestone_approved`**: Show milestone title as a badge

**Validation Criteria**:
- [ ] Shipped messages show tracking info
- [ ] Funded messages show amount
- [ ] Expandable metadata doesn't break layout on mobile

---

### Task 5.3 — Auto-scroll and Animation

**File**: Escrow detail page chat section

**Changes**:
- Auto-scroll to bottom when new messages (user or system) appear
- Add a subtle slide-in animation for new system messages
- Add a "jump to latest" button when scrolled up

**Validation Criteria**:
- [ ] Chat auto-scrolls to latest message on load
- [ ] New messages trigger smooth scroll
- [ ] Animation doesn't cause jank (tested at 60fps)

---

### Task 5.4 — Empty State Update

**File**: Escrow detail page chat section

**Change**: Update the empty chat state to acknowledge that system messages will populate automatically:

```tsx
<p className="text-xs text-slate-400">
  Activity and messages will appear here as the transaction progresses.
</p>
```

**Validation Criteria**:
- [ ] Empty state is user-friendly and sets expectations
- [ ] Not confusing for first-time users

---

### Task 5.5 — Filter Controls (Optional Enhancement)

**File**: Escrow detail page messages tab

**Add toggle buttons** to filter the chat view:
- **All** — shows everything (default)
- **Activity** — shows only system messages
- **Chat** — shows only user messages

```tsx
<div className="flex gap-2 mb-3">
  <button className={filter === 'all' ? 'active' : ''} onClick={() => setFilter('all')}>All</button>
  <button className={filter === 'activity' ? 'active' : ''} onClick={() => setFilter('activity')}>Activity</button>
  <button className={filter === 'chat' ? 'active' : ''} onClick={() => setFilter('chat')}>Chat</button>
</div>
```

**Validation Criteria**:
- [ ] Filter toggles work correctly
- [ ] Default is "All"
- [ ] Counts update per filter

---

### Phase 5 Validation

- [ ] No crashes with missing/null fields
- [ ] Rich metadata cards render correctly
- [ ] Chat auto-scrolls properly
- [ ] Filter controls work if implemented
- [ ] Full responsive verification (mobile + desktop)
- [ ] Performance: no lag with 50+ messages in thread

---

## Phase 6: Testing & QA

**Objective**: Write tests, verify the complete flow, and ensure no regressions.

### Task 6.1 — Backend Unit Tests

**File**: `internal/escrow/service_test.go` (extend existing)

**Test Cases**:
1. `TestEmitSystemMessage_CreatesMessage` — Verify helper creates message with correct fields
2. `TestCreateEscrow_EmitsSystemMessage` — Verify escrow creation emits chat message
3. `TestAcceptEscrow_EmitsSystemMessage` — Verify acceptance emits message
4. `TestFundEscrow_EmitsSystemMessage` — Verify funding emits message
5. `TestMarkAsShipped_EmitsSystemMessage` — Verify shipping emits message with tracking
6. `TestSystemMessage_DoesNotBlockOnError` — Verify message creation failure doesn't crash the parent action

**Validation Criteria**:
- [ ] All tests pass
- [ ] Tests verify message_type, event_type, sender_id, and content
- [ ] Failure resilience test confirms fire-and-forget behavior

---

### Task 6.2 — Backend Integration Test: Full Lifecycle

**File**: `internal/escrow/lifecycle_test.go` (extend existing)

**Test**: Walk through the complete lifecycle in a single test:
1. Create escrow → verify system message
2. Submit to seller → verify system message
3. Accept → verify system message
4. Fund → verify system message
5. Submit evidence → verify system message
6. Ship → verify system message
7. Confirm delivery → verify system message
8. Verify delivery → verify system message
9. Approve → verify system message
10. Query all messages: verify 9 system messages + correct chronological order

**Validation Criteria**:
- [ ] Full lifecycle produces exactly the expected number of system messages
- [ ] Messages are in correct chronological order
- [ ] All messages have `message_type = "system"`

---

### Task 6.3 — Frontend Component Test

Verify the `SystemMessage` component renders correctly for all event types.

**Validation Criteria**:
- [ ] All 18+ event types render without errors
- [ ] Unknown event types render with fallback icon
- [ ] Component handles null metadata gracefully

---

### Task 6.4 — End-to-End Smoke Test

**Manual walkthrough**:
1. Create escrow as buyer
2. Accept as seller
3. Fund as buyer
4. Submit evidence as seller
5. Mark shipped as seller
6. Confirm delivery as buyer
7. Verify delivery as buyer
8. Approve & release as buyer
9. Send a regular chat message
10. Open the Activity/Messages tab

**Expected result**: All 8 lifecycle events + 1 user message visible in the chat thread, in correct order, with correct icons and attribution.

**Validation Criteria**:
- [ ] All actions visible in chat
- [ ] System messages are centered, user messages are aligned
- [ ] Icons and colors match event types
- [ ] Timestamps are correct
- [ ] Chat is scrollable and performant

---

### Phase 6 Validation

- [ ] All unit tests pass: `go test ./internal/escrow/... -v`
- [ ] Integration lifecycle test passes
- [ ] Frontend builds: `npm run build`
- [ ] End-to-end smoke test passes
- [ ] No console errors in frontend

---

## Summary

| Phase | Tasks | Key Deliverables |
|-------|-------|-----------------|
| 1. Data Model & Helper | 3 | Extended `Message` model, `emitSystemMessage` helper, exported interface |
| 2. Escrow Service Wiring | 16 | System messages in all 16 escrow service methods |
| 3. Cross-Module Wiring | 9 | Worker, dispute, payment webhook system messages + DI wiring |
| 4. Frontend Display | 5 | `SystemMessage` component, chat rendering update, icons, tab rename |
| 5. Polish & Edge Cases | 5 | Backward compat, rich metadata, auto-scroll, empty state, filters |
| 6. Testing & QA | 4 | Unit tests, integration test, component test, E2E smoke test |
| **Total** | **42** | **23 unique escrow events → chat messages, full lifecycle coverage** |

---

## Risk Considerations

> [!WARNING]
> **Database Migration**: Adding columns to the `messages` table is safe (nullable with defaults), but should be tested against production data volume. GORM AutoMigrate handles this gracefully.

> [!NOTE]
> **Performance**: System messages add 1 extra DB write per escrow action. This is negligible since escrow actions are infrequent (max ~10 per transaction lifetime). The fire-and-forget pattern ensures no latency impact on the parent action.

> [!CAUTION]
> **Import Cycles**: The `dispute` package already imports `escrow`. Using an interface (`EscrowSystemMessenger`) injected via setter avoids reverse imports. This pattern is already used in the codebase (`WalletService`, `PaymentDisburser`).

---

## File Change Summary

### Backend Files Modified
| File | Type | Changes |
|------|------|---------|
| `internal/escrow/model.go` | MODIFY | Add `MessageType`, `EventType`, `Metadata` to `Message` |
| `internal/escrow/service.go` | MODIFY | Add `emitSystemMessage` helper + calls in 12 methods |
| `internal/escrow/invite.go` | MODIFY | Add calls in `acceptEscrowRecord`, `rejectEscrowRecord` |
| `internal/escrow/worker.go` | MODIFY | Add calls in `autoReleaseEscrow`, `expireEscrow` |
| `internal/escrow/product_details.go` | MODIFY | Add call in product details submission |
| `internal/dispute/service.go` | MODIFY | Add `EscrowSystemMessenger` interface + 4 calls |
| `cmd/server/main.go` | MODIFY | Wire messenger dependencies |

### Frontend Files Modified/Created
| File | Type | Changes |
|------|------|---------|
| `src/lib/types.ts` | MODIFY | Add `message_type`, `event_type`, `metadata` to `Message` |
| `src/components/escrow/system-message.tsx` | NEW | System message component |
| `src/app/app/escrows/[id]/page.tsx` | MODIFY | Update chat rendering, tab label |

### Test Files Modified
| File | Type | Changes |
|------|------|---------|
| `internal/escrow/service_test.go` | MODIFY | Add system message tests |
| `internal/escrow/lifecycle_test.go` | MODIFY | Add full lifecycle message test |
