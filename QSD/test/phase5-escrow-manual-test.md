# Phase 5 — Escrow Transaction Module: Manual Testing Guide (Swagger UI)

This guide walks through every escrow endpoint via the **Swagger UI** at
[http://localhost:3000/swagger/](http://localhost:3000/swagger/), simulating
the full escrow lifecycle from creation to completion.

> All endpoints belong to the **Escrows** tag group in Swagger.

---

## Prerequisites

1. **Start the server**

   ```bash
   go run cmd/server/main.go
   ```

   The server loads `.env` automatically via `godotenv`.

2. **Open Swagger UI** — navigate to [http://localhost:3000/swagger/](http://localhost:3000/swagger/) in your browser.

3. **Two verified user accounts** — one acts as **Buyer**, the other as **Seller**.
   If you don't have them, use the `POST /api/v1/auth/register` endpoint in
   the **Authentication** tag on the same Swagger page.

4. **Buyer must be KYC Tier 1+** — the create-escrow endpoint enforces
   `KYCRequired(tier1)`. Promote a user in the database if needed:

   ```sql
   UPDATE users SET kyc_tier = 'tier1' WHERE email = 'buyer@example.com';
   ```

---

## How to Authenticate in Swagger UI

1. Call `POST /api/v1/auth/login` with the buyer's email/password.
2. Copy the `access_token` from the response.
3. Click the **Authorize** 🔒 button (top-right of the Swagger page).
4. Enter: `Bearer <paste_token_here>` → click **Authorize**.
5. All subsequent requests in this browser session will include the JWT.

> **Switching users**: Click **Authorize** → **Logout** → re-login with the
> other user's token and authorize again.

---

## Step 1 — Create Escrow (Buyer)

| Field | Value |
|-------|-------|
| **Endpoint** | `POST /api/v1/escrows` |
| **Auth** | Buyer token |
| **Swagger tag** | Escrows |

Click **Try it out**, paste this body, and click **Execute**:

```json
{
  "title": "Logo Design Project",
  "description": "A professional brand logo with 3 revision rounds",
  "amount_kobo": 500000,
  "currency": "NGN",
  "seller_email": "seller@example.com",
  "inspection_period_days": 7
}
```

### ✅ Verify

- Response code: **201**
- `status` = `"draft"`
- `buyer_id` matches your user ID
- `platform_fee_kobo` is calculated (e.g., 3% of 500000 = 15000)

**Save the `id` from the response** — you'll need it for all subsequent steps.

---

## Step 2 — Get Escrow Details (Buyer)

| Field | Value |
|-------|-------|
| **Endpoint** | `GET /api/v1/escrows/{id}` |
| **Auth** | Buyer token |

Enter the escrow `id` in the path parameter → **Execute**.

### ✅ Verify

- Response code: **200**
- All fields from creation are present
- Status is still `"draft"`

---

## Step 3 — Update Escrow Draft (Buyer)

| Field | Value |
|-------|-------|
| **Endpoint** | `PUT /api/v1/escrows/{id}` |
| **Auth** | Buyer token |

```json
{
  "title": "Logo Design Project (Updated)",
  "description": "Now includes brand guidelines document",
  "inspection_period_days": 10
}
```

### ✅ Verify

- Response code: **200**
- Title, description, and inspection period reflect the new values
- Status is still `"draft"`

---

## Step 4 — List Escrows (Buyer)

| Field | Value |
|-------|-------|
| **Endpoint** | `GET /api/v1/escrows` |
| **Auth** | Buyer token |

Optional query params: `page`, `per_page`, `role`, `status`.

### ✅ Verify

- Response code: **200**
- `data` array contains at least 1 entry
- Pagination meta (`page`, `per_page`, `total`) is present

---

## Step 5 — Submit Escrow to Seller (Buyer)

| Field | Value |
|-------|-------|
| **Endpoint** | `PUT /api/v1/escrows/{id}/submit` |
| **Auth** | Buyer token |

No request body needed → **Execute**.

### ✅ Verify

- Response code: **200**
- Status changed to `"pending_acceptance"`

---

## Step 6 — Switch to Seller & View Escrow

🔑 **Authorize as Seller** (see "How to Authenticate" above).

| Field | Value |
|-------|-------|
| **Endpoint** | `GET /api/v1/escrows/{id}` |
| **Auth** | Seller token |

### ✅ Verify

- Response code: **200**
- Seller can see full escrow details
- Status is `"pending_acceptance"`

---

## Step 7 — Seller Accepts Escrow

| Field | Value |
|-------|-------|
| **Endpoint** | `PUT /api/v1/escrows/{id}/accept` |
| **Auth** | Seller token |

### ✅ Verify

- Response code: **200**
- Status changed to `"accepted"`

---

## Step 8 — Buyer Funds the Escrow

🔑 **Switch back to Buyer**.

| Field | Value |
|-------|-------|
| **Endpoint** | `POST /api/v1/escrows/{id}/fund` |
| **Auth** | Buyer token |

### ✅ Verify

- Response code: **200**
- Status changed to `"funded"`
- `funded_at` timestamp is present

---

## Step 9 — Send Chat Message (Buyer)

| Field | Value |
|-------|-------|
| **Endpoint** | `POST /api/v1/escrows/{id}/messages` |
| **Auth** | Buyer token |

```json
{
  "content": "Hi! I have funded the escrow. Please start working on the logo."
}
```

### ✅ Verify

- Response code: **201**
- Message ID returned
- `sender_id` matches the buyer

---

## Step 10 — Send Chat Message (Seller)

🔑 **Switch to Seller**.

| Field | Value |
|-------|-------|
| **Endpoint** | `POST /api/v1/escrows/{id}/messages` |
| **Auth** | Seller token |

```json
{
  "content": "Got it! I will start right away and deliver within 3 days."
}
```

### ✅ Verify

- Response code: **201**

---

## Step 11 — Get Chat Messages

| Field | Value |
|-------|-------|
| **Endpoint** | `GET /api/v1/escrows/{id}/messages` |
| **Auth** | Buyer or Seller |

### ✅ Verify

- Response code: **200**
- Array of 2 messages in chronological order
- Each message has `sender` details

---

## Step 12 — Seller Marks as Delivered

🔑 **Ensure you're authorized as Seller**.

| Field | Value |
|-------|-------|
| **Endpoint** | `PUT /api/v1/escrows/{id}/deliver` |
| **Auth** | Seller token |

```json
{
  "delivery_proof_url": "https://drive.google.com/file/logo-final-v2.pdf"
}
```

### ✅ Verify

- Response code: **200**
- Status changed to `"delivered"`
- `delivered_at` timestamp is present

---

## Step 13 — Buyer Approves Delivery (Happy Path → Completed)

🔑 **Switch to Buyer**.

| Field | Value |
|-------|-------|
| **Endpoint** | `PUT /api/v1/escrows/{id}/approve-delivery` |
| **Auth** | Buyer token |

### ✅ Verify

- Response code: **200**
- Status changed to `"completed"`
- `completed_at` timestamp is present

---

## Step 14 — Verify Final State

| Field | Value |
|-------|-------|
| **Endpoint** | `GET /api/v1/escrows/{id}` |
| **Auth** | Buyer or Seller |

### ✅ Verify

- `status` = `"completed"`
- `funded_at`, `delivered_at`, `completed_at` all set

---

## Alternative Paths

> For each alternative, **create a new escrow** and progress it to the
> required state before testing.

### Alt A — Seller Rejects

Progress: `draft` → `pending_acceptance`.

| Endpoint | `PUT /api/v1/escrows/{id}/reject` |
|----------|-----------------------------------|
| Auth | Seller |
| Expected | `200`, status = `"rejected"` |

### Alt B — Buyer Disputes

Progress: `draft` → `pending_acceptance` → `accepted` → `funded` → `delivered`.

| Endpoint | `PUT /api/v1/escrows/{id}/dispute` |
|----------|------------------------------------|
| Auth | Buyer |
| Expected | `200`, status = `"disputed"` |

### Alt C — Cancel Escrow

Escrow must be in `draft` state.

| Endpoint | `DELETE /api/v1/escrows/{id}` |
|----------|-------------------------------|
| Auth | Buyer |
| Expected | `200`, status = `"cancelled"` |

### Alt D — Extend Inspection Period

Escrow must be in `delivered` state.

| Endpoint | `PUT /api/v1/escrows/{id}/extend-inspection` |
|----------|----------------------------------------------|
| Auth | Buyer |
| Body | `{"additional_days": 5}` |
| Expected | `200`, inspection period extended |

---

## Milestone Endpoints

### Create Escrow with Milestones

Use `POST /api/v1/escrows` with this body:

```json
{
  "title": "Website Redesign (Milestoned)",
  "description": "Full website redesign with milestones",
  "amount_kobo": 1000000,
  "currency": "NGN",
  "seller_email": "seller@example.com",
  "inspection_period_days": 7,
  "milestones": [
    {
      "title": "Wireframes",
      "description": "Low-fidelity wireframes for all pages",
      "amount_kobo": 300000,
      "sequence_order": 1
    },
    {
      "title": "UI Design",
      "description": "High-fidelity mockups",
      "amount_kobo": 400000,
      "sequence_order": 2
    },
    {
      "title": "Development",
      "description": "Final coded website",
      "amount_kobo": 300000,
      "sequence_order": 3
    }
  ]
}
```

### List Milestones

| Endpoint | `GET /api/v1/escrows/{id}/milestones` |
|----------|---------------------------------------|
| Auth | Buyer or Seller |

### Seller Completes Milestone

| Endpoint | `PUT /api/v1/escrows/{id}/milestones/{mid}/complete` |
|----------|------------------------------------------------------|
| Auth | Seller |

### Buyer Approves Milestone

| Endpoint | `PUT /api/v1/escrows/{id}/milestones/{mid}/approve` |
|----------|-----------------------------------------------------|
| Auth | Buyer |

---

## Negative Test Cases

### N1. Unauthenticated Request

Click **Authorize** → **Logout**, then call `GET /api/v1/escrows`.

**Expected**: `401` — `"Missing authorization token"`.

### N2. Third Party Cannot View Escrow

Register a third user, authorize with their token, call `GET /api/v1/escrows/{id}`.

**Expected**: `403` — `"Access denied"`.

### N3. No KYC → Cannot Create Escrow

Set a user's `kyc_tier` to `"none"`, authorize, call `POST /api/v1/escrows`.

**Expected**: `403` — `"KYC Tier 1 required for this operation"`.

### N4. Self-Dealing Blocked

Authorize as buyer, create escrow with `seller_email` = buyer's own email.

**Expected**: `400` — `"Cannot create escrow with yourself"`.

### N5. Invalid State Transition

Try `POST /api/v1/escrows/{id}/fund` on a `draft`-state escrow.

**Expected**: `400` — state transition error.

### N6. Wrong Role Acting

Authorize as **Seller**, call `PUT /api/v1/escrows/{id}/approve-delivery`.

**Expected**: `403` — `"Only the buyer can approve delivery"`.

---

## Summary: Full Happy-Path State Sequence

```
draft → pending_acceptance → accepted → funded → (in_progress) → delivered → completed
```

| Step | Actor  | Swagger Endpoint                              | Method | Result Status        |
|------|--------|-----------------------------------------------|--------|----------------------|
| 1    | Buyer  | `/api/v1/escrows`                             | POST   | `draft`              |
| 2    | Buyer  | `/api/v1/escrows/{id}`                        | GET    | (verify details)     |
| 3    | Buyer  | `/api/v1/escrows/{id}`                        | PUT    | `draft` (updated)    |
| 4    | Buyer  | `/api/v1/escrows`                             | GET    | (list)               |
| 5    | Buyer  | `/api/v1/escrows/{id}/submit`                 | PUT    | `pending_acceptance` |
| 6    | Seller | `/api/v1/escrows/{id}`                        | GET    | (verify details)     |
| 7    | Seller | `/api/v1/escrows/{id}/accept`                 | PUT    | `accepted`           |
| 8    | Buyer  | `/api/v1/escrows/{id}/fund`                   | POST   | `funded`             |
| 9    | Buyer  | `/api/v1/escrows/{id}/messages`               | POST   | (send message)       |
| 10   | Seller | `/api/v1/escrows/{id}/messages`               | POST   | (send message)       |
| 11   | Buyer  | `/api/v1/escrows/{id}/messages`               | GET    | (list messages)      |
| 12   | Seller | `/api/v1/escrows/{id}/deliver`                | PUT    | `delivered`          |
| 13   | Buyer  | `/api/v1/escrows/{id}/approve-delivery`       | PUT    | `completed`          |
| 14   | Either | `/api/v1/escrows/{id}`                        | GET    | (verify final state) |
