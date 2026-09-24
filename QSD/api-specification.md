# Vescrow — API Specification

**Version**: 2.0  
**Last Updated**: August 2026  
**Format**: RESTful JSON API  
**Base URL**: `/api/v1`  
**Security reference**: [security-architecture.md](./security-architecture.md)

---

## 1. Overview & Conventions

### 1.1 Headers

**Authentication** — use one of:

```http
Authorization: Bearer <JWT_ACCESS_TOKEN>
```

Or rely on the HttpOnly `access_token` cookie set at login (browser clients via same-origin proxy).

**Optional / conditional headers**:

| Header | When Required | Purpose |
|--------|--------------|---------|
| `Content-Type: application/json` | JSON bodies | Request format |
| `Idempotency-Key: <uuid>` | `POST /payments/initialize`, `POST /wallet/withdraw` | Prevent duplicate money operations |
| `X-Device-FP: <fingerprint>` | Recommended on authenticated requests | Trust/device tracking |
| `X-API-Key: <prefix>.<secret>` | B2B integrations | Organization-scoped API access |
| `X-TOTP-Code: <6-digit>` | Admin routes in production when 2FA enabled | Admin MFA |

**Cookies** (set by server, HttpOnly):

| Cookie | Purpose |
|--------|---------|
| `access_token` | RS256 JWT, 15 min |
| `refresh_token` | RS256 JWT, 30 days; used by `POST /auth/refresh` |

### 1.2 Rate Limiting

Rate limiting uses Redis when available (`InitLimiterRedis`).

- **General API**: 100 requests per minute per IP.
- **Auth (Login/OTP)**: 5 requests per 15 minutes per IP.
- **Financial/Wallet Operations**: 10 requests per minute per User.

When a rate limit is exceeded, the server returns `429 Too Many Requests` with:
```json
{
  "success": false,
  "message": "Rate limit exceeded. Try again in X seconds."
}
```

### 1.3 Standard Response Formats

#### Success Response
```json
{
  "success": true,
  "message": "Action completed successfully",
  "data": {}
}
```

#### Error Response
```json
{
  "success": false,
  "message": "Validation failed",
  "errors": [
    {
      "field": "email",
      "message": "Must be a valid email address"
    }
  ]
}
```

#### Paginated Response
```json
{
  "success": true,
  "message": "Resources retrieved successfully",
  "data": [],
  "meta": {
    "page": 1,
    "per_page": 20,
    "total": 150,
    "total_pages": 8
  }
}
```

---

## 2. Authentication — `/api/v1/auth`

### 2.1 Register User
- **Method**: `POST`
- **Path**: `/register`
- **Authentication**: None
- **Request Body**:
```json
{
  "name": "Oluwaseun Adebayo",
  "email": "seun.adebayo@example.com",
  "phone": "+2348031234567",
  "password": "SecurePassword123!"
}
```
- **Responses**:
  - **201 Created**:
  ```json
  {
    "success": true,
    "message": "Registration successful. Please verify your email and phone number.",
    "data": {
      "id": "e83b4fa4-7e8c-4a3b-8d9e-1f2a3b4c5d6e",
      "name": "Oluwaseun Adebayo",
      "email": "seun.adebayo@example.com",
      "phone": "+234803****567",
      "email_verified": false,
      "phone_verified": false,
      "kyc_status": "none",
      "created_at": "2026-06-14T17:15:00Z"
    }
  }
  ```
  - **400 Bad Request** (Validation or Email/Phone exists):
  ```json
  {
    "success": false,
    "message": "Email or phone number already registered"
  }
  ```

### 2.2 Login User
- **Method**: `POST`
- **Path**: `/login`
- **Authentication**: None
- **Request Body**:
```json
{
  "email": "seun.adebayo@example.com",
  "password": "SecurePassword123!"
}
```
- **Responses**:
  - **200 OK**:
  ```json
  {
    "success": true,
    "message": "Login successful",
    "data": {
      "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
      "authenticated": true
    }
  }
  ```
  - **Set-Cookie Headers** (HttpOnly, Secure in HTTPS):
  ```http
  Set-Cookie: access_token=eyJhbGciOiJSUzI1NiI...; HttpOnly; Secure; SameSite=Lax; Max-Age=900
  Set-Cookie: refresh_token=eyJhbGciOiJSUzI1NiI...; HttpOnly; Secure; SameSite=Lax; Max-Age=2592000
  ```
  - **401 Unauthorized**:
  ```json
  {
    "success": false,
    "message": "Invalid email or password"
  }
  ```

### 2.3 Verify Email
- **Method**: `POST`
- **Path**: `/verify-email`
- **Authentication**: None
- **Request Body**:
```json
{
  "email": "seun.adebayo@example.com",
  "otp": "123456"
}
```
- **Responses**:
  - **200 OK**:
  ```json
  {
    "success": true,
    "message": "Email verified successfully"
  }
  ```
  - **400 Bad Request**:
  ```json
  {
    "success": false,
    "message": "Invalid or expired OTP"
  }
  ```

### 2.4 Verify Phone
- **Method**: `POST`
- **Path**: `/verify-phone`
- **Authentication**: None
- **Request Body**:
```json
{
  "phone": "+2348031234567",
  "otp": "654321"
}
```
- **Responses**:
  - **200 OK**:
  ```json
  {
    "success": true,
    "message": "Phone number verified successfully"
  }
  ```

### 2.5 Resend OTP
- **Method**: `POST`
- **Path**: `/resend-otp`
- **Authentication**: None
- **Request Body**:
```json
{
  "channel": "email", 
  "target": "seun.adebayo@example.com"
}
```
- **Responses**:
  - **200 OK**:
  ```json
  {
    "success": true,
    "message": "OTP sent successfully"
  }
  ```

### 2.6 Refresh Token
- **Method**: `POST`
- **Path**: `/refresh`
- **Authentication**: Refresh Token (passed in HTTP-only Cookie `refresh_token`)
- **Responses**:
  - **200 OK**:
  ```json
  {
    "success": true,
    "message": "Token refreshed successfully",
    "data": {
      "access_token": "new_access_token_here"
    }
  }
  ```
  - **401 Unauthorized**:
  ```json
  {
    "success": false,
    "message": "Invalid or expired refresh token"
  }
  ```

### 2.7 Logout
- **Method**: `POST`
- **Path**: `/logout`
- **Authentication**: None (uses `refresh_token` cookie to revoke session)
- **Responses**:
  - **200 OK**:
  ```json
  {
    "success": true,
    "message": "Logout successful"
  }
  ```
  Clears `access_token` and `refresh_token` cookies.

### 2.8 Get Session
- **Method**: `GET`
- **Path**: `/session`
- **Authentication**: JWT or `access_token` cookie
- **Responses**:
  - **200 OK**:
  ```json
  {
    "success": true,
    "message": "Session active",
    "data": {
      "id": "e83b4fa4-7e8c-4a3b-8d9e-1f2a3b4c5d6e",
      "name": "Oluwaseun Adebayo",
      "email": "seun.adebayo@example.com",
      "kyc_status": "verified",
      "role": "user"
    }
  }
  ```
  - **401 Unauthorized**: Token missing, expired, or user not found.

### 2.9 Forgot Password
- **Method**: `POST`
- **Path**: `/forgot-password`
- **Authentication**: None
- **Request Body**:
```json
{
  "email": "seun.adebayo@example.com"
}
```
- **Responses**:
  - **200 OK**:
  ```json
  {
    "success": true,
    "message": "Password reset instructions sent if email exists"
  }
  ```

### 2.10 Reset Password
- **Method**: `POST`
- **Path**: `/reset-password`
- **Authentication**: Reset Token (passed as query param or body token)
- **Request Body**:
```json
{
  "token": "reset_token_from_email",
  "new_password": "NewSecurePassword123!"
}
```
- **Responses**:
  - **200 OK**:
  ```json
  {
    "success": true,
    "message": "Password reset successfully"
  }
  ```

### 2.11 Change Password
- **Method**: `POST`
- **Path**: `/change-password`
- **Authentication**: JWT
- **Request Body**:
```json
{
  "current_password": "SecurePassword123!",
  "new_password": "NewSecurePassword123!"
}
```
- **Responses**:
  - **200 OK**:
  ```json
  {
    "success": true,
    "message": "Password changed successfully"
  }
  ```

### 2.12 Get Profile
- **Method**: `GET`
- **Path**: `/profile`
- **Authentication**: JWT
- **Responses**:
  - **200 OK**:
  ```json
  {
    "success": true,
    "message": "Profile retrieved successfully",
    "data": {
      "id": "e83b4fa4-7e8c-4a3b-8d9e-1f2a3b4c5d6e",
      "name": "Oluwaseun Adebayo",
      "email": "seun.adebayo@example.com",
      "phone": "+234803****567",
      "email_verified": true,
      "phone_verified": true,
      "role": "user",
      "kyc_status": "approved",
      "created_at": "2026-06-14T17:15:00Z"
    }
  }
  ```

### 2.13 Update Profile
- **Method**: `PUT`
- **Path**: `/profile`
- **Authentication**: JWT
- **Request Body**:
```json
{
  "name": "Seun A. Adebayo"
}
```
- **Responses**:
  - **200 OK**:
  ```json
  {
    "success": true,
    "message": "Profile updated successfully",
    "data": {
      "id": "e83b4fa4-7e8c-4a3b-8d9e-1f2a3b4c5d6e",
      "name": "Seun A. Adebayo"
    }
  }
  ```

---

## 3. KYC Management — `/api/v1/kyc`

### 3.1 Verify BVN (Bank Verification Number)
- **Method**: `POST`
- **Path**: `/verify-bvn`
- **Authentication**: JWT
- **Request Body**:
```json
{
  "bvn": "22233344455"
}
```
- **Responses**:
  - **200 OK**:
  ```json
  {
    "success": true,
    "message": "BVN verified and linked successfully.",
    "data": {
      "kyc_status": "verified"
    }
  }
  ```
  - **400 Bad Request**:
  ```json
  {
    "success": false,
    "message": "BVN verification failed: Name mismatch or invalid BVN"
  }
  ```

### 3.2 Verify NIN (National Identification Number)
- **Method**: `POST`
- **Path**: `/verify-nin`
- **Authentication**: JWT
- **Request Body**:
```json
{
  "nin": "12345678901"
}
```
- **Responses**:
  - **200 OK**:
  ```json
  {
    "success": true,
    "message": "NIN verified and linked successfully"
  }
  ```

### 3.3 Submit KYC Document (IDs, Proof of Address)
- **Method**: `POST`
- **Path**: `/submit-document`
- **Authentication**: JWT
- **Request Body**: Multipart Form Data
  - `document_type`: `passport` | `drivers_license` | `national_id` | `utility_bill` | `cac_cert`
  - `document_number`: `A12345678`
  - `document_file`: [File Upload]
  - `selfie_file`: [File Upload] (Required for tier 3 verification)
- **Responses**:
  - **202 Accepted**:
  ```json
  {
    "success": true,
    "message": "KYC document submitted successfully. Awaiting admin approval.",
    "data": {
      "id": "c73a4fa4-7e8c-4a3b-8d9e-1f2a3b4c5d6f",
      "document_type": "passport",
      "status": "pending"
    }
  }
  ```

### 3.4 Get KYC Status
- **Method**: `GET`
- **Path**: `/status`
- **Authentication**: JWT
- **Responses**:
  - **200 OK**:
  ```json
  {
    "success": true,
    "message": "KYC status retrieved",
    "data": {
      "kyc_status": "verified",
      "kyc_status": "approved",
      "bvn_linked": true,
      "nin_linked": false,
      "documents": [
        {
          "id": "c73a4fa4-7e8c-4a3b-8d9e-1f2a3b4c5d6f",
          "document_type": "passport",
          "status": "approved",
          "verified_at": "2026-06-14T17:15:00Z"
        }
      ]
    }
  }
  ```

---

## 4. Escrows — `/api/v1/escrows`

### 4.1 Create Escrow
- **Method**: `POST`
- **Path**: `/`
- **Authentication**: JWT + KYC verified (`kyc_status` = `verified` or `approved`) + TrustCheck
- **Request Body**:
```json
{
  "title": "Purchase of MacBook Pro M3 Max",
  "description": "MacBook Pro 16-inch, 32GB RAM, 1TB SSD in pristine condition",
  "amount_kobo": 350000000,
  "currency": "NGN",
  "seller_email": "seller.merch@example.com",
  "delivery_date": "2026-06-20",
  "inspection_period_days": 3,
  "milestones": [
    {
      "title": "Initial Deposit & Inspection",
      "description": "50% deposit release upon visual inspection",
      "amount_kobo": 175000000,
      "sequence_order": 1
    },
    {
      "title": "Final Handover",
      "description": "Remaining 50% release upon device verification",
      "amount_kobo": 175000000,
      "sequence_order": 2
    }
  ]
}
```
- **Responses**:
  - **201 Created**:
  ```json
  {
    "success": true,
    "message": "Escrow transaction created as draft",
    "data": {
      "id": "a11b4fa4-7e8c-4a3b-8d9e-1f2a3b4c5d6e",
      "title": "Purchase of MacBook Pro M3 Max",
      "amount_kobo": 350000000,
      "currency": "NGN",
      "status": "draft",
      "buyer_id": "e83b4fa4-7e8c-4a3b-8d9e-1f2a3b4c5d6e",
      "seller_id": "b22b4fa4-7e8c-4a3b-8d9e-1f2a3b4c5d6f",
      "escrow_fee_kobo": 10500000,
      "inspection_period_days": 3,
      "created_at": "2026-06-14T17:20:00Z"
    }
  }
  ```

### 4.2 List Escrows
- **Method**: `GET`
- **Path**: `/`
- **Authentication**: JWT
- **Query Parameters**:
  - `page`: default `1`
  - `per_page`: default `20`
  - `role`: `buyer` | `seller`
  - `status`: `draft` | `pending_acceptance` | `accepted` | `funded` | `in_progress` | `delivered` | `completed` | `disputed` | `refunded`
- **Responses**:
  - **200 OK**:
  ```json
  {
    "success": true,
    "message": "Escrow list retrieved",
    "data": [
      {
        "id": "a11b4fa4-7e8c-4a3b-8d9e-1f2a3b4c5d6e",
        "title": "Purchase of MacBook Pro M3 Max",
        "amount_kobo": 350000000,
        "currency": "NGN",
        "status": "funded",
        "partner_name": "Tech Merchants Ltd",
        "created_at": "2026-06-14T17:20:00Z"
      }
    ],
    "meta": {
      "page": 1,
      "per_page": 20,
      "total": 1,
      "total_pages": 1
    }
  }
  ```

### 4.3 Get Escrow Details
- **Method**: `GET`
- **Path**: `/:id`
- **Authentication**: JWT
- **Responses**:
  - **200 OK**:
  ```json
  {
    "success": true,
    "message": "Escrow details retrieved",
    "data": {
      "id": "a11b4fa4-7e8c-4a3b-8d9e-1f2a3b4c5d6e",
      "title": "Purchase of MacBook Pro M3 Max",
      "description": "MacBook Pro 16-inch, 32GB RAM, 1TB SSD in pristine condition",
      "amount_kobo": 350000000,
      "currency": "NGN",
      "status": "funded",
      "buyer_id": "e83b4fa4-7e8c-4a3b-8d9e-1f2a3b4c5d6e",
      "seller_id": "b22b4fa4-7e8c-4a3b-8d9e-1f2a3b4c5d6f",
      "escrow_fee_kobo": 10500000,
      "inspection_period_days": 3,
      "delivery_date": "2026-06-20",
      "funded_at": "2026-06-14T17:25:00Z",
      "milestones": [
        {
          "id": "m1-id-goes-here",
          "title": "Initial Deposit & Inspection",
          "amount_kobo": 175000000,
          "status": "pending"
        }
      ]
    }
  }
  ```

### 4.4 Submit Escrow Terms
- **Method**: `PUT`
- **Path**: `/:id/submit`
- **Authentication**: JWT (Buyer)
- **Description**: Transitions escrow from `draft` to `pending_acceptance`. Notifies the seller to accept terms.
- **Responses**:
  - **200 OK**:
  ```json
  {
    "success": true,
    "message": "Escrow terms submitted to seller",
    "data": {
      "status": "pending_acceptance"
    }
  }
  ```

### 4.5 Accept Escrow Terms
- **Method**: `PUT`
- **Path**: `/:id/accept`
- **Authentication**: JWT (Seller)
- **Responses**:
  - **200 OK**:
  ```json
  {
    "success": true,
    "message": "Escrow terms accepted by seller. Awaiting funding.",
    "data": {
      "status": "accepted"
    }
  }
  ```

### 4.6 Reject Escrow Terms
- **Method**: `PUT`
- **Path**: `/:id/reject`
- **Authentication**: JWT (Seller)
- **Request Body**:
```json
{
  "reason": "Amount is too low; agreed on ₦360k, not ₦350k."
}
```
- **Responses**:
  - **200 OK**:
  ```json
  {
    "success": true,
    "message": "Escrow terms rejected",
    "data": {
      "status": "rejected"
    }
  }
  ```

### 4.7 Initialize Escrow Funding (Dev Only)

> **Production**: Use `POST /api/v1/payments/initialize` instead. This endpoint bypasses Paystack and is blocked outside development/test environments.

- **Method**: `POST`
- **Path**: `/:id/fund`
- **Authentication**: JWT (Buyer) + `DevOnly()` middleware
- **Environment**: `development` or `test` only
- **Responses**:
  - **200 OK** (dev only):
  ```json
  {
    "success": true,
    "message": "Escrow funded (dev bypass)",
    "data": {
      "status": "funded"
    }
  }
  ```
  - **403 Forbidden** (production/staging): Endpoint not available.

### 4.8 Mark as Delivered
- **Method**: `PUT`
- **Path**: `/:id/deliver`
- **Authentication**: JWT (Seller)
- **Request Body**:
```json
{
  "delivery_proof_url": "https://s3.amazonaws.com/vescrow/proofs/proof-123.jpg",
  "notes": "Delivered via GIG Logistics. Tracking ID: GIG-998877"
}
```
- **Responses**:
  - **200 OK**:
  ```json
  {
    "success": true,
    "message": "Escrow marked as delivered. Inspection period has begun.",
    "data": {
      "status": "delivered",
      "delivered_at": "2026-06-15T10:00:00Z"
    }
  }
  ```

### 4.9 Approve Delivery & Release Funds
- **Method**: `PUT`
- **Path**: `/:id/approve-delivery`
- **Authentication**: JWT (Buyer)
- **Responses**:
  - **200 OK**:
  ```json
  {
    "success": true,
    "message": "Delivery approved. Funds released to seller wallet.",
    "data": {
      "status": "completed",
      "completed_at": "2026-06-16T12:00:00Z"
    }
  }
  ```

### 4.10 Request Inspection Period Extension
- **Method**: `PUT`
- **Path**: `/:id/extend-inspection`
- **Authentication**: JWT (Buyer)
- **Request Body**:
```json
{
  "additional_days": 3,
  "reason": "Need to take it to Apple service center for hardware diagnostic."
}
```
- **Responses**:
  - **200 OK**:
  ```json
  {
    "success": true,
    "message": "Inspection extension request submitted to seller"
  }
  ```

### 4.11 Send Message
- **Method**: `POST`
- **Path**: `/:id/messages`
- **Authentication**: JWT
- **Request Body**:
```json
{
  "content": "Hello! Just shipped the laptop via GIG Logistics.",
  "attachment_url": ""
}
```
- **Responses**:
  - **201 Created**:
  ```json
  {
    "success": true,
    "message": "Message sent",
    "data": {
      "id": "msg-992381-da8b",
      "sender_id": "b22b4fa4-7e8c-4a3b-8d9e-1f2a3b4c5d6f",
      "content": "Hello! Just shipped the laptop via GIG Logistics.",
      "created_at": "2026-06-14T17:30:00Z"
    }
  }
  ```

### 4.12 Get Messages
- **Method**: `GET`
- **Path**: `/:id/messages`
- **Authentication**: JWT
- **Responses**:
  - **200 OK**:
  ```json
  {
    "success": true,
    "data": [
      {
        "id": "msg-992381-da8b",
        "sender_id": "b22b4fa4-7e8c-4a3b-8d9e-1f2a3b4c5d6f",
        "content": "Hello! Just shipped the laptop via GIG Logistics.",
        "created_at": "2026-06-14T17:30:00Z"
      }
    ]
  }
  ```

---

## 5. Wallet & Payouts — `/api/v1/wallet`

### 5.1 Get Wallet Balance
- **Method**: `GET`
- **Path**: `/`
- **Authentication**: JWT
- **Responses**:
  - **200 OK**:
  ```json
  {
    "success": true,
    "message": "Wallet balance retrieved",
    "data": {
      "balance_kobo": 339500000, 
      "currency": "NGN",
      "is_frozen": false
    }
  }
  ```

### 5.2 Get Wallet Transaction Ledger
- **Method**: `GET`
- **Path**: `/transactions`
- **Authentication**: JWT
- **Responses**:
  - **200 OK**:
  ```json
  {
    "success": true,
    "data": [
      {
        "id": "ledger-uuid-1",
        "entry_type": "credit",
        "amount_kobo": 339500000,
        "balance_after_kobo": 339500000,
        "category": "escrow_release",
        "description": "Released funds for MacBook Pro M3 Max (3% fee deducted)",
        "reference": "LEDGER_REF_12345",
        "created_at": "2026-06-16T12:00:00Z"
      }
    ]
  }
  ```

### 5.3 Verify Bank Account (NUBAN Lookup)
- **Method**: `POST`
- **Path**: `/verify-account`
- **Authentication**: JWT
- **Request Body**:
```json
{
  "account_number": "0123456789",
  "bank_code": "058"
}
```
- **Responses**:
  - **200 OK**:
  ```json
  {
    "success": true,
    "message": "Account resolved successfully",
    "data": {
      "account_number": "0123456789",
      "account_name": "OLUWASEUN ADEBAYO"
    }
  }
  ```

### 5.4 Request Payout (Withdrawal)
- **Method**: `POST`
- **Path**: `/withdraw`
- **Authentication**: JWT + KYC verified + TrustCheck
- **Required Header**: `Idempotency-Key: <uuid>`
- **Request Body**:
```json
{
  "amount_kobo": 300000000,
  "bank_code": "058",
  "account_number": "0123456789",
  "account_name": "OLUWASEUN ADEBAYO"
}
```
- **Responses**:
  - **200 OK**:
  ```json
  {
    "success": true,
    "message": "Withdrawal queued for processing",
    "data": {
      "id": "with-uuid-goes-here",
      "amount_kobo": 300000000,
      "status": "pending"
    }
  }
  ```
  - **409 Conflict**: Duplicate `Idempotency-Key`
  - **403 Forbidden**: Wallet frozen, trust block, or KYC insufficient

---

## 5A. Payments — `/api/v1/payments`

### 5A.1 Initialize Payment (Production Funding Path)
- **Method**: `POST`
- **Path**: `/initialize`
- **Authentication**: JWT (Buyer) + TrustCheck
- **Required Header**: `Idempotency-Key: <uuid>`
- **Request Body**:
```json
{
  "escrow_id": "a11b4fa4-7e8c-4a3b-8d9e-1f2a3b4c5d6e",
  "callback_url": "https://app.vescrow.ng/payments/callback"
}
```
- **Responses**:
  - **200 OK**:
  ```json
  {
    "success": true,
    "message": "Payment initialized successfully",
    "data": {
      "authorization_url": "https://checkout.paystack.com/abcdefgh12345",
      "reference": "VESC_REF_9876543210"
    }
  }
  ```
  - **409 Conflict**: Duplicate idempotency key

### 5A.2 Verify Payment
- **Method**: `GET`
- **Path**: `/verify/:reference`
- **Authentication**: JWT
- **Responses**:
  - **200 OK**:
  ```json
  {
    "success": true,
    "message": "Payment status retrieved",
    "data": {
      "status": "confirmed",
      "reference": "VESC_REF_9876543210",
      "amount_kobo": 36050000,
      "paid_at": "2026-06-14T17:25:00Z"
    }
  }
  ```

---

## 5B. Listings — `/api/v1/listings`

### 5B.1 Get Public Listing
- **Method**: `GET`
- **Path**: `/:token/public`
- **Authentication**: None
- **Responses**: Enriched public listing detail for the product page:
```json
{
  "title": "MacBook Pro M3",
  "description": "Factory sealed, 1-year warranty, includes charger and box.",
  "images": ["http://localhost:3000/api/v1/listings/media/abc123.jpg"],
  "amount_kobo": 350000000,
  "currency": "NGN",
  "quantity_total": 10,
  "quantity_available": 8,
  "status": "active",
  "seller_name": "Ada O.",
  "share_token": "abc123token",
  "inspection_period_days": 3,
  "incoming_fee_kobo": 10500000,
  "buyer_total_kobo": 360500000,
  "outgoing_fee_kobo": 10500000,
  "seller_receives_kobo": 339500000,
  "created_at": "2026-08-23T10:00:00Z"
}
```

### 5B.2 Upload Listing Image
- **Method**: `POST`
- **Path**: `/images`
- **Authentication**: JWT + KYC verified
- **Content-Type**: `multipart/form-data` (field name: `image`)
- **Constraints**: JPEG, PNG, or WebP; max 5 MB
- **Responses**: `{ "url": "https://..." }` — public URL to include in create payload
- **Storage**: Local disk when `LISTING_MEDIA_PROVIDER=local` (default); Cloudinary when configured

### 5B.3 Serve Listing Media (local provider only)
- **Method**: `GET`
- **Path**: `/media/*`
- **Authentication**: None
- **Responses**: Binary image file (path traversal blocked)

### 5B.4 Create Listing
- **Method**: `POST`
- **Path**: `/`
- **Authentication**: JWT + KYC verified + TrustCheck
- **Request Body**:
```json
{
  "title": "MacBook Pro M3 — Limited Stock",
  "description": "Factory sealed, 1-year warranty. Includes charger and original box.",
  "images": [
    "http://localhost:3000/api/v1/listings/media/abc123.jpg"
  ],
  "amount_kobo": 350000000,
  "currency": "NGN",
  "creator_role": "seller",
  "quantity_total": 10,
  "inspection_period_days": 3
}
```
- **Validation**: `description` required, 50–5000 chars; `images` required, 1–5 URLs from prior upload

### 5B.5 List My Listings
- **Method**: `GET`
- **Path**: `/`
- **Authentication**: JWT

### 5B.6 Claim Listing (Buyer)
- **Method**: `POST`
- **Path**: `/:token/claim`
- **Authentication**: JWT + KYC verified + TrustCheck
- **Responses**: Creates draft escrow + payment reservation; returns `escrow_id`.

### 5B.7 Get Listing Orders
- **Method**: `GET`
- **Path**: `/:id/orders`
- **Authentication**: JWT (listing creator)

---

## 5C. Notifications — `/api/v1/notifications`

### 5C.1 List Notifications
- **Method**: `GET`
- **Path**: `/`
- **Authentication**: JWT
- **Responses**: `{ notifications: [...], unread_count: N }`

### 5C.2 Mark Notification Read
- **Method**: `PUT`
- **Path**: `/:id/read`
- **Authentication**: JWT

### 5C.3 Mark All Read
- **Method**: `PUT`
- **Path**: `/read-all`
- **Authentication**: JWT

---

## 5D. Trust — `/api/v1`

### 5D.1 Get My Trust Profile
- **Method**: `GET`
- **Path**: `/trust/profile`
- **Authentication**: JWT

### 5D.2 Submit Appeal
- **Method**: `POST`
- **Path**: `/appeals`
- **Authentication**: JWT + TrustCheck
- **Request Body**:
```json
{
  "block_id": "block-uuid",
  "reason": "This block was applied in error"
}
```

### 5D.3 List My Appeals
- **Method**: `GET`
- **Path**: `/appeals/mine`
- **Authentication**: JWT

---

## 6. Disputes — `/api/v1/disputes`

### 6.1 Raise Dispute
- **Method**: `POST`
- **Path**: `/`
- **Authentication**: JWT (Buyer)
- **Request Body**:
```json
{
  "escrow_id": "a11b4fa4-7e8c-4a3b-8d9e-1f2a3b4c5d6e",
  "reason": "defect_discovered",
  "description": "The laptop screen has a major crack on the bottom left corner, which was not disclosed in the listing details."
}
```
- **Responses**:
  - **201 Created**:
  ```json
  {
    "success": true,
    "message": "Dispute opened successfully. Escrow funds locked.",
    "data": {
      "id": "disp-uuid-goes-here",
      "status": "open",
      "opened_at": "2026-06-16T15:30:00Z"
    }
  }
  ```

### 6.2 Upload Dispute Evidence
- **Method**: `POST`
- **Path**: `/:id/evidence`
- **Authentication**: JWT
- **Request Body**: Multipart Form Data
  - `evidence_file`: [File Upload] (Image/PDF)
  - `description`: "Photo of the screen crack details."
- **Responses**:
  - **200 OK**:
  ```json
  {
    "success": true,
    "message": "Evidence uploaded successfully",
    "data": {
      "file_url": "https://s3.amazonaws.com/vescrow/evidence/crack.jpg"
    }
  }
  ```

### 6.3 Submit Seller Response
- **Method**: `POST`
- **Path**: `/:id/respond`
- **Authentication**: JWT (Seller)
- **Request Body**:
```json
{
  "response": "The laptop was packaged extremely well and did not have any crack when shipped. It must have occurred during delivery with GIG logistics."
}
```
- **Responses**:
  - **200 OK**:
  ```json
  {
    "success": true,
    "message": "Response submitted. Escrow updated to review status.",
    "data": {
      "status": "under_review"
    }
  }
  ```

---

## 7. Webhooks — `/api/v1/payments`

### 7.1 Paystack Webhook Handler
- **Method**: `POST`
- **Path**: `/webhook/paystack`
- **Authentication**: Signature verification via `X-Paystack-Signature` header.
- **Payload examples**:
  - **Charge Success**:
  ```json
  {
    "event": "charge.success",
    "data": {
      "reference": "VESC_REF_9876543210",
      "amount": 36050000,
      "status": "success",
      "metadata": {
        "escrow_id": "a11b4fa4-7e8c-4a3b-8d9e-1f2a3b4c5d6e"
      }
    }
  }
  ```
  - **Transfer Success**:
  ```json
  {
    "event": "transfer.success",
    "data": {
      "transfer_code": "TRF_123456789abcdef",
      "amount": 30000000,
      "status": "success",
      "reference": "with-uuid-goes-here"
    }
  }
  ```
- **Responses**:
  - **200 OK**: Required by Paystack to acknowledge receipt.
  ```json
  {
    "success": true,
    "message": "Webhook processed successfully"
  }
  ```
  - **500 Internal Server Error**: Processing failed; payload persisted to `webhook_failures` for manual replay (triggers Paystack retry).

---

## 8. Admin Operations — `/api/v1/admin`

*All endpoints require `admin` or `super_admin` role. In production/staging, admin accounts with 2FA enabled must include `X-TOTP-Code`.*

### 8.1 List KYC Submissions
- **Method**: `GET`
- **Path**: `/kyc/submissions`
- **Authentication**: JWT (Admin) + AdminMFA

### 8.2 Get KYC Submission
- **Method**: `GET`
- **Path**: `/kyc/submissions/:id`
- **Authentication**: JWT (Admin) + AdminMFA

### 8.3 Approve KYC Submission
- **Method**: `PUT`
- **Path**: `/kyc/submissions/:id/approve`
- **Authentication**: JWT (Admin) + AdminMFA
- **Responses**:
  - **200 OK**:
  ```json
  {
    "success": true,
    "message": "KYC document approved",
    "data": {
      "kyc_status": "approved",
      "status": "approved"
    }
  }
  ```

### 8.4 Reject KYC Submission
- **Method**: `PUT`
- **Path**: `/kyc/submissions/:id/reject`
- **Authentication**: JWT (Admin) + AdminMFA

### 8.5 List Audit Logs
- **Method**: `GET`
- **Path**: `/audit-logs`
- **Authentication**: JWT (Admin) + AdminMFA + `view_audit` permission
- **Query Parameters**: `page` (default 1), `per_page` (default 50, max 100)
- **Responses**:
  - **200 OK**:
  ```json
  {
    "success": true,
    "message": "Audit logs retrieved",
    "data": {
      "logs": [],
      "meta": {
        "page": 1,
        "per_page": 50,
        "total": 0,
        "total_pages": 0
      }
    }
  }
  ```

### 8.6 Review Trust Appeal
- **Method**: `PUT`
- **Path**: `/appeals/:id/review` *(registered at `/api/v1/admin/appeals/:id/review`)*
- **Authentication**: JWT (Admin)
- **Request Body**:
```json
{
  "approve": true,
  "note": "Block lifted after identity re-verification"
}
```

### 8.7 Resolve Dispute
- **Method**: `PUT`
- **Path**: `/disputes/:id/resolve`
- **Authentication**: JWT (Admin)
- **Request Body**:
```json
{
  "resolution": "partial_split",
  "buyer_amount_kobo": 175000000,
  "seller_amount_kobo": 175000000,
  "notes": "Both parties agreed to split the escrow 50/50 since the screen crack repairs will cost ₦175,000."
}
```
- **Responses**:
  - **200 OK**:
  ```json
  {
    "success": true,
    "message": "Dispute resolved and ledger entries written successfully",
    "data": {
      "status": "resolved",
      "resolution": "partial_split"
    }
  }
  ```

---

## 9. Health Check

- **Method**: `GET`
- **Path**: `/api/v1/health`
- **Authentication**: None
- **Responses**:
  ```json
  {
    "status": "healthy",
    "timestamp": "2026-08-23T14:00:00+01:00"
  }
  ```

---

**Document Version**: 2.0  
**Last Updated**: August 2026
