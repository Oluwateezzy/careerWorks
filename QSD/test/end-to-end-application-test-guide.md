# Vescrow — End-to-End Application Test Guide

**Version**: 1.0  
**Last Updated**: August 2026  
**Scope**: Full application flow — registration through completion for **buyer**, **seller**, and **admin** roles  
**Covers**: UI (Next.js) + API validation criteria at each step

**Related docs**

| Doc | Use for |
|-----|---------|
| [Frontend manual test guide](../../vescrow_frontend/docs/manual-test-guide.md) | UI-specific screenshots and DevTools checks |
| [New workflow backend guide](./new-workflow-backend-manual-test.md) | Swagger/curl deep dives |
| [Phase 5 escrow guide](./phase5-escrow-manual-test.md) | Legacy escrow API details |
| [Phase 8 dispute guide](./phase8-dispute-manual-test.md) | Dispute API details |
| [State machine](../state-machine-and-transaction-flow.md) | Valid status transitions |
| [API specification](../api-specification.md) | Endpoint reference |

---

## How to use this guide

Each test case follows this structure:

| Column | Meaning |
|--------|---------|
| **Step** | Ordered action (UI route or API call) |
| **Actor** | Who performs it (`Buyer`, `Seller`, `Admin`, `System`) |
| **Validation criteria** | Pass/fail checks — UI behaviour **and** backend state |
| **API / DB check** | Optional curl or SQL to confirm when debugging |

Mark each criterion `[x]` when passed. Do not proceed to dependent steps if a **blocking** criterion fails.

---

## Part 0 — Environment & test setup

### 0.1 Start services

**Terminal 1 — Backend**

```bash
cd vescrow_backend
cp .env.example .env   # first time only
go run cmd/server/main.go
```

**Terminal 2 — Frontend**

```bash
cd vescrow_frontend
npm install            # first time only
npm run dev            # http://localhost:3001
```

**Validation criteria**

- [ ] `curl http://localhost:3000/api/v1/health` → `{"status":"healthy",...}`
- [ ] Frontend loads at `http://localhost:3001`
- [ ] Backend logs show `[Boot] Vescrow server ready`
- [ ] PostgreSQL and Redis connected (no fatal errors on startup)

**Required `.env` (backend)**

```env
APP_BASE_URL=http://localhost:3001
ENVIRONMENT=development
CORS_ORIGINS=http://localhost:3001
PAYMENT_PROVIDER=paystack
PAYSTACK_SECRET_KEY=sk_test_...
PAYSTACK_PUBLIC_KEY=pk_test_...
PAYSTACK_WEBHOOK_SECRET=...
INCOMING_FEE_PERCENT=3
OUTGOING_FEE_PERCENT=3
```

**Frontend** (`.env.local` optional):

```env
NEXT_PUBLIC_API_URL=http://localhost:3000/api/v1
```

Or use the Next.js proxy (`NEXT_PUBLIC_API_URL=/api/v1`) with `API_ORIGIN=http://localhost:3000` in `next.config.ts`.

---

### 0.2 Browser & tooling setup

| Tool | Purpose |
|------|---------|
| Chrome + Firefox | Cross-browser smoke |
| Two profiles (normal + incognito) | Simulate buyer and seller simultaneously |
| DevTools → Network | Inspect API status codes and payloads |
| DevTools → Application → Cookies | Confirm HttpOnly `access_token` / `refresh_token` |
| DevTools → Application → Local Storage | Confirm `vescrow_device_fp` UUID |
| Swagger UI | `http://localhost:3000/swagger/` (dev only) |
| Paystack test card | `4084084084084081`, CVV `408`, PIN `0000`, any future expiry |

**Validation criteria**

- [ ] Swagger accessible in development
- [ ] OTP codes appear in backend terminal logs as `[DEV OTP] key=... code=...`

---

### 0.3 Test personas

Register four accounts through the UI or API. Use distinct emails and phones.

| Persona | Email | Password | Role in tests |
|---------|-------|----------|---------------|
| **Seller** | `seller@test.com` | `TestPass123!` | Creates listings, delivers goods |
| **Buyer A** | `buyer-a@test.com` | `TestPass123!` | Primary buyer, pays escrows |
| **Buyer B** | `buyer-b@test.com` | `TestPass123!` | Race-condition / sold-out tests |
| **Admin** | `admin@test.com` | `TestPass123!` | KYC review, disputes, appeals |

After registration, promote KYC and admin (required before money flows):

```sql
-- KYC gate: verified or approved required for listings, claims, withdrawals
UPDATE users SET kyc_status = 'verified', email_verified = true, phone_verified = true
WHERE email IN ('seller@test.com', 'buyer-a@test.com', 'buyer-b@test.com');

UPDATE users SET role_id = (SELECT id FROM roles WHERE name = 'admin')
WHERE email = 'admin@test.com';
```

**Validation criteria**

- [ ] Four users exist in `users` table
- [ ] Seller and buyers have `kyc_status = 'verified'`
- [ ] Admin has `role_id` pointing to `admin` role

---

## Part 1 — Registration & authentication

### 1.1 Register new user (Seller)

| Step | Action | Actor |
|------|--------|-------|
| 1 | Open `/auth/register` | Seller |
| 2 | Fill name, email (`seller@test.com`), phone (`+2348012345678`), password | Seller |
| 3 | Submit form | Seller |

**Validation criteria**

- [ ] **UI**: Success message or redirect to email verification
- [ ] **API**: `POST /api/v1/auth/register` → **201**
- [ ] **DB**: Row in `users` with `email_verified = false`, `kyc_status = 'none'`
- [ ] **DB**: No plaintext password stored (`password_hash` is bytea)

Repeat for Buyer A, Buyer B, and Admin (use different phone numbers).

---

### 1.2 Verify email (OTP)

| Step | Action | Actor |
|------|--------|-------|
| 1 | Open `/auth/verify-email` | Seller |
| 2 | Enter email and 6-digit OTP | Seller |
| 3 | Submit | Seller |

**Validation criteria**

- [ ] **Backend log**: `[DEV OTP] key=otp:email:...` shows the code
- [ ] **UI**: Success toast; redirect to login or phone verification
- [ ] **API**: `POST /api/v1/auth/verify-email` → **200**
- [ ] **DB**: `users.email_verified = true`

Repeat for all personas.

---

### 1.3 Verify phone (OTP)

| Step | Action | Actor |
|------|--------|-------|
| 1 | Open `/auth/verify-phone` | Seller |
| 2 | Enter phone and OTP from backend log | Seller |
| 3 | Submit | Seller |

**Validation criteria**

- [ ] **API**: `POST /api/v1/auth/verify-phone` → **200**
- [ ] **DB**: `users.phone_verified = true`

---

### 1.4 Login

| Step | Action | Actor |
|------|--------|-------|
| 1 | Open `/auth/login` | Seller |
| 2 | Enter email + password | Seller |
| 3 | Submit | Seller |

**Validation criteria**

- [ ] **UI**: Redirect to `/app/overview`
- [ ] **API**: `POST /api/v1/auth/login` → **200** with `{ access_token, authenticated: true }`
- [ ] **Cookies**: HttpOnly `access_token` and `refresh_token` set
- [ ] **Network**: Subsequent requests include `X-Device-FP` header
- [ ] **Local Storage**: `vescrow_device_fp` UUID created and stable across refresh
- [ ] **DB**: Audit log entry for login (if audit enabled on login)

---

### 1.5 Session persistence & refresh

| Step | Action | Actor |
|------|--------|-------|
| 1 | Login as Seller | Seller |
| 2 | Refresh browser | Seller |
| 3 | Navigate protected route `/app/escrows` | Seller |
| 4 | Call `GET /api/v1/auth/session` (Swagger or Network tab) | Seller |

**Validation criteria**

- [ ] **UI**: Still authenticated after refresh
- [ ] **API**: `GET /auth/session` → **200** with user profile
- [ ] **API**: After access token expires (~15 min), `POST /auth/refresh` succeeds using cookie

---

### 1.6 Logout

| Step | Action | Actor |
|------|--------|-------|
| 1 | Click logout in app settings or sidebar | Seller |
| 2 | Try `/app/overview` | Seller |

**Validation criteria**

- [ ] **UI**: Redirect to `/auth/login`
- [ ] **API**: `POST /api/v1/auth/logout` → **200**
- [ ] **Cookies**: `access_token` and `refresh_token` cleared
- [ ] **UI**: Protected routes redirect to login

---

### 1.7 Password reset flow

| Step | Action | Actor |
|------|--------|-------|
| 1 | Open `/auth/forgot-password` | Buyer A |
| 2 | Enter email, submit | Buyer A |
| 3 | Copy OTP from backend log | Buyer A |
| 4 | Open `/auth/reset-password`, enter OTP + new password | Buyer A |
| 5 | Login with new password | Buyer A |

**Validation criteria**

- [ ] **API**: `POST /auth/forgot-password` → **200** (always, no email enumeration)
- [ ] **API**: `POST /auth/reset-password` → **200**
- [ ] **UI**: Login succeeds with new password only
- [ ] Reset password back to `TestPass123!` for remaining tests

---

### 1.8 Change password (authenticated)

| Step | Action | Actor |
|------|--------|-------|
| 1 | Login as Seller | Seller |
| 2 | Open `/app/settings` | Seller |
| 3 | Change password (current + new) | Seller |

**Validation criteria**

- [ ] **API**: `POST /api/v1/auth/change-password` → **200**
- [ ] **UI**: Success feedback
- [ ] Old password no longer works; new password works

---

## Part 2 — KYC verification

> KYC must be `verified` or `approved` before creating escrows, listings, or withdrawing. For full UI KYC testing, complete the wizard; for faster E2E, use the SQL in Part 0.3 and validate the gate separately.

### 2.1 KYC status page

| Step | Action | Actor |
|------|--------|-------|
| 1 | Login as Buyer A (before SQL shortcut, or after resetting `kyc_status = 'none'`) | Buyer A |
| 2 | Open `/app/kyc/status` | Buyer A |

**Validation criteria**

- [ ] **UI**: Shows current KYC status (`none`, `pending`, `verified`, etc.)
- [ ] **API**: `GET /api/v1/kyc/status` → **200**

---

### 2.2 BVN verification

| Step | Action | Actor |
|------|--------|-------|
| 1 | Open `/app/kyc/verify` | Buyer A |
| 2 | Submit BVN (test value per KYC provider / mock) | Buyer A |

**Validation criteria**

- [ ] **API**: `POST /api/v1/kyc/verify-bvn` → **200**
- [ ] **DB**: `users.kyc_status` updated (typically `verified` or `pending`)
- [ ] **DB**: `bvn_hash` stored encrypted (not plaintext)

---

### 2.3 Document submission

| Step | Action | Actor |
|------|--------|-------|
| 1 | Open `/app/kyc/documents` | Buyer A |
| 2 | Upload ID document + selfie (test PNG/PDF) | Buyer A |

**Validation criteria**

- [ ] **API**: `POST /api/v1/kyc/submit-document` → **200/201**
- [ ] **DB**: Row in `kyc_documents` with `status = 'pending'`

---

### 2.4 KYC gate enforcement

| Step | Action | Actor |
|------|--------|-------|
| 1 | Set user `kyc_status = 'none'` in DB | System |
| 2 | Login as that user | Buyer A |
| 3 | Try create listing or legacy escrow | Buyer A |

**Validation criteria**

- [ ] **API**: `POST /api/v1/listings` → **403**
- [ ] **API**: `POST /api/v1/escrows` → **403**
- [ ] **UI**: Error message directing user to complete KYC
- [ ] Restore `kyc_status = 'verified'` before Part 3

---

### 2.5 Admin KYC approval (optional full path)

| Step | Action | Actor |
|------|--------|-------|
| 1 | Login as Admin (Swagger or future admin UI) | Admin |
| 2 | `GET /api/v1/admin/kyc/submissions` | Admin |
| 3 | `PUT /api/v1/admin/kyc/submissions/{id}/approve` | Admin |

**Validation criteria**

- [ ] **API**: Approve → **200**; user `kyc_status = 'approved'`
- [ ] User can now create listings and claim

---

## Part 3 — Seller path: shareable listing workflow

Primary seller journey — create a product listing, share link, fulfil orders.

### 3.1 Create shareable listing

| Step | Action | Actor |
|------|--------|-------|
| 1 | Login as **Seller** | Seller |
| 2 | Open `/app/escrows/create` | Seller |
| 3 | Select **I'm the Seller** | Seller |
| 4 | Enter title: `Test Widget`, amount: `₦100,000`, quantity: `3`, inspection: `3` days | Seller |
| 5 | Enter description (≥ 50 characters) describing the product | Seller |
| 6 | Upload **1–5 product images** (JPEG/PNG/WebP, max 5 MB each) | Seller |
| 7 | Review fee breakdown | Seller |
| 8 | Submit **Create Share Listing** | Seller |

**Validation criteria**

- [ ] **UI**: Description counter shows minimum 50 characters; submit blocked below threshold
- [ ] **UI**: At least one image required before continuing to review
- [ ] **API**: `POST /api/v1/listings/images` → **201** with `{ url }` per upload

- [ ] **UI**: Success toast; redirect to `/app/listings`
- [ ] **UI**: Share link copied (format: `http://localhost:3001/e/{token}`)
- [ ] **UI fee display** (₦100,000 item):

  | Field | Expected |
  |-------|----------|
  | Item price | ₦100,000.00 |
  | Incoming fee (3%) | ₦3,000.00 |
  | Buyer pays | ₦103,000.00 |
  | Seller receives (note) | ₦97,000.00 after 3% outgoing |

- [ ] **API**: `POST /api/v1/listings` → **201**
- [ ] **API response**: `share_token`, `quantity_available = 3`, `status = active`, `images` array populated
- [ ] **DB**: Row in `escrow_listings`; `quantity_sold = 0`, `quantity_reserved = 0`, `images` JSONB non-empty

**Save**: `SHARE_TOKEN`, `LISTING_ID`

---

### 3.2 Manage listings

| Step | Action | Actor |
|------|--------|-------|
| 1 | Open `/app/listings` | Seller |
| 2 | Click **Copy Link** | Seller |
| 3 | Click **Preview** | Seller |

**Validation criteria**

- [ ] **UI**: Listing shows `3/3 left` (or equivalent available count)
- [ ] **UI**: Copy produces valid URL
- [ ] **UI**: Preview opens `/e/{token}` in new tab
- [ ] **API**: `GET /api/v1/listings` → **200** includes listing

---

### 3.3 Public listing page (anonymous)

| Step | Action | Actor |
|------|--------|-------|
| 1 | Open `/e/{SHARE_TOKEN}` **without logging in** | Anonymous |

**Validation criteria**

- [ ] **UI**: Full product detail page — image gallery, description, seller name, fee breakdown
- [ ] **UI**: **Proceed to Secure Payment** visible; clicking while logged out redirects to login with `redirect=/e/{token}&action=pay`
- [ ] **UI**: No app sidebar (public layout)
- [ ] **API**: `GET /api/v1/listings/{token}/public` → **200** with `images[]`, `incoming_fee_kobo`, `buyer_total_kobo`
- [ ] **API**: Response does **not** expose seller email or internal UUIDs

---

### 3.4 Auth-gated payment (anonymous → login → pay)

| Step | Action | Actor |
|------|--------|-------|
| 1 | Open `/e/{SHARE_TOKEN}` **without logging in** | Anonymous |
| 2 | Click **Proceed to Secure Payment** | Anonymous |
| 3 | Sign in at `/auth/login?redirect=...&action=pay` | Buyer A |
| 4 | Confirm payment modal on return to listing | Buyer A |

**Validation criteria**

- [ ] **UI**: Unauthenticated click redirects to login (not a generic API error)
- [ ] **UI**: After login, user returns to listing; confirmation modal appears (no surprise auto-redirect)
- [ ] **UI**: User without KYC redirected to `/app/kyc/verify?redirect=...`
- [ ] **API**: Claim and payment only occur after auth + KYC + user confirmation

---

## Part 4 — Buyer path: purchase via share link

### 4.1 Claim listing slot & pay

| Step | Action | Actor |
|------|--------|-------|
| 1 | Login as **Buyer A** (incognito profile) | Buyer A |
| 2 | Open seller's share link `/e/{token}` | Buyer A |
| 3 | Click **Proceed to Secure Payment** → confirm modal → Paystack | Buyer A |
| 4 | Complete Paystack checkout (test card) | Buyer A |
| 5 | Land on callback / escrow detail | Buyer A |

**Validation criteria**

- [ ] **API**: `POST /api/v1/listings/{token}/claim` → **201** with `escrow_id`
- [ ] **API**: `POST /api/v1/payments/initialize` → **200** with `authorization_url`
- [ ] **API**: Request includes `Idempotency-Key` header
- [ ] **Paystack**: Charge amount = **₦103,000** (principal + 3%), not ₦100,000
- [ ] **After webhook** (check server logs within ~30s):

  | Check | Expected |
  |-------|----------|
  | Escrow status | `funded` |
  | `escrow_payments.status` | `confirmed` |
  | `payment_reservations.status` | `confirmed` |
  | Listing `quantity_sold` | `1` |
  | Listing `quantity_available` | `2` |

- [ ] **UI (Buyer)**: Escrow appears at `/app/escrows` with status **Funded**
- [ ] **UI (Seller)**: `/app/listings` shows `2/3 left`
- [ ] **UI**: Notification at `/app/notifications` (if inbox wired)

**Save**: `ESCROW_ID`, `PAYMENT_REF`

**API fallback** (if webhook delayed locally):

```bash
curl -H "Authorization: Bearer $BUYER_A_TOKEN" \
  "$API/payments/verify/$PAYMENT_REF"
```

---

### 4.2 Verify escrow detail (buyer view)

| Step | Action | Actor |
|------|--------|-------|
| 1 | Open `/app/escrows/{ESCROW_ID}` | Buyer A |

**Validation criteria**

- [ ] **UI**: Status badge **Funded**
- [ ] **UI**: Amount, fees, buyer/seller names correct
- [ ] **UI**: Buyer cannot see "Mark Delivered" (seller action)
- [ ] **API**: `GET /api/v1/escrows/{id}` → **200**

---

### 4.3 Sold-out behaviour (quantity = 1 listing)

Create a **separate** listing with `quantity_total = 1`, then:

| Step | Action | Actor |
|------|--------|-------|
| 1 | Buyer A claims and pays successfully | Buyer A |
| 2 | Buyer B opens same link | Buyer B |
| 3 | Buyer B clicks Pay Now | Buyer B |

**Validation criteria**

- [ ] **UI**: **Sold Out** badge OR disabled Pay Now button
- [ ] **API**: Claim after sold out → **409** `"sold_out"`
- [ ] **DB**: Listing `status = sold_out` or `quantity_available = 0`

---

### 4.4 Race condition (quantity = 1, two payers)

| Step | Action | Actor |
|------|--------|-------|
| 1 | Buyer A and B both claim before either pays | Both |
| 2 | Buyer A completes Paystack first | Buyer A |
| 3 | Buyer B completes Paystack second (if they pay) | Buyer B |

**Validation criteria**

- [ ] Exactly **one** escrow reaches `funded`
- [ ] Loser's escrow cancelled or payment refunded
- [ ] Listing `quantity_sold = 1`

---

### 4.5 Reservation timeout (no payment)

| Step | Action | Actor |
|------|--------|-------|
| 1 | Buyer A clicks Pay Now | Buyer A |
| 2 | Close Paystack tab **without paying** | Buyer A |
| 3 | Wait **16+ minutes** | System |
| 4 | Buyer B opens link and claims | Buyer B |

**Validation criteria**

- [ ] **DB**: Original reservation `status = expired`
- [ ] **DB**: Listing `quantity_reserved` decremented
- [ ] Buyer B can claim the slot again (if not sold out)

---

## Part 5 — Buyer path: legacy direct escrow

Buyer creates a custom escrow deal (no listing). Seller must accept with bank details.

### 5.1 Buyer creates escrow (registered seller)

| Step | Action | Actor |
|------|--------|-------|
| 1 | Login as **Buyer A** | Buyer A |
| 2 | Open `/app/escrows/create` | Buyer A |
| 3 | Select **I'm the Buyer** | Buyer A |
| 4 | Enter seller email (`seller@test.com`), title, amount `₦50,000`, milestones optional | Buyer A |
| 5 | Submit **Confirm & Create Deal** | Buyer A |

**Validation criteria**

- [ ] **API**: `POST /api/v1/escrows` → **201**; response includes `escrow`, `invite_url`, `seller_pending_registration: false`
- [ ] **DB**: `status = pending_acceptance`, `seller_id` set, `invite_token` populated, `expires_at` ~72h
- [ ] **Email**: Seller receives `escrow_seller_invite` (or `[MOCK EMAIL]` in dev)
- [ ] **UI**: Success screen shows **Copy invite link**; escrow visible at `/app/escrows`

**Save**: `LEGACY_ESCROW_ID`, `LEGACY_INVITE_URL`

---

### 5.1b Buyer creates escrow (unregistered seller)

| Step | Action | Actor |
|------|--------|-------|
| 1 | Login as **Buyer A** | Buyer A |
| 2 | Create escrow with seller email that is **not** registered (e.g. `new-seller@example.com`) | Buyer A |
| 3 | Copy invite link from success screen | Buyer A |

**Validation criteria**

- [ ] **API**: `seller_pending_registration: true`
- [ ] **DB**: `seller_id IS NULL`, `pending_seller_email` set
- [ ] **Email**: Invite sent to pending email
- [ ] **Public**: `GET /api/v1/escrows/invite/{token}/public` → **200**, `requires_registration: true`

**Save**: `PENDING_SELLER_INVITE_URL`

---

### 5.2 Seller accepts invite

Auto-submit on create replaces the old draft + submit flow. Seller accepts from invite link or in-app, then adds product details before the buyer can fund.

| Step | Action | Actor |
|------|--------|-------|
| 1 | Open `LEGACY_INVITE_URL` (or `/invite/{token}`) | Seller |
| 2 | Log in as seller if prompted | Seller |
| 3 | **Accept deal** | Seller |
| 4 | Redirected to escrow detail — upload 1–5 images + description (50+ chars) | Seller |
| 5 | **Publish product & notify buyer** | Seller |

**Validation criteria**

- [ ] **API**: `PUT /api/v1/escrows/invite/{token}/accept` → **200**; `status = awaiting_product_details`
- [ ] **API**: `PUT /api/v1/escrows/{id}/product-details` → **200**; `status = accepted`; `images` JSONB populated
- [ ] **UI (Seller)**: Product submission form on escrow detail after accept
- [ ] **Email**: Buyer receives `escrow_product_ready` when product is submitted (not on accept alone)

For unregistered seller: register at `/auth/register?email=...&redirect=/invite/{token}`, verify email, log in, then accept.

---

### 5.3 Seller accepts with bank details (in-app)

| Step | Action | Actor |
|------|--------|-------|
| 1 | Login as **Seller** | Seller |
| 2 | Open legacy escrow detail | Seller |
| 3 | Accept terms; provide bank code `058`, account `0123456789` | Seller |
| 4 | Submit product details (images + description) | Seller |

**Validation criteria**

- [ ] **API**: `PUT /api/v1/escrows/{id}/accept` → **200**; `status = awaiting_product_details`
- [ ] **API**: `PUT /api/v1/escrows/{id}/product-details` → **200**; `status = accepted`
- [ ] **DB**: `seller_bank_code`, `seller_account_number` stored (if provided on accept)
- [ ] **DB**: `disbursement_status = pending` (when bank details provided)

---

### 5.4 Buyer funds legacy escrow

| Step | Action | Actor |
|------|--------|-------|
| 1 | Login as **Buyer A** | Buyer A |
| 2 | Review product gallery and description on escrow detail | Buyer A |
| 3 | Initiate payment from escrow detail (only after seller submitted product) | Buyer A |
| 4 | Complete Paystack checkout | Buyer A |

**Validation criteria**

- [ ] **API**: `POST /api/v1/payments/initialize` returns **400** while `status = awaiting_product_details`
- [ ] **API**: Payment succeeds only when `status = accepted` and `images` present

- [ ] Same funding criteria as Part 4.1
- [ ] Escrow → `funded`

---

### 5.5 Seller rejects terms (negative path)

| Step | Action | Actor |
|------|--------|-------|
| 1 | Buyer creates + submits new escrow | Buyer A |
| 2 | Seller rejects with reason | Seller |

**Validation criteria**

- [ ] **API**: `PUT /api/v1/escrows/{id}/reject` → **200**; `status = rejected`
- [ ] **UI**: Terminal state; no payment option

---

## Part 6 — Escrow lifecycle (delivery & release)

Applies to **both** listing-based and legacy escrows once `funded`.

### 6.1 Seller marks delivered

| Step | Action | Actor |
|------|--------|-------|
| 1 | Login as **Seller** | Seller |
| 2 | Open funded escrow `/app/escrows/{id}` | Seller |
| 3 | Mark as delivered (optional proof URL) | Seller |

**Validation criteria**

- [ ] **API**: `PUT /api/v1/escrows/{id}/deliver` → **200**
- [ ] **DB**: `status = delivered`; `delivered_at` set
- [ ] **UI**: Status badge **Delivered**
- [ ] **UI (Buyer)**: Approve delivery action now visible
- [ ] Notification sent (email log or `/app/notifications`)

---

### 6.2 Buyer approves delivery

| Step | Action | Actor |
|------|--------|-------|
| 1 | Login as **Buyer A** | Buyer A |
| 2 | Open delivered escrow | Buyer A |
| 3 | Approve delivery (confirm disbursement disclaimer if shown) | Buyer A |

**Validation criteria**

- [ ] **API**: `PUT /api/v1/escrows/{id}/approve-delivery` → **200**
- [ ] **DB**: `status = completed`; `completed_at` set
- [ ] **DB ledger**: Escrow wallet debited; platform fee wallet credited
- [ ] **DB**: Seller receives `amount_kobo - outgoing_fee_kobo`
- [ ] **Paystack**: Transfer initiated to seller bank (check dashboard)
- [ ] **UI**: Status **Completed** for both parties

**Fee math check** (₦100,000 principal):

| Leg | Kobo | Naira |
|-----|------|-------|
| Buyer paid | 10,300,000 | ₦103,000 |
| Seller received | 9,700,000 | ₦97,000 |
| Platform fees | ~600,000 total | ~₦6,000 |

---

### 6.3 Extend inspection period

| Step | Action | Actor |
|------|--------|-------|
| 1 | With escrow in `delivered`, buyer requests extension | Buyer A |

**Validation criteria**

- [ ] **API**: `PUT /api/v1/escrows/{id}/extend-inspection` → **200**
- [ ] **DB**: `inspection_period_days` updated

---

### 6.4 In-escrow messaging

| Step | Action | Actor |
|------|--------|-------|
| 1 | Buyer sends message on escrow detail | Buyer A |
| 2 | Seller replies | Seller |
| 3 | View message thread | Both |

**Validation criteria**

- [ ] **API**: `POST /api/v1/escrows/{id}/messages` → **201**
- [ ] **API**: `GET /api/v1/escrows/{id}/messages` → **200** with both messages
- [ ] **UI**: Messages appear in chronological order

---

### 6.5 Milestone escrows (if configured)

| Step | Action | Actor |
|------|--------|-------|
| 1 | Create escrow with 2 milestones (50/50 split) | Buyer A |
| 2 | Fund escrow | Buyer A |
| 3 | Seller completes milestone 1 | Seller |
| 4 | Buyer approves milestone 1 | Buyer A |

**Validation criteria**

- [ ] **API**: `PUT /api/v1/escrows/{id}/milestones/{mid}/complete` → **200**
- [ ] **API**: `PUT /api/v1/escrows/{id}/milestones/{mid}/approve` → **200**
- [ ] Partial release reflected in ledger

---

## Part 7 — Dispute flow

Requires escrow in **`delivered`** status (do **not** approve delivery yet).

### 7.1 Buyer opens dispute

| Step | Action | Actor |
|------|--------|-------|
| 1 | Progress escrow to `delivered` | Seller |
| 2 | Login as **Buyer A** | Buyer A |
| 3 | Open dispute from escrow detail or API | Buyer A |

**Validation criteria**

- [ ] **API**: `POST /api/v1/disputes` → **201**; dispute `status = open`
- [ ] **DB**: Escrow `status = disputed`
- [ ] **UI**: Dispute badge on escrow detail

**Save**: `DISPUTE_ID`

---

### 7.2 Seller responds

| Step | Action | Actor |
|------|--------|-------|
| 1 | Login as **Seller** | Seller |
| 2 | Submit seller response | Seller |
| 3 | Upload evidence (optional PDF/image) | Seller |

**Validation criteria**

- [ ] **API**: `POST /api/v1/disputes/{id}/respond` → **200**
- [ ] **API**: `POST /api/v1/disputes/{id}/evidence` → **200**
- [ ] **DB**: Dispute status progresses (`seller_responded` / `under_review`)

---

### 7.3 Admin resolves dispute

Test each resolution type on **separate** escrows:

| Resolution | Expected escrow status | Funds |
|------------|------------------------|-------|
| `full_refund` | `refunded` | Buyer receives full principal |
| `full_release` | `completed` | Seller receives principal − outgoing fee |
| `partial_split` | `partial_release` | Split per admin amounts |

| Step | Action | Actor |
|------|--------|-------|
| 1 | Login as **Admin** | Admin |
| 2 | `PUT /api/v1/disputes/{id}/resolve` with resolution body | Admin |

**Validation criteria**

- [ ] **API**: → **200**; dispute `status = resolved`
- [ ] **DB**: Ledger entries match resolution
- [ ] **Trust**: Seller loses trust points on `full_refund` (`trust_events` row)

---

## Part 8 — Wallet & withdrawals (seller)

After escrow **completed**, seller may have wallet balance (if not direct-disbursed to bank).

### 8.1 View wallet

| Step | Action | Actor |
|------|--------|-------|
| 1 | Login as **Seller** | Seller |
| 2 | Open `/app/wallet` | Seller |
| 3 | Open `/app/wallet/transactions` | Seller |

**Validation criteria**

- [ ] **API**: `GET /api/v1/wallet` → **200** with balance in kobo
- [ ] **API**: `GET /api/v1/wallet/transactions` → **200** paginated ledger
- [ ] **UI**: Balance formatted as Naira (kobo ÷ 100)

---

### 8.2 Verify bank account

| Step | Action | Actor |
|------|--------|-------|
| 1 | Open `/app/wallet/withdraw` | Seller |
| 2 | Enter bank code `058`, account `0123456789` | Seller |
| 3 | Verify account name resolves | Seller |

**Validation criteria**

- [ ] **API**: `POST /api/v1/wallet/verify-account` → **200** with account name
- [ ] **API**: `GET /api/v1/wallet/banks` → **200** bank list

---

### 8.3 Request withdrawal

| Step | Action | Actor |
|------|--------|-------|
| 1 | Enter withdrawal amount (within limits) | Seller |
| 2 | Submit withdrawal | Seller |

**Validation criteria**

- [ ] **API**: `POST /api/v1/wallet/withdraw` → **200** with `Idempotency-Key`
- [ ] **DB**: Row in `withdrawals`; row in `payout_outbox` with `status = pending`
- [ ] **UI**: Success message; withdrawal appears in history
- [ ] Duplicate submit with same idempotency key → **409**

---

### 8.4 Withdrawal negative paths

| Condition | Expected |
|-----------|----------|
| `kyc_status = none` | **403** |
| Wallet `is_frozen = true` | **403** |
| Amount > daily limit | **400/422** |
| Duplicate `Idempotency-Key` | **409** |
| Trust-blocked user | **403** `ACCOUNT_BLOCKED` |

---

## Part 9 — Trust & appeals

### 9.1 View trust profile

| Step | Action | Actor |
|------|--------|-------|
| 1 | `GET /api/v1/trust/profile` | Buyer A |

**Validation criteria**

- [ ] **API**: → **200**; new user `score = 100`, `status = good`

---

### 9.2 Blocked user experience

| Step | Action | Actor |
|------|--------|-------|
| 1 | Insert block in DB (see SQL below) | System |
| 2 | Login as blocked buyer | Buyer A |
| 3 | Try Pay Now on listing | Buyer A |

```sql
UPDATE trust_profiles SET score = 15, status = 'blocked'
WHERE user_id = (SELECT id FROM users WHERE email = 'buyer-a@test.com');

INSERT INTO block_entries (id, user_id, reason, blocked_at, active)
VALUES (uuid_generate_v4(),
  (SELECT id FROM users WHERE email = 'buyer-a@test.com'),
  'E2E test block', NOW(), true);
```

**Validation criteria**

- [ ] **API**: Claim/pay → **403** `{ "code": "ACCOUNT_BLOCKED" }`
- [ ] **UI**: Error toast; redirect hint to `/app/appeals/new`

---

### 9.3 Submit appeal

| Step | Action | Actor |
|------|--------|-------|
| 1 | Open `/app/appeals/new` | Buyer A |
| 2 | Enter block ID + reason (20+ chars) | Buyer A |
| 3 | Submit | Buyer A |

**Validation criteria**

- [ ] **API**: `POST /api/v1/appeals` → **201**; `status = pending`
- [ ] **UI**: Success toast

---

### 9.4 Admin approves appeal

| Step | Action | Actor |
|------|--------|-------|
| 1 | `PUT /api/v1/admin/appeals/{id}/review` `{ "approve": true, "note": "..." }` | Admin |

**Validation criteria**

- [ ] Block deactivated; user can transact again
- [ ] Trust score reset to warning level (~50)

---

## Part 10 — Notifications & profile

### 10.1 In-app notifications

| Step | Action | Actor |
|------|--------|-------|
| 1 | Trigger event (payment, delivery) | System |
| 2 | Open `/app/notifications` | Seller |
| 3 | Mark one read; mark all read | Seller |

**Validation criteria**

- [ ] **API**: `GET /api/v1/notifications` → **200** with `unread_count`
- [ ] **API**: `PUT /api/v1/notifications/{id}/read` → **200**
- [ ] **UI**: Unread count decreases

---

### 10.2 Profile management

| Step | Action | Actor |
|------|--------|-------|
| 1 | Open `/app/settings` or profile | Seller |
| 2 | Update display name | Seller |
| 3 | `GET /api/v1/auth/profile` | Seller |

**Validation criteria**

- [ ] **API**: `PUT /api/v1/auth/profile` → **200**
- [ ] **UI**: Updated name persists after refresh
- [ ] PII masked in responses (phone, email partial mask)

---

## Part 11 — Admin operations

### 11.1 KYC queue

| Step | Action | Actor |
|------|--------|-------|
| 1 | `GET /api/v1/admin/kyc/submissions` | Admin |
| 2 | Approve or reject a pending submission | Admin |

**Validation criteria**

- [ ] **API**: List → **200**
- [ ] Approve → user `kyc_status = approved`
- [ ] Reject → document `status = rejected` with reason

---

### 11.2 Audit logs

| Step | Action | Actor |
|------|--------|-------|
| 1 | `GET /api/v1/admin/audit-logs?page=1&per_page=50` | Admin |

**Validation criteria**

- [ ] **API**: → **200** with paginated logs
- [ ] Logs contain recent escrow/payment actions from earlier parts
- [ ] Requires `view_audit` permission

---

## Part 12 — Security validations

Run after functional flows pass.

| # | Test | Steps | Pass criteria |
|---|------|-------|---------------|
| S1 | HttpOnly cookies | Login; inspect cookies in DevTools | `access_token` and `refresh_token` are HttpOnly |
| S2 | No token in localStorage | Check Application → Local Storage | Access token **not** stored (memory only) |
| S3 | Device fingerprint | Any API call in Network tab | `X-Device-FP` header present |
| S4 | Idempotency | Submit withdrawal twice with same key | Second request → **409** |
| S5 | Dev fund blocked | `POST /escrows/{id}/fund` in production env | **403** (test with `ENVIRONMENT=production`) |
| S6 | CORS | Frontend API calls from `:3001` | No CORS errors in console |
| S7 | Unauthorized access | Call `GET /escrows` without token | **401** |
| S8 | Cross-user access | Buyer A requests Buyer B's escrow by ID | **403** or **404** |
| S9 | Rate limit | 6+ rapid login attempts | **429** on auth endpoint |
| S10 | Swagger disabled | Set `ENVIRONMENT=production`, restart | `/swagger` not available |

---

## Part 13 — Master sign-off checklist

Complete all rows before demo or release candidate sign-off.

### Authentication & identity

| # | Scenario | UI | API/DB | Pass |
|---|----------|----|----|------|
| 1 | Register 4 personas | ☐ | ☐ | ☐ |
| 2 | Email + phone OTP verification | ☐ | ☐ | ☐ |
| 3 | Login / logout / session refresh | ☐ | ☐ | ☐ |
| 4 | Password reset + change | ☐ | ☐ | ☐ |
| 5 | KYC gate blocks unverified users | ☐ | ☐ | ☐ |
| 6 | Admin KYC approve/reject | ☐ | ☐ | ☐ |

### Seller flows

| # | Scenario | UI | API/DB | Pass |
|---|----------|----|----|------|
| 7 | Create shareable listing (qty > 1) | ☐ | ☐ | ☐ |
| 8 | Copy link + public preview | ☐ | ☐ | ☐ |
| 9 | Listings page shows quantity | ☐ | ☐ | ☐ |
| 10 | Accept legacy escrow + bank details | ☐ | ☐ | ☐ |
| 11 | Mark delivered | ☐ | ☐ | ☐ |
| 12 | Respond to dispute + upload evidence | ☐ | ☐ | ☐ |

### Buyer flows

| # | Scenario | UI | API/DB | Pass |
|---|----------|----|----|------|
| 13 | Pay via share link (Paystack) | ☐ | ☐ | ☐ |
| 14 | Paystack amount = price + 3% | ☐ | ☐ | ☐ |
| 15 | Create legacy escrow + submit | ☐ | ☐ | ☐ |
| 16 | Fund legacy escrow | ☐ | ☐ | ☐ |
| 17 | Approve delivery | ☐ | ☐ | ☐ |
| 18 | Open dispute on delivered escrow | ☐ | ☐ | ☐ |
| 19 | In-escrow messaging | ☐ | ☐ | ☐ |

### Marketplace edge cases

| # | Scenario | UI | API/DB | Pass |
|---|----------|----|----|------|
| 20 | qty=1 sold out after first payment | ☐ | ☐ | ☐ |
| 21 | qty=3 — three buyers succeed, fourth blocked | ☐ | ☐ | ☐ |
| 22 | Two-buyer race — one winner | ☐ | ☐ | ☐ |
| 23 | Reservation expires after 15 min | ☐ | ☐ | ☐ |

### Money & wallet

| # | Scenario | UI | API/DB | Pass |
|---|----------|----|----|------|
| 24 | Webhook funds escrow + updates listing qty | ☐ | ☐ | ☐ |
| 25 | Dual fee math correct (in + out 3%) | ☐ | ☐ | ☐ |
| 26 | Seller disbursement to bank on approve | ☐ | ☐ | ☐ |
| 27 | Wallet balance + transaction history | ☐ | ☐ | ☐ |
| 28 | Withdrawal with idempotency | ☐ | ☐ | ☐ |

### Trust & admin

| # | Scenario | UI | API/DB | Pass |
|---|----------|----|----|------|
| 29 | Blocked user → 403 + appeal path | ☐ | ☐ | ☐ |
| 30 | Appeal submit + admin approve | ☐ | ☐ | ☐ |
| 31 | Admin dispute resolution (3 types) | ☐ | ☐ | ☐ |
| 32 | Admin audit logs accessible | ☐ | ☐ | ☐ |

### Cross-cutting

| # | Scenario | UI | API/DB | Pass |
|---|----------|----|----|------|
| 33 | Notifications inbox | ☐ | ☐ | ☐ |
| 34 | Two-browser buyer/seller parallel test | ☐ | ☐ | ☐ |
| 35 | Mobile responsive (375px) | ☐ | — | ☐ |
| 36 | Security checks (Part 12) | ☐ | ☐ | ☐ |

---

## Part 14 — 15-minute demo script

Use for stakeholder walkthroughs:

```
1. [Seller]  Register/login → KYC verified
2. [Seller]  Create listing (qty=1, ₦50,000) → copy share link
3. [Buyer]   Open link (incognito) → show public page + fee breakdown
4. [Buyer]   Pay Now → Paystack test payment → funded escrow
5. [Seller]  Show listing sold out (0/1)
6. [Seller]  Mark delivered on escrow detail
7. [Buyer]   Approve delivery → completed
8. [Both]    Show notifications + final status
9. [Optional] Open dispute on a second escrow → admin resolves
10. Mention: seller paid via Paystack transfer; platform fees in ledger
```

---

## Part 15 — Troubleshooting quick reference

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| 403 on create/claim | KYC not verified | `UPDATE users SET kyc_status = 'verified'` |
| 403 ACCOUNT_BLOCKED | Trust block active | Check `block_entries`, submit appeal |
| Pay Now fails immediately | Not logged in / KYC / trust | See above |
| Paystack wrong amount | Fee not applied | Expect price + 3%; check `buyer_total_kobo` |
| Webhook not firing locally | No tunnel to localhost | Use `GET /payments/verify/{ref}` manually |
| CORS error | Origin not allowed | Set `CORS_ORIGINS=http://localhost:3001` |
| 401 after refresh | Redis down / session revoked | Restart Redis; login again |
| OTP not received | Email/SMS not configured | Read `[DEV OTP]` in backend terminal |
| Listing not found | Bad token or backend down | Verify `NEXT_PUBLIC_API_URL` |
| Duplicate withdrawal | Idempotency working | Expected **409** — use new key |

---

## Part 16 — Database validation queries

Run after each major flow to confirm state:

```sql
-- Users & KYC
SELECT email, kyc_status, email_verified, role_id FROM users
WHERE email LIKE '%@test.com';

-- Listings
SELECT title, quantity_total, quantity_sold, quantity_reserved, status, share_token
FROM escrow_listings ORDER BY created_at DESC LIMIT 5;

-- Escrows
SELECT id, status, amount_kobo, buyer_total_kobo, listing_id, funded_at, completed_at
FROM escrows ORDER BY created_at DESC LIMIT 10;

-- Payments
SELECT paystack_reference, status, amount_kobo, paid_at
FROM escrow_payments ORDER BY created_at DESC LIMIT 5;

-- Reservations
SELECT escrow_id, status, expires_at FROM payment_reservations ORDER BY created_at DESC LIMIT 5;

-- Ledger (seller wallet)
SELECT entry_type, amount_kobo, category, description, created_at
FROM ledger_entries ORDER BY created_at DESC LIMIT 10;

-- Withdrawals & outbox
SELECT w.status, w.amount_kobo, p.status AS outbox_status
FROM withdrawals w
LEFT JOIN payout_outbox p ON p.withdrawal_id = w.id
ORDER BY w.created_at DESC LIMIT 5;

-- Trust
SELECT u.email, tp.score, tp.status FROM trust_profiles tp
JOIN users u ON u.id = tp.user_id;

-- Disputes
SELECT d.status, d.resolution, e.status AS escrow_status
FROM disputes d JOIN escrows e ON e.id = d.escrow_id
ORDER BY d.created_at DESC LIMIT 5;
```

---

## Part 17 — API curl helpers

```bash
export API=http://localhost:3000/api/v1

# Login
curl -s -X POST "$API/auth/login" \
  -H "Content-Type: application/json" \
  -d '{"email":"seller@test.com","password":"TestPass123!"}' | jq .

# Health
curl -s "$API/health" | jq .

# Public listing
curl -s "$API/listings/$SHARE_TOKEN/public" | jq .

# Initialize payment (include idempotency)
curl -s -X POST "$API/payments/initialize" \
  -H "Authorization: Bearer $BUYER_A_TOKEN" \
  -H "Idempotency-Key: $(uuidgen)" \
  -H "X-Device-FP: test-fp-001" \
  -H "Content-Type: application/json" \
  -d "{\"escrow_id\":\"$ESCROW_ID\",\"callback_url\":\"http://localhost:3001/app/payments/callback\"}" | jq .
```

---

**Document version:** 1.0  
**Maintained by:** Vescrow Engineering  
**Next review:** After major workflow or auth changes
