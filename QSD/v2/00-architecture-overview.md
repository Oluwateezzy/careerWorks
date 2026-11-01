# Vescrow V2 — Architecture Overview

## Executive Summary

This document defines the architectural changes required to implement the Vescrow V2 feature set. The changes span **frontend (Next.js/TypeScript)**, **backend (Go/Fiber/GORM)**, and **state machine** layers.

---

## Current Architecture Snapshot

```
┌─────────────────────────────────────────────────────────────────┐
│                        FRONTEND (Next.js)                       │
│  src/app/app/  → Pages (escrows, wallet, kyc, settings, etc.)  │
│  src/components/ → UI (escrow, fintech, layout, wallet, kyc)   │
│  src/stores/   → Zustand (auth, escrow, wallet, kyc, notif)    │
│  src/lib/      → API clients, types, constants, themes (100)   │
└────────────────────────────┬────────────────────────────────────┘
                             │ REST API
┌────────────────────────────▼────────────────────────────────────┐
│                       BACKEND (Go/Fiber)                        │
│  internal/escrow/  → State machine, service, handlers, worker  │
│  internal/wallet/  → Ledger, service, handlers                 │
│  internal/trust/   → Rating system, block/appeal               │
│  internal/payment/ → Paystack integration                      │
│  internal/auth/    → JWT, registration, OTP                    │
│  internal/notification/ → Email/push notifications             │
└─────────────────────────────────────────────────────────────────┘
```

---

## V2 Change Categories

### 1. Escrow Flow Redesign
- **Fund-on-Create**: Buyer funds escrow immediately during creation (single atomic step)
- **Remove Confirm Delivery Received**: Eliminate the `delivered_awaiting_verification` intermediate state
- **Verify = Approve + Release**: The "Verify Product Condition & Inspection" modal now forces approval and fund release in one step
- **Completed Transaction Lock**: Completed escrow detail pages show a read-only summary with no further actions

### 2. Rating Penalty for Non-Approval
- New trust event `EventBuyerVerificationTimeout` penalizing buyers who don't approve/release after verification
- Backend worker enhancement to record negative trust events on inspection period expiry

### 3. Wallet → Ledger Rename
- All frontend references to "Wallet" renamed to "Ledger"
- Route paths remain unchanged to avoid link breakage, but display labels change everywhere

### 4. Escrow Guidelines System
- Global escrow guidelines document that users must agree to before creating/accepting escrows
- Persistent access point in sidebar, modals, and escrow detail pages

### 5. Theme System Simplification
- Remove the 100-theme variation system entirely
- Remove `themes.ts`, `theme-switcher.tsx`, and all related imports

### 6. Sidebar Cleanup
- Remove KYC & Limits, Notifications, and Settings **from sidebar navigation only**
- Pages remain accessible via the navbar/topbar to avoid redundancy — no pages or code deleted

---

## Revised Escrow State Machine

### Current Flow (V1)
```
Draft → Pending Acceptance → Accepted → Funded → Preparing Shipment
→ Shipped → Delivered (Awaiting Verification) → Delivery Verified → Completed
```

### New Flow (V2)
```
Draft → Pending Acceptance → Accepted+Funded (atomic) → Preparing Shipment
→ Shipped → Delivery Verified (+ auto-release) → Completed
```

### State Machine Changes

| Transition | V1 | V2 |
|---|---|---|
| Create escrow | Creates in `pending_acceptance` | Creates in `pending_acceptance` AND triggers payment flow |
| Seller accepts | → `accepted` (buyer must fund separately) | → `accepted` (already funded) |
| `shipped` → next | → `delivered_awaiting_verification` (buyer confirms receipt) | → `delivery_verified` (buyer verifies + approves + releases) |
| `delivered_awaiting_verification` | → `delivery_verified` (separate step) | **REMOVED** |
| `delivery_verified` → next | → `completed` (buyer manually approves) | → `completed` (auto on verify) |

---

## File Impact Matrix

| File / Module | Change Type | Description |
|---|---|---|
| `src/app/app/escrows/create/page.tsx` | **Major** | Add payment gateway redirect after escrow creation |
| `src/app/app/escrows/[id]/page.tsx` | **Major** | Remove confirm-delivery modal, merge verify+approve, add completed summary, add guidelines link |
| `src/stores/escrow-store.ts` | **Moderate** | Remove `confirmDelivery`, merge `verifyDelivery` + `approveDelivery`, add `createAndFundEscrow` |
| `src/lib/types.ts` | **Moderate** | Remove `ConfirmDeliveryRequest`, update `EscrowStatus` enum |
| `src/lib/constants.ts` | **Minor** | Remove `delivered_awaiting_verification` from status config |
| `src/lib/themes.ts` | **Delete** | Remove 100-theme system |
| `src/components/shared/theme-switcher.tsx` | **Delete** | Remove theme switcher component |
| `src/components/layout/app-sidebar.tsx` | **Moderate** | Remove KYC, Notifications, Settings nav items (pages stay via navbar); add Guidelines link |
| `src/components/layout/mobile-bottom-nav.tsx` | **Moderate** | Remove KYC, Notifications, Settings from "More" menu (pages stay via navbar) |
| `src/app/app/wallet/page.tsx` | **Minor** | Rename "Wallet" → "Ledger" labels |
| `src/components/kyc/` | **(Empty)** | Already empty — leave as-is |
| `internal/escrow/state_machine.go` | **Moderate** | Remove `delivered_awaiting_verification` transitions |
| `internal/escrow/service.go` | **Major** | Remove `ConfirmDelivery`, merge verify+release in `VerifyDelivery` |
| `internal/escrow/handler.go` | **Major** | Remove confirm-delivery endpoint, update verify endpoint |
| `internal/escrow/enum.go` | **Minor** | Keep status for backward compat but mark deprecated |
| `internal/escrow/worker.go` | **Moderate** | Add trust penalty on auto-release |
| `internal/trust/enum.go` | **Minor** | Add `EventBuyerVerificationTimeout` event type |
| `internal/trust/service.go` | **Minor** | Add delta for new event type |

---

## New Component: Escrow Guidelines

A global guidelines modal/page accessible from:
1. Sidebar permanent link
2. Create escrow flow (must agree before proceeding)
3. Accept escrow flow (must agree before accepting)
4. Escrow detail page (info button)
5. Footer link on all app pages
