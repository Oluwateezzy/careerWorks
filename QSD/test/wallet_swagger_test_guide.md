# Wallet API — Manual Swagger Test Guide

This guide walks you through testing the Wallet & Double-Entry Ledger endpoints using the Swagger UI at `http://localhost:3000/swagger/index.html`.

> **Prerequisites**: The server must be running (`go run cmd/server/main.go`), a test user must be registered, and a valid JWT access token must be obtained via the `/api/v1/auth/login` endpoint.

---

## 1. Authenticate

1. Open **Swagger UI** → Click the green **Authorize** button (top-right).
2. Paste your JWT access token (without `Bearer ` prefix).
3. Click **Authorize** → **Close**.

All subsequent requests will include the `Authorization: Bearer <token>` header automatically.

---

## 2. Test Endpoints

### 2.1 GET `/api/v1/wallet` — Get Wallet Balance

1. Expand **Wallet** → `GET /api/v1/wallet`.
2. Click **Try it out** → **Execute**.
3. **Expected Response** (200):
```json
{
  "success": true,
  "message": "Balance retrieved successfully",
  "data": {
    "balance_kobo": 0,
    "currency": "NGN",
    "is_frozen": false
  }
}
```
4. **Verify**: `balance_kobo` is `0` for a new user. `currency` is `"NGN"`.

---

### 2.2 GET `/api/v1/wallet/banks` — List Nigerian Banks

1. Expand **Wallet** → `GET /api/v1/wallet/banks`.
2. Click **Try it out** → **Execute**.
3. **Expected Response** (200):
```json
{
  "success": true,
  "message": "Banks retrieved successfully",
  "data": [
    { "name": "Guaranty Trust Bank", "code": "058" },
    { "name": "Access Bank", "code": "044" }
  ]
}
```
4. **Verify**: Response contains an array of banks with `name` and `code` fields.

---

### 2.3 POST `/api/v1/wallet/verify-account` — Verify Bank Account

1. Expand **Wallet** → `POST /api/v1/wallet/verify-account`.
2. Click **Try it out**.
3. Enter the request body:
```json
{
  "account_number": "0123456789",
  "bank_code": "058"
}
```
4. Click **Execute**.
5. **Expected Response** (200):
```json
{
  "success": true,
  "message": "Account verified successfully",
  "data": {
    "account_number": "0123456789",
    "account_name": "JOHN DOE"
  }
}
```
6. **Verify**: Response includes a resolved `account_name` from Paystack.

> **Note**: This endpoint calls the live Paystack API. Use real test account numbers from Paystack's test environment.

---

### 2.4 GET `/api/v1/wallet/transactions` — Get Transaction History

1. Expand **Wallet** → `GET /api/v1/wallet/transactions`.
2. Click **Try it out**.
3. Set query params: `page=1`, `per_page=10`.
4. Click **Execute**.
5. **Expected Response** (200):
```json
{
  "success": true,
  "message": "Transactions retrieved successfully",
  "data": [],
  "meta": {
    "page": 1,
    "per_page": 10,
    "total": 0,
    "total_pages": 1
  }
}
```
6. **Verify**: `data` is an empty array for a fresh wallet. After funding an escrow, entries will appear.

---

### 2.5 POST `/api/v1/wallet/withdraw` — Request Withdrawal

> **Prerequisite**: Your wallet must have a positive balance. Complete an escrow lifecycle first (create → fund → deliver → approve) to receive seller payout.

1. Expand **Wallet** → `POST /api/v1/wallet/withdraw`.
2. Click **Try it out**.
3. Enter the request body:
```json
{
  "amount_kobo": 200000,
  "bank_code": "058",
  "account_number": "0123456789",
  "account_name": "John Doe"
}
```
4. Click **Execute**.
5. **Expected Response** (200):
```json
{
  "success": true,
  "message": "Withdrawal request initiated successfully",
  "data": {
    "id": "uuid-here",
    "wallet_id": "uuid-here",
    "user_id": "uuid-here",
    "amount_kobo": 200000,
    "bank_code": "058",
    "account_number": "0123456789",
    "account_name": "John Doe",
    "status": "pending",
    "created_at": "2026-07-11T10:00:00Z"
  }
}
```

#### Error Cases to Test

| Scenario | Request | Expected Status | Expected Message |
|---|---|---|---|
| Below minimum (₦1,000) | `amount_kobo: 50000` | 400 | "amount ... below minimum limit" |
| Exceeds per-tx limit (₦2M) | `amount_kobo: 200000001` | 400 | "exceeds per-transaction limit" |
| Insufficient balance | `amount_kobo: 999999999` | 400 | "insufficient wallet balance" |
| Missing fields | `{}` | 400 | Validation errors array |
| 24h cooldown (new user) | Any valid amount | 400 | "cooldown active until..." |

---

### 2.6 GET `/api/v1/wallet/withdrawals` — Get Withdrawal History

1. Expand **Wallet** → `GET /api/v1/wallet/withdrawals`.
2. Click **Try it out**.
3. Set query params: `page=1`, `per_page=10`.
4. Click **Execute**.
5. **Expected Response** (200):
```json
{
  "success": true,
  "message": "Withdrawal history retrieved successfully",
  "data": [
    {
      "id": "uuid-here",
      "amount_kobo": 200000,
      "status": "pending"
    }
  ],
  "meta": {
    "page": 1,
    "per_page": 10,
    "total": 1,
    "total_pages": 1
  }
}
```

---

## 3. Full Escrow Lifecycle Test (End-to-End)

To properly test the wallet with real balance changes, run through the full lifecycle:

| Step | Endpoint | Actor | What Happens to Wallets |
|---|---|---|---|
| 1 | `POST /api/v1/escrows` | Buyer | Creates escrow (no wallet change) |
| 2 | `PUT /api/v1/escrows/:id/submit` | Buyer | Submits to seller (no wallet change) |
| 3 | `PUT /api/v1/escrows/:id/accept` | Seller | Accepts terms (no wallet change) |
| 4 | `POST /api/v1/payments/initialize` | Buyer | Gets Paystack checkout URL |
| 5 | Complete Paystack payment | Buyer | Webhook fires → `FundEscrow` credits escrow system wallet |
| 6 | `PUT /api/v1/escrows/:id/deliver` | Seller | Marks delivery (no wallet change) |
| 7 | `PUT /api/v1/escrows/:id/approve-delivery` | Buyer | Approves → `ReleaseToSeller` credits seller wallet |
| 8 | `GET /api/v1/wallet` | Seller | Verify balance = amount - platform fee |
| 9 | `POST /api/v1/wallet/withdraw` | Seller | Withdraw funds to bank |

After steps 5 and 7, verify the **double-entry invariant** by querying:
```sql
SELECT
  SUM(CASE WHEN entry_type='credit' THEN amount_kobo ELSE 0 END) -
  SUM(CASE WHEN entry_type='debit' THEN amount_kobo ELSE 0 END) AS balance
FROM ledger_entries
WHERE escrow_id = '<escrow-uuid>';
-- Expected: 0
```

---

## 4. Authentication Error Cases

| Scenario | Action | Expected |
|---|---|---|
| No token | Call any endpoint without Authorization header | 401 Unauthorized |
| Expired token | Use an expired JWT | 401 Unauthorized |
| Invalid token | Use a garbage string | 401 Unauthorized |
| KYC not met | Call `POST /wallet/withdraw` with KYC tier "none" | 403 Forbidden |
