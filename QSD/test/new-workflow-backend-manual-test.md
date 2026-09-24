# New Workflow — Backend Manual Testing Guide

This guide covers manual testing of the **New Vescrow Workflow** features via **Swagger UI** and **curl**. It supplements the existing Phase 5 escrow and Phase 6 payment guides.

**Swagger UI:** [http://localhost:3000/swagger/](http://localhost:3000/swagger/)

**Covers:**

- Shareable listings (seller/buyer creator)
- First-payer-wins + quantity locking
- Dual fee model (3% in + 3% out)
- Direct bank disbursement (Paystack Transfer)
- Trust scoring, blocking, and appeals
- Payment provider abstraction (Paystack live, Flutterwave stub)

---

## Prerequisites

> **No Docker?** This guide uses **locally installed PostgreSQL and Redis**, or **free cloud instances**. Docker is optional — see [Optional: Docker setup](#optional-docker-setup) at the bottom.

### 1. One-time setup (no Docker)

#### 1a. Install PostgreSQL and Redis

**macOS (Homebrew):**

```bash
brew install postgresql@15 redis
brew services start postgresql@15
brew services start redis
```

**Ubuntu / Debian:**

```bash
sudo apt update
sudo apt install postgresql postgresql-contrib redis-server
sudo systemctl start postgresql redis-server
```

**Windows:** Install [PostgreSQL](https://www.postgresql.org/download/windows/) and [Redis for Windows](https://github.com/microsoftarchive/redis/releases) (or use cloud options below).

Verify services:

```bash
psql --version
redis-cli ping   # Expected: PONG
```

#### 1b. Create the database

```bash
# macOS: postgres user often matches your OS user
createuser -s postgres 2>/dev/null || true
createdb vescrow_db 2>/dev/null || psql -c "CREATE DATABASE vescrow_db;"

# Optional: dedicated user (recommended)
psql -c "CREATE USER vescrow_user WITH PASSWORD 'localdev';" 2>/dev/null || true
psql -c "GRANT ALL PRIVILEGES ON DATABASE vescrow_db TO vescrow_user;" 2>/dev/null || true
```

#### 1c. Generate JWT keys (first time only)

```bash
cd vescrow_backend
mkdir -p keys
openssl genrsa -out keys/private.pem 2048
openssl rsa -in keys/private.pem -pubout -out keys/public.pem
```

#### 1d. Configure environment

```bash
cp .env.example .env
```

Edit `.env` for **local** services (defaults in `.env.example` already target localhost):

```env
DB_HOST=localhost
DB_PORT=5432
DB_USER=postgres          # or vescrow_user
DB_PASSWORD=              # your local postgres password, if any
DB_NAME=vescrow_db
DB_SSL_MODE=disable

REDIS_URL=redis://localhost:6379
```

#### 1e. Cloud alternative (no local PostgreSQL/Redis install)

If you prefer not to install databases locally:

| Service | Provider | Set in `.env` |
|---------|----------|---------------|
| PostgreSQL | [Neon](https://neon.tech), [Supabase](https://supabase.com), [Railway](https://railway.app) | `DB_HOST`, `DB_PORT`, `DB_USER`, `DB_PASSWORD`, `DB_NAME`, `DB_SSL_MODE=require` |
| Redis | [Upstash](https://upstash.com), [Redis Cloud](https://redis.io/cloud/) | `REDIS_URL=rediss://...` (use the full connection URL from the dashboard) |

Copy the connection details from your provider into `.env`, then continue to step 2.

---

### 2. Start the backend

```bash
cd vescrow_backend
go mod download
go run cmd/server/main.go
```

On first run the server **auto-migrates** the schema and **seeds** roles/settings.

Verify health:

```bash
curl http://localhost:3000/api/v1/health
# Expected: {"status":"healthy",...}
```

**Common startup issues:**

| Error | Fix |
|-------|-----|
| `failed to connect to database` | PostgreSQL not running; wrong `DB_*` in `.env`; create `vescrow_db` |
| `connection refused` on Redis | Run `brew services start redis` or fix `REDIS_URL` |
| `Failed to initialize JWT keys` | Run the openssl commands in step 1c |
| `address already in use :3000` | Stop other process or set `PORT=3001` in `.env` |

---

### 3. Environment variables

Ensure `.env` includes at minimum (Paystack test keys from your [Paystack dashboard](https://dashboard.paystack.com)):

```env
APP_BASE_URL=http://localhost:3001
PAYMENT_PROVIDER=paystack
INCOMING_FEE_PERCENT=3
OUTGOING_FEE_PERCENT=3
PLATFORM_MARGIN_PERCENT=1
PAYSTACK_SECRET_KEY=sk_test_...
PAYSTACK_PUBLIC_KEY=pk_test_...
PAYSTACK_WEBHOOK_SECRET=...
```

---

### 4. Test accounts

Create **three** users via `POST /api/v1/auth/register`:

| Role | Email | Purpose |
|------|-------|---------|
| Seller | `seller@test.com` | Creates listings, delivers goods |
| Buyer A | `buyer-a@test.com` | Claims listing, pays |
| Buyer B | `buyer-b@test.com` | Race-condition / sold-out tests |
| Admin | `admin@test.com` | Appeal review (must have admin role) |

Promote KYC so create/claim/fund endpoints pass the gate:

```sql
UPDATE users SET kyc_status = 'verified'
WHERE email IN ('seller@test.com', 'buyer-a@test.com', 'buyer-b@test.com');
```

Promote admin:

```sql
UPDATE users SET role_id = (SELECT id FROM roles WHERE name = 'admin')
WHERE email = 'admin@test.com';
```

Connect to PostgreSQL for SQL checks:

```bash
# Local install
psql -d vescrow_db

# Or with explicit user/host
psql -h localhost -U postgres -d vescrow_db
```

---

### 5. Authenticate in Swagger

1. `POST /api/v1/auth/login` → copy `access_token`
2. Click **Authorize** → enter `Bearer <token>`
3. Re-authorize when switching users

### 6. Save these helper variables (curl)

```bash
export API=http://localhost:3000/api/v1
export SELLER_TOKEN="<seller_access_token>"
export BUYER_A_TOKEN="<buyer_a_access_token>"
export BUYER_B_TOKEN="<buyer_b_access_token>"
export ADMIN_TOKEN="<admin_access_token>"
export DEVICE_FP="test-device-fp-001"
```

All authenticated curl examples should include:

```bash
-H "Authorization: Bearer $TOKEN" \
-H "X-Device-FP: $DEVICE_FP" \
-H "Content-Type: application/json"
```

---

## Section A — Listings (Phase 1)

### A1. Seller creates a listing

| Field | Value |
|-------|-------|
| **Endpoint** | `POST /api/v1/listings` |
| **Auth** | Seller token + KYC |
| **Tag** | Listings |

```json
{
  "title": "iPhone 15 Pro Max 256GB",
  "description": "UK used, pristine condition",
  "amount_kobo": 85000000,
  "currency": "NGN",
  "creator_role": "seller",
  "quantity_total": 3,
  "inspection_period_days": 3
}
```

**Verify (201):**

- [ ] Response includes `share_token` and `share_url`
- [ ] `share_url` format: `{APP_BASE_URL}/e/{share_token}`
- [ ] `quantity_available` = `quantity_total`
- [ ] `status` = `"active"`
- [ ] Row exists in `escrow_listings` table

**Save:** `SHARE_TOKEN`, `LISTING_ID`

---

### A2. Public listing preview (no auth)

| Field | Value |
|-------|-------|
| **Endpoint** | `GET /api/v1/listings/{share_token}/public` |
| **Auth** | None |

```bash
curl "$API/listings/$SHARE_TOKEN/public"
```

**Verify (200):**

- [ ] Returns title, amount, `quantity_available`, seller name
- [ ] Does **not** expose internal UUIDs or seller email

---

### A3. Buyer claims a slot

| Field | Value |
|-------|-------|
| **Endpoint** | `POST /api/v1/listings/{share_token}/claim` |
| **Auth** | Buyer A token + KYC |

**Verify (201):**

- [ ] Returns `escrow_id` and message about 15-minute reservation
- [ ] New row in `escrows` with `listing_id` set, `status` = `"accepted"`
- [ ] New row in `payment_reservations` with `status` = `"reserved"`
- [ ] `escrow_listings.quantity_reserved` incremented by 1
- [ ] Fee fields on escrow: `incoming_fee_kobo`, `outgoing_fee_kobo`, `buyer_total_kobo`

**Save:** `ESCROW_ID`

---

### A4. Seller lists their listings

| Field | Value |
|-------|-------|
| **Endpoint** | `GET /api/v1/listings` |
| **Auth** | Seller token |

**Verify (200):**

- [ ] Paginated list includes the listing created in A1
- [ ] Each item has `share_url`, `quantity_available`, `order_count`

---

### A5. Seller views orders for a listing

| Field | Value |
|-------|-------|
| **Endpoint** | `GET /api/v1/listings/{listing_id}/orders` |
| **Auth** | Seller token |

**Verify (200):**

- [ ] Returns escrow orders linked to the listing
- [ ] Includes buyer info (preloaded)
- [ ] Non-seller/non-creator gets **403**

---

### A6. Buyer-initiated listing (optional)

Repeat A1 with `creator_role: "buyer"` and `seller_email: "seller@test.com"`.

**Verify:**

- [ ] `seller_id` resolves to seller account
- [ ] Creator cannot equal seller (400 if same email)

---

## Section B — Payments & First-Payer-Wins (Phase 2)

### B1. Initialize payment (dual fee)

| Field | Value |
|-------|-------|
| **Endpoint** | `POST /api/v1/payments/initialize` |
| **Auth** | Buyer A token (must be escrow buyer) |

```json
{
  "escrow_id": "<ESCROW_ID>",
  "callback_url": "http://localhost:3001/app/escrows/<ESCROW_ID>"
}
```

**Verify (200):**

- [ ] Returns `authorization_url` and `reference`
- [ ] Paystack charge amount = `buyer_total_kobo` (principal + 3%), not principal alone
- [ ] Example: ₦850,000 item → charge ₦875,500 (85000000 + 2550000 kobo)

**Save:** `PAYMENT_REF`, open `authorization_url` in browser

---

### B2. Complete Paystack sandbox payment

1. Open `authorization_url`
2. Use Paystack test card: `4084084084084081`, CVV `408`, expiry any future date, PIN `0000`
3. Complete checkout

**Verify:**

- [ ] Webhook `charge.success` received (check server logs)
- [ ] `escrow_payments.status` = `"confirmed"`
- [ ] Escrow `status` = `"funded"`, `funded_at` set
- [ ] `payment_reservations.status` = `"confirmed"`
- [ ] `escrow_listings.quantity_sold` incremented, `quantity_reserved` decremented
- [ ] System escrow wallet credited with **principal only** (`amount_kobo`)

Manual verify fallback:

```bash
curl -H "Authorization: Bearer $BUYER_A_TOKEN" \
  "$API/payments/verify/$PAYMENT_REF"
```

---

### B3. Race condition — qty = 1 (first payer wins)

**Setup:** Create a new listing with `quantity_total: 1`.

1. Buyer A claims → get `escrow_id_a`
2. Buyer B claims → get `escrow_id_b` (both should succeed — reservation phase)
3. Buyer A pays and webhook confirms
4. Buyer B pays (if they complete checkout)

**Verify:**

- [ ] Exactly **one** escrow reaches `funded`
- [ ] The loser gets `cancelled_overcapacity` OR auto-refund (check `escrow_payments.status` = `"refunded"`)
- [ ] Listing `quantity_sold` = 1, `status` = `"sold_out"`
- [ ] Third claim attempt returns **409** `"sold_out"`

---

### B4. Quantity = 3 (multiple winners)

**Setup:** Listing with `quantity_total: 3`.

1. Three different buyers claim and pay successfully
2. Fourth buyer attempts claim

**Verify:**

- [ ] Three escrows reach `funded`
- [ ] Fourth claim returns **409** `"sold_out"`
- [ ] Listing `quantity_sold` = 3

---

### B5. Reservation expiry

**Setup:** Claim a slot but do **not** pay within 15 minutes.

Wait 16+ minutes (or temporarily set `reservationTTL` lower in code for dev testing).

**Verify:**

- [ ] `payment_reservations.status` = `"expired"`
- [ ] Escrow reverts to `expired` if still `accepted`
- [ ] `quantity_reserved` decremented on listing
- [ ] Listing becomes available again if not sold out

---

## Section C — Escrow Lifecycle & Disbursement (Phase 3)

### C1. Seller accepts legacy escrow with bank details

For buyer-initiated escrows (`listing_id` IS NULL):

| Field | Value |
|-------|-------|
| **Endpoint** | `PUT /api/v1/escrows/{id}/accept` |
| **Auth** | Seller token |

```json
{
  "bank_code": "058",
  "account_number": "0123456789"
}
```

**Verify (200):**

- [ ] `seller_bank_code`, `seller_account_number` stored on escrow
- [ ] `disbursement_status` = `"pending"`

Listing-based escrows skip accept — they go straight to `accepted` on claim.

---

### C2. Seller marks delivered

| Field | Value |
|-------|-------|
| **Endpoint** | `PUT /api/v1/escrows/{id}/deliver` |
| **Auth** | Seller token |

```json
{
  "delivery_proof_url": "https://example.com/proof.jpg"
}
```

**Verify:** `status` = `"delivered"`, seller receives email notification

---

### C3. Buyer approves delivery → disbursement

| Field | Value |
|-------|-------|
| **Endpoint** | `PUT /api/v1/escrows/{id}/approve-delivery` |
| **Auth** | Buyer token |

**Verify (200):**

- [ ] Escrow `status` = `"completed"`
- [ ] Ledger: escrow wallet debited full principal
- [ ] Platform fee wallet credited outgoing fee
- [ ] If bank details present: Paystack Transfer initiated
- [ ] `disbursement_status` = `"processing"` or `"completed"` after webhook
- [ ] Seller receives `amount_kobo - outgoing_fee_kobo` (3% deducted)

Check Paystack dashboard for transfer record when using live test keys.

---

### C4. Dual fee math spot-check

For `amount_kobo = 10000000` (₦100,000):

| Leg | Calculation | Expected (kobo) |
|-----|-------------|-----------------|
| Incoming fee | 3% of principal | 300000 |
| Buyer pays | principal + incoming | 10300000 |
| Outgoing fee | 3% of principal | 300000 |
| Seller receives | principal - outgoing | 9700000 |
| Platform total | incoming margin + outgoing margin | ~200000 (1% each leg if configured) |

---

### C5. KYC gate (no tiers required)

**Verify:**

- [ ] User with `kyc_status = 'verified'` can create listings and claim
- [ ] User with `kyc_status = 'none'` gets **403** on `POST /listings` and `POST /listings/:token/claim`
- [ ] User with `kyc_status = 'approved'` also passes the gate (admin-approved KYC)

---

## Section D — Trust, Blocking & Appeals (Phase 4)

### D1. View trust profile

| Field | Value |
|-------|-------|
| **Endpoint** | `GET /api/v1/trust/profile` |
| **Auth** | Any user token |

**Verify (200):**

- [ ] New user: `score` = 100, `status` = `"good"`
- [ ] Profile auto-created on first access

---

### D2. Simulate trust score drop (database)

Manually lower a user's score to test blocking:

```sql
UPDATE trust_profiles SET score = 15, status = 'blocked' WHERE user_id = '<buyer_uuid>';
INSERT INTO block_entries (id, user_id, reason, blocked_at, active)
VALUES (uuid_generate_v4(), '<buyer_uuid>', 'Manual test block', NOW(), true);
```

**Verify blocked user cannot:**

- [ ] `POST /api/v1/listings` → **403** `{ "code": "ACCOUNT_BLOCKED" }`
- [ ] `POST /api/v1/listings/:token/claim` → **403**
- [ ] Response includes appeal hint

**Verify existing funded escrows still accessible** via `GET /api/v1/escrows/{id}`

---

### D3. Device fingerprint blocking

```sql
INSERT INTO block_entries (id, device_fp, reason, blocked_at, active)
VALUES (uuid_generate_v4(), 'blocked-device-123', 'Device flagged', NOW(), true);
```

Send requests with:

```bash
-H "X-Device-FP: blocked-device-123"
```

**Verify:** **403 ACCOUNT_BLOCKED** even for a different user account

---

### D4. Submit appeal

| Field | Value |
|-------|-------|
| **Endpoint** | `POST /api/v1/appeals` |
| **Auth** | Blocked user token |

```json
{
  "block_id": "<BLOCK_ENTRY_UUID>",
  "reason": "This was a misunderstanding. I have provided proof of delivery to support."
}
```

**Verify (201):**

- [ ] Appeal created with `status` = `"pending"`

---

### D5. Admin reviews appeal

| Field | Value |
|-------|-------|
| **Endpoint** | `PUT /api/v1/admin/appeals/{id}/review` |
| **Auth** | Admin token |

Approve:

```json
{ "approve": true, "note": "Verified with support team." }
```

**Verify:**

- [ ] Appeal `status` = `"approved"`
- [ ] `block_entries.active` = false for linked block
- [ ] User trust profile reset to `score` = 50, `status` = `"warning"`
- [ ] User can create listings again

Reject:

```json
{ "approve": false, "note": "Insufficient evidence." }
```

**Verify:** Block remains active

---

### D6. Dispute → trust event

1. Complete escrow to `delivered`
2. Buyer opens dispute: `POST /api/v1/disputes`
3. Admin resolves with `full_refund`

**Verify:**

- [ ] Seller trust score decreased by 15
- [ ] `trust_events` row with `event_type` = `"dispute_lost"`

---

## Section E — Legacy Buyer-Initiated Escrow

The original flow still works for escrows with `listing_id = NULL`:

1. `POST /api/v1/escrows` (buyer creates draft)
2. `PUT /api/v1/escrows/{id}/submit`
3. Seller `PUT /api/v1/escrows/{id}/accept` (with bank details)
4. Buyer `POST /api/v1/payments/initialize`
5. Paystack webhook → `funded`
6. Seller `PUT /api/v1/escrows/{id}/deliver`
7. Buyer `PUT /api/v1/escrows/{id}/approve-delivery`

See [phase5-escrow-manual-test.md](./phase5-escrow-manual-test.md) for detailed steps.

**Verify new fee fields populated** on create for legacy escrows too.

---

## Section F — End-to-End Checklist

Use this as a sign-off list before demo/production:

| # | Scenario | Pass |
|---|----------|------|
| 1 | Seller creates listing → share URL returned | ☐ |
| 2 | Public preview works without login | ☐ |
| 3 | Buyer claims → escrow + reservation created | ☐ |
| 4 | Payment charges principal + 3% | ☐ |
| 5 | Webhook funds escrow + confirms reservation | ☐ |
| 6 | qty=1 race → one winner, one refunded/blocked | ☐ |
| 7 | qty=3 → three successful payments, fourth sold out | ☐ |
| 8 | Reservation expires after 15 min without payment | ☐ |
| 9 | Seller delivers → buyer approves | ☐ |
| 10 | Paystack Transfer initiated to seller bank | ☐ |
| 11 | Seller receives principal - 3% | ☐ |
| 12 | Blocked user gets 403 with appeal URL | ☐ |
| 13 | Admin appeal approval unblocks user | ☐ |
| 14 | Email notifications contain working deep links | ☐ |

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| 403 on create/claim | KYC not verified | Set `kyc_status = 'verified'` |
| 403 ACCOUNT_BLOCKED | Trust score / block entry | Check `trust_profiles`, `block_entries` |
| Payment init 400 | Reservation expired | Re-claim the listing |
| Paystack 400 on init | Invalid test keys | Verify `.env` Paystack keys |
| Transfer failed | Invalid bank details | Use Paystack test bank codes |
| CORS error from frontend | Origin not allowed | Backend allows `localhost:3001` |
| Webhook not firing | Local dev without tunnel | Use `POST /payments/verify/{ref}` manually |
| quantity_sold wrong | Webhook retry / race | Check idempotency; inspect `payment_reservations` |

---

## Database Quick Reference

```sql
-- Listings
SELECT id, title, quantity_total, quantity_sold, quantity_reserved, status, share_token
FROM escrow_listings ORDER BY created_at DESC LIMIT 5;

-- Orders linked to listing
SELECT id, listing_id, buyer_id, status, slot_number, buyer_total_kobo
FROM escrows WHERE listing_id IS NOT NULL ORDER BY created_at DESC;

-- Reservations
SELECT id, escrow_id, status, expires_at FROM payment_reservations ORDER BY created_at DESC;

-- Trust
SELECT * FROM trust_profiles WHERE user_id = '<uuid>';
SELECT * FROM block_entries WHERE active = true;
SELECT * FROM appeals ORDER BY created_at DESC;
```

---

**Document version:** 1.1  
**Last updated:** August 2026  
**Related guides:** [phase5-escrow-manual-test.md](./phase5-escrow-manual-test.md), [payment_manual_test_guide.md](./payment_manual_test_guide.md), [phase8-dispute-manual-test.md](./phase8-dispute-manual-test.md)

---

## Optional: Docker setup

Use this only if you have Docker installed and prefer containers over local services.

```bash
cd vescrow_backend
docker-compose up -d vescrow_db redis
# Set in .env for docker-compose network:
# DB_HOST=vescrow_db
# REDIS_URL=redis://redis:6379
go run cmd/server/main.go
```

When running the Go app **on your host** (not inside the `app` container), keep `DB_HOST=localhost` and map ports — `docker-compose` exposes Postgres on `5432` and Redis on `6379` by default, so the local `.env` values in step 1d still work.
