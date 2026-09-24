# Phase 8 — Dispute Resolution Module: Manual Testing Guide (Swagger UI)

This guide walks through every dispute endpoint via the **Swagger UI** at
[http://localhost:3000/swagger/](http://localhost:3000/swagger/), simulating
the full dispute lifecycle from opening to admin resolution.

> All dispute endpoints belong to the **Disputes** tag group in Swagger.

---

## Prerequisites

1. **Start the server**

   ```bash
   go run cmd/server/main.go
   ```

2. **Open Swagger UI** — navigate to [http://localhost:3000/swagger/](http://localhost:3000/swagger/) in your browser.

3. **Three user accounts** required:
   - **Buyer** — a verified user with KYC Tier 1+
   - **Seller** — a verified user
   - **Admin** — a user with `admin` or `super_admin` role

   If you don't have an admin user, promote one in the database:

   ```sql
   UPDATE users SET role_id = 2 WHERE email = 'admin@example.com';
   ```

4. **An escrow in `delivered` state** — you must progress an escrow through the full lifecycle
   before opening a dispute. Follow the [Phase 5 test guide](phase5-escrow-manual-test.md)
   Steps 1–12 to reach `delivered` status. **Do NOT approve delivery** — that would complete it.

---

## How to Authenticate in Swagger UI

1. Call `POST /api/v1/auth/login` with the user's email/password.
2. Copy the `access_token` from the response.
3. Click the **Authorize** 🔒 button (top-right of the Swagger page).
4. Enter: `Bearer <paste_token_here>` → click **Authorize**.
5. All subsequent requests in this browser session will include the JWT.

> **Switching users**: Click **Authorize** → **Logout** → re-login with
> the other user's token and authorize again.

---

## Step 1 — Open Dispute (Buyer)

| Field | Value |
|-------|-------|
| **Endpoint** | `POST /api/v1/disputes` |
| **Auth** | Buyer token |
| **Swagger tag** | Disputes |

Click **Try it out**, paste this body, and click **Execute**:

```json
{
  "escrow_id": "<PASTE_ESCROW_ID_HERE>",
  "reason": "defect_discovered",
  "description": "The item received has a major defect that was not disclosed in the listing. The screen has visible cracks on the bottom left corner."
}
```

### ✅ Verify

- Response code: **201**
- `status` = `"open"`
- `opened_at` is set
- `escrow.status` = `"disputed"` (check via `GET /api/v1/escrows/{id}`)

**Save the dispute `id`** — you'll need it for all subsequent steps.

---

## Step 2 — Get Dispute Details (Buyer)

| Field | Value |
|-------|-------|
| **Endpoint** | `GET /api/v1/disputes/{id}` |
| **Auth** | Buyer token |

Enter the dispute `id` in the path parameter → **Execute**.

### ✅ Verify

- Response code: **200**
- All fields from creation are present
- `escrow` details embedded with `buyer` and `seller` info
- `evidence` and `comments` arrays are empty

---

## Step 3 — Upload Evidence (Buyer)

| Field | Value |
|-------|-------|
| **Endpoint** | `POST /api/v1/disputes/{id}/evidence` |
| **Auth** | Buyer token |

```json
{
  "file_url": "https://s3.amazonaws.com/vescrow/evidence/screen-crack-photo.jpg",
  "file_type": "image",
  "description": "Photo showing the crack on the bottom-left corner of the screen"
}
```

### ✅ Verify

- Response code: **201**
- Evidence record returned with `file_url`, `file_type`, `description`
- `uploaded_by` matches buyer's user ID

---

## Step 4 — Upload Evidence (Seller)

🔑 **Switch to Seller** (see "How to Authenticate" above).

| Field | Value |
|-------|-------|
| **Endpoint** | `POST /api/v1/disputes/{id}/evidence` |
| **Auth** | Seller token |

```json
{
  "file_url": "https://s3.amazonaws.com/vescrow/evidence/packaging-proof.pdf",
  "file_type": "pdf",
  "description": "Proof of careful packaging — item was in perfect condition when shipped"
}
```

### ✅ Verify

- Response code: **201**
- Evidence record returned
- `uploaded_by` matches seller's user ID

---

## Step 5 — Seller Responds to Dispute

| Field | Value |
|-------|-------|
| **Endpoint** | `POST /api/v1/disputes/{id}/respond` |
| **Auth** | Seller token |

```json
{
  "response": "The item was packaged extremely well and did not have any crack when shipped. It must have occurred during delivery by the logistics company. I have uploaded packaging proof."
}
```

### ✅ Verify

- Response code: **200**
- Dispute `status` changed to `"seller_responded"`
- Seller's response appears in `comments` array with role `"seller"`

---

## Step 6 — Verify Both Parties' Evidence

| Field | Value |
|-------|-------|
| **Endpoint** | `GET /api/v1/disputes/{id}` |
| **Auth** | Buyer or Seller token |

### ✅ Verify

- `evidence` array contains 2 entries (buyer's photo + seller's PDF)
- `comments` array contains at least 1 entry (seller's response)
- Evidence items ordered by `created_at ASC`

---

## Step 7 — Admin Adds Comment

🔑 **Switch to Admin**.

| Field | Value |
|-------|-------|
| **Endpoint** | `POST /api/v1/disputes/{id}/comment` |
| **Auth** | Admin token |

```json
{
  "content": "I have reviewed the evidence from both parties. The packaging proof is clear, but the delivery damage is also evident. I will propose a fair resolution."
}
```

### ✅ Verify

- Response code: **201**
- Comment returned with `role` = `"admin"`
- `author_id` matches admin's user ID

---

## Step 8 — Admin Resolves Dispute (Partial Split)

| Field | Value |
|-------|-------|
| **Endpoint** | `PUT /api/v1/disputes/{id}/resolve` |
| **Auth** | Admin token |

```json
{
  "resolution": "partial_split",
  "buyer_amount_kobo": 250000,
  "seller_amount_kobo": 250000,
  "notes": "Both parties agreed to split the escrow 50/50 since the damage likely occurred during shipping and neither party is fully at fault."
}
```

> ⚠️ **Important**: `buyer_amount_kobo + seller_amount_kobo` must equal the escrow's `amount_kobo`.
> Adjust these values to match your escrow's total amount.

### ✅ Verify

- Response code: **200**
- Dispute `status` = `"resolved"`
- `resolution` = `"partial_split"`
- `resolved_by` = admin's user ID
- `resolved_at` is set
- `buyer_amount_kobo` and `seller_amount_kobo` reflect the split

---

## Step 9 — Verify Escrow Final State

| Field | Value |
|-------|-------|
| **Endpoint** | `GET /api/v1/escrows/{id}` |
| **Auth** | Buyer, Seller, or Admin |

### ✅ Verify

- Escrow `status` = `"partial_release"`

---

## Step 10 — Verify Wallet Balances

🔑 **Switch to Seller**.

| Field | Value |
|-------|-------|
| **Endpoint** | `GET /api/v1/wallet` |
| **Auth** | Seller token |

### ✅ Verify

- Seller wallet `balance_kobo` increased by the `seller_amount_kobo` from the split

🔑 **Switch to Buyer**, check `GET /api/v1/wallet/transactions` for the refund credit entry.

---

## Step 11 — List Disputes (User View)

| Field | Value |
|-------|-------|
| **Endpoint** | `GET /api/v1/disputes` |
| **Auth** | Buyer token |

### ✅ Verify

- Response code: **200**
- `data` array contains the dispute
- Pagination meta present

---

## Step 12 — List Disputes (Admin View with Status Filter)

🔑 **Switch to Admin**.

| Field | Value |
|-------|-------|
| **Endpoint** | `GET /api/v1/disputes?status=resolved` |
| **Auth** | Admin token |

### ✅ Verify

- Response code: **200**
- Only resolved disputes appear in the list

---

## Alternative Resolution Paths

> For each alternative, **create a new escrow**, progress it to `delivered`, and open a fresh dispute.

### Alt A — Full Refund

| Field | Value |
|-------|-------|
| **Endpoint** | `PUT /api/v1/disputes/{id}/resolve` |
| **Auth** | Admin |

```json
{
  "resolution": "full_refund",
  "notes": "Buyer's claim is valid. Full refund issued."
}
```

**Expected**:
- `200`, dispute `status` = `"resolved"`, `resolution` = `"full_refund"`
- Escrow `status` = `"refunded"`
- Buyer wallet credited with full escrow amount

### Alt B — Full Release

| Field | Value |
|-------|-------|
| **Endpoint** | `PUT /api/v1/disputes/{id}/resolve` |
| **Auth** | Admin |

```json
{
  "resolution": "full_release",
  "notes": "Seller's evidence is conclusive. Funds released to seller."
}
```

**Expected**:
- `200`, dispute `status` = `"resolved"`, `resolution` = `"full_release"`
- Escrow `status` = `"completed"`, `completed_at` set
- Seller wallet credited with `amount - platform_fee`
- Platform fee wallet credited with `platform_fee`

---

## Negative Test Cases

### N1. Dispute on Non-Delivered Escrow

Create escrow in `funded` state. Try to open dispute:

```json
{
  "escrow_id": "<FUNDED_ESCROW_ID>",
  "reason": "not_as_described",
  "description": "Test"
}
```

**Expected**: `409` — `"Dispute can only be opened on a delivered escrow"`

### N2. Seller Tries to Open Dispute

🔑 Authorize as **Seller**, try `POST /api/v1/disputes` with the seller's escrow.

**Expected**: `403` — `"Only the buyer can open a dispute"`

### N3. Non-Participant Tries to View Dispute

Register a third user, authorize with their token, call `GET /api/v1/disputes/{id}`.

**Expected**: `403` — `"Access denied"`

### N4. Non-Admin Tries to Resolve

🔑 Authorize as **Buyer**, call `PUT /api/v1/disputes/{id}/resolve`.

**Expected**: `403` — `"Admin access required"`

### N5. Partial Split with Wrong Amounts

🔑 Authorize as **Admin**, resolve with amounts that don't sum to escrow total:

```json
{
  "resolution": "partial_split",
  "buyer_amount_kobo": 100000,
  "seller_amount_kobo": 100000,
  "notes": "Bad split"
}
```

**Expected**: `400` — `"Buyer and seller amounts must equal the escrow total amount"`

### N6. Duplicate Dispute on Same Escrow

Open dispute on an escrow that already has an active (unresolved) dispute.

**Expected**: `409` — `"An active dispute already exists for this escrow"`

### N7. Comment on Resolved Dispute

After resolution, try `POST /api/v1/disputes/{id}/comment`:

```json
{
  "content": "This should fail"
}
```

**Expected**: `400` — `"Cannot comment on a resolved dispute"`

### N8. Evidence on Resolved Dispute

After resolution, try `POST /api/v1/disputes/{id}/evidence`:

```json
{
  "file_url": "https://example.com/late-evidence.jpg",
  "file_type": "image",
  "description": "This should fail"
}
```

**Expected**: `400` — `"Cannot upload evidence to a resolved dispute"`

---

## Summary: Full Happy-Path Dispute Lifecycle

```
Escrow: delivered → disputed → partial_release (or refunded / completed)
Dispute: open → seller_responded → resolved
```

| Step | Actor  | Swagger Endpoint                              | Method | Result                     |
|------|--------|-----------------------------------------------|--------|----------------------------|
| 1    | Buyer  | `/api/v1/disputes`                            | POST   | Dispute opened, escrow disputed |
| 2    | Buyer  | `/api/v1/disputes/{id}`                       | GET    | Verify details             |
| 3    | Buyer  | `/api/v1/disputes/{id}/evidence`              | POST   | Evidence uploaded          |
| 4    | Seller | `/api/v1/disputes/{id}/evidence`              | POST   | Evidence uploaded          |
| 5    | Seller | `/api/v1/disputes/{id}/respond`               | POST   | Dispute → seller_responded |
| 6    | Either | `/api/v1/disputes/{id}`                       | GET    | Verify evidence + comments |
| 7    | Admin  | `/api/v1/disputes/{id}/comment`               | POST   | Admin comment added        |
| 8    | Admin  | `/api/v1/disputes/{id}/resolve`               | PUT    | Dispute resolved + wallet  |
| 9    | Either | `/api/v1/escrows/{id}`                        | GET    | Escrow = partial_release   |
| 10   | Either | `/api/v1/wallet`                              | GET    | Verify wallet balances     |
| 11   | User   | `/api/v1/disputes`                            | GET    | List user's disputes       |
| 12   | Admin  | `/api/v1/disputes?status=resolved`            | GET    | List with status filter    |
