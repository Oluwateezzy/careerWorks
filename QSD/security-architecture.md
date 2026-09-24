# Vescrow — Security Architecture

**Version**: 2.0  
**Last Updated**: August 2026  
**Audience**: Security engineers, backend developers, compliance reviewers, enterprise customers  
**Related docs**: [architecture.md](./architecture.md) · [api-specification.md](./api-specification.md) · [database-schema.md](./database-schema.md)

---

## 1. Purpose & Scope

This document describes the **as-built** security architecture of the Vescrow platform — a Nigerian escrow service for B2B and consumer transactions. It covers authentication, authorization, financial integrity, fraud prevention, data protection, and operational security controls implemented in the Go backend and Next.js frontend.

Vescrow follows a **defence-in-depth** model: no single control is relied upon to protect money or identity. Each layer assumes the layer above may fail.

---

## 2. System Context

```
┌──────────────────────────────────────────────────────────────────────────┐
│  Browser / Mobile Web (Next.js :3001)                                     │
│  • Same-origin API proxy (/api/v1 → backend)                              │
│  • Access token in memory; refresh via HttpOnly cookie                    │
│  • Security headers (HSTS, X-Frame-Options, etc.)                         │
└───────────────────────────────┬──────────────────────────────────────────┘
                                │ HTTPS (TLS 1.3 in production)
                                ▼
┌──────────────────────────────────────────────────────────────────────────┐
│  Vescrow API (Go/Fiber :3000)                                             │
│  SecurityHeaders → CORS → RateLimit → APIKeyAuth → Auth → TrustCheck …  │
└───────────────┬──────────────────────────────┬───────────────────────────┘
                │                              │
         ┌──────┴──────┐                ┌──────┴──────┐
         │ PostgreSQL  │                │    Redis     │
         │ (ledger,    │                │ OTP, sessions│
         │  audit)     │                │ rate limits, │
         └─────────────┘                │ idempotency  │
                                        └─────────────┘
                │
         ┌──────┴──────┐
         │  Paystack   │  Webhooks (HMAC-SHA512)
         │  KYC prov.  │  Outbound transfers
         └─────────────┘
```

---

## 3. Security Layers

| Layer | Controls |
|-------|----------|
| **Transport** | TLS 1.3 (production), CORS whitelist via `CORS_ORIGINS`, security response headers |
| **Edge / App** | Redis-backed rate limiting (global, auth, financial), 10 MB body limit |
| **Authentication** | RS256 JWT (15 min access), HttpOnly cookies, Redis refresh session revocation |
| **B2B Auth** | Organization-scoped API keys (`X-API-Key`, prefix + Argon2id hash) |
| **Authorization** | RBAC (`user` / `admin` / `super_admin`), permission middleware, resource ownership |
| **Trust & Fraud** | Trust score, block lists (user/NIN/IP/device), `TrustCheck` on money paths |
| **KYC Gate** | Verified KYC required for escrow create, listings, withdrawals |
| **Financial** | Double-entry ledger, idempotency keys, payout outbox, daily reconciliation |
| **Data** | AES-256-GCM for BVN/NIN, parameterized queries, PII masking in responses |
| **Audit** | Append-only audit logs, 5-year retention target, admin audit API |
| **Admin** | TOTP MFA (`X-TOTP-Code`) enforced in non-dev environments when 2FA enabled |

Detailed runbooks: [docs/security/runbooks.md](./security/runbooks.md)

---

## 4. Authentication & Session Management

### 4.1 Token Model

| Token | Lifetime | Storage | Revocation |
|-------|----------|---------|------------|
| Access JWT | 15 minutes | HttpOnly `access_token` cookie **and** JSON body (SPA compatibility) | Expires naturally; not stored server-side |
| Refresh JWT | 30 days | HttpOnly `refresh_token` cookie | Redis session key per JTI; revoked on logout |

The auth middleware accepts tokens from either `Authorization: Bearer <token>` or the `access_token` cookie.

### 4.2 Login & Refresh Flow

1. `POST /api/v1/auth/login` — validates credentials (Argon2id), writes audit log, sets both cookies, returns `{ access_token, authenticated }`.
2. `POST /api/v1/auth/refresh` — validates refresh cookie against Redis session store, rotates refresh JTI, issues new access token.
3. `POST /api/v1/auth/logout` — revokes Redis refresh session, clears cookies (does not require valid access token).
4. `GET /api/v1/auth/session` — lightweight session validation for frontend boot.

### 4.3 Frontend Session Strategy

The Next.js client stores the access token **in memory only** (not `localStorage`). API calls go through a same-origin proxy (`/api/v1` → backend) so HttpOnly cookies are sent automatically. User profile hints may be cached in `sessionStorage` for UX; tokens are never persisted there.

### 4.4 Password & OTP

- Passwords hashed with **Argon2id** (never returned in API responses).
- Email/phone OTP stored in Redis with TTL; rate-limited auth endpoints (5 req / 15 min per IP).
- No hardcoded or bypass OTP values in production UI or backend.

---

## 5. Authorization

### 5.1 Role-Based Access Control

| Role | Capabilities |
|------|-------------|
| `user` | Own escrows, wallet, KYC, listings |
| `admin` | KYC review, dispute resolution, appeal review, audit logs (with `view_audit` permission) |
| `super_admin` | All admin permissions + platform settings |

Permissions are stored on the `roles` table as a comma-separated string and enforced via `RequirePermission()` middleware.

### 5.2 Resource Ownership

Escrow, wallet, and dispute handlers validate that the authenticated user is the buyer, seller, or an admin before mutating state.

### 5.3 B2B Organization Model

| Entity | Purpose |
|--------|---------|
| `organizations` | B2B tenant |
| `members` | User ↔ org membership with role (`owner`, `admin`, `operator`, `viewer`) |
| `api_keys` | Hashed keys with prefix lookup, scopes, optional expiry/revocation |

API keys are validated globally via `APIKeyAuth` middleware. When present, `organizationID` and `apiKeyScopes` are set in request context. Keys use format `{prefix}.{secret}`; only the hash is stored.

Users and escrows may carry an optional `organization_id` for tenant scoping.

### 5.4 Admin MFA

In `production` and `staging` (`ENVIRONMENT` ≠ `development` / `test`), admin routes require `X-TOTP-Code` when the admin account has `is_2fa_enabled = true` and a configured `totp_secret`.

---

## 6. Financial Security

### 6.1 Double-Entry Ledger

All money movement goes through `WalletService.RecordEntry()` — debits and credits with immutable `ledger_entries`. Direct balance mutation outside the ledger path is prohibited.

Escrow funding via Paystack webhook uses `WalletService.FundEscrow()` — the same ledger path as internal transfers.

### 6.2 Idempotency

Money-mutating endpoints require an `Idempotency-Key` header:

- `POST /api/v1/payments/initialize`
- `POST /api/v1/wallet/withdraw`

Keys are stored in Redis (24-hour window). Duplicate keys return `409 Conflict`.

Payment records also carry a database-level `idempotency_key` unique constraint. Webhook processing checks `provider_reference` uniqueness before crediting.

### 6.3 Async Payout Outbox

Withdrawals enqueue a `payout_outbox` entry processed by a background worker. This decouples user-facing API latency from Paystack transfer reliability and prevents double-payout on retries (status tracked per entry).

### 6.4 Daily Reconciliation

A reconciliation worker compares Paystack settlements, `escrow_payments`, and ledger totals. Mismatches are logged for manual investigation — balances are never silently adjusted.

### 6.5 Dev-Only Endpoints

`POST /api/v1/escrows/:id/fund` is gated with `DevOnly()` middleware. **Production funding must use** `POST /api/v1/payments/initialize` + Paystack checkout + webhook confirmation.

### 6.6 Withdrawal Controls

- KYC verified (`middleware.KYCVerified`)
- Trust check passed (`middleware.TrustCheck`)
- Wallet not frozen (`is_frozen`)
- Platform-configured daily/per-tx limits and cool-down periods

---

## 7. Trust & Fraud Prevention

### 7.1 Trust Profile

Each user has a `trust_profiles` row (score 0–100, status). Events (disputes, chargebacks, policy violations) adjust score via `trust_events`.

### 7.2 Block List

`block_entries` supports blocking by user ID, NIN hash, IP address, or device fingerprint (`X-Device-FP` header). Blocks may expire or be permanent.

### 7.3 TrustCheck Middleware

Applied on: escrow create, payment initialize, wallet withdraw, listing create/claim.

On block: returns `403` with `code: ACCOUNT_BLOCKED` and appeal URL hint.

### 7.4 Appeals

Users submit appeals via `POST /api/v1/appeals`. Admins review via `PUT /api/v1/admin/appeals/:id/review`.

---

## 8. KYC & Sensitive Data

### 8.1 KYC Provider Abstraction

KYC verification is routed through `internal/kycprovider/` based on `KYC_PROVIDER` env (e.g. Youverify, Smile ID, mock for dev). Document uploads use secure storage (`internal/storage/kyc_upload.go`).

### 8.2 KYC Status Gate

`middleware.KYCVerified` requires `kyc_status` of `verified` or `approved` before:

- Creating escrows or listings
- Requesting withdrawals

> **Note**: The legacy `kyc_tier` column has been removed. Access is gated on `kyc_status` only.

### 8.3 Encryption & Masking

| Field | At Rest | In API Response |
|-------|---------|-----------------|
| Password | Argon2id hash | Never returned |
| BVN / NIN | AES-256-GCM | Masked (`****5678`) |
| Bank account | AES-256-GCM | Masked |
| TOTP secret | Plaintext in DB (admin-only column) | Never returned |
| API key | Argon2id hash | Only shown once at creation |

Key rotation procedures: [docs/security/key-rotation.md](./security/key-rotation.md)

---

## 9. Webhook Security

### 9.1 Paystack Webhooks

- Endpoint: `POST /api/v1/payments/webhook/paystack`
- Signature verified via `X-Paystack-Signature` (HMAC-SHA512 with webhook secret)
- Idempotent processing by `provider_reference`
- On processing failure: returns **500** (triggers Paystack retry) and persists payload to `webhook_failures` for manual replay

### 9.2 Failure Handling

Failed webhooks are queryable from `webhook_failures`. Operators follow the webhook runbook before manual replay.

---

## 10. Audit & Compliance

### 10.1 Audit Logs

All significant state transitions write to `audit_logs` (append-only):

- Actor ID, type (`user` / `admin` / `system`)
- Action, resource type/ID
- Old/new state JSON snapshots
- IP address, user agent

Admin access: `GET /api/v1/admin/audit-logs` (requires `view_audit` permission + admin MFA when enabled).

### 10.2 Compliance Artifacts

| Document | Purpose |
|----------|---------|
| [soc2-checklist.md](./security/soc2-checklist.md) | SOC 2 Type II readiness checklist |
| [pen-test-scope.md](./security/pen-test-scope.md) | Penetration test scope definition |
| [runbooks.md](./security/runbooks.md) | Incident response procedures |
| [key-rotation.md](./security/key-rotation.md) | Key and credential rotation |

---

## 11. Infrastructure & Deployment Security

### 11.1 Environment Variables

Sensitive configuration is loaded from environment (never committed):

- `JWT_PRIVATE_KEY_PATH` / `JWT_PUBLIC_KEY_PATH`
- `PAYSTACK_SECRET_KEY`, `PAYSTACK_WEBHOOK_SECRET`
- `ENCRYPTION_KEY` (AES for PII)
- `REDIS_URL`, `DATABASE_URL`
- `CORS_ORIGINS`, `ENVIRONMENT`
- `KYC_PROVIDER`, `KYC_PROVIDER_API_KEY`

### 11.2 Swagger

OpenAPI/Swagger UI is **disabled** when `ENVIRONMENT` is not `development` or `test`.

### 11.3 Docker

Production images use multi-stage builds (Alpine). Non-root execution recommended for deployment.

### 11.4 CORS

Origins are explicitly whitelisted via `CORS_ORIGINS`. Credentials (`AllowCredentials: true`) enabled for cookie-based auth.

Allowed headers include: `Authorization`, `X-Device-FP`, `X-TOTP-Code`, `Idempotency-Key`, `X-API-Key`.

---

## 12. Background Workers (Security-Relevant)

| Worker | Security Role |
|--------|--------------|
| Auto-release | Time-bound fund release after inspection period |
| Pending acceptance expiry | Prevents stale open escrows |
| Pre-release reminders | User notification before auto-release |
| Listing reservation expiry | Releases held inventory slots |
| Payout outbox | Reliable, retry-safe bank transfers |
| Daily reconciliation | Detects ledger/payment drift |

Workers run in-process within the monolith. Future decomposition should preserve exactly-once payout semantics.

---

## 13. Threat Model Summary

| Threat | Mitigation |
|--------|-----------|
| Stolen access token | Short TTL (15 min); refresh rotation; logout revokes session |
| XSS token theft | HttpOnly cookies; in-memory access token on frontend |
| Payment bypass | Dev-only direct fund; production requires Paystack webhook + ledger |
| Double spend / replay | Idempotency keys; unique payment references; outbox status tracking |
| Webhook spoofing | HMAC signature verification |
| Privilege escalation | RBAC + permission middleware; admin MFA |
| API key leak | Revocation, scoped keys, audit trail |
| Account takeover | OTP verification, trust blocks, wallet freeze runbook |
| SQL injection | GORM parameterized queries |
| Brute force login | Rate limiting on auth endpoints |

Pen test scope: [docs/security/pen-test-scope.md](./security/pen-test-scope.md)

---

## 14. Security Roadmap (Outstanding)

| Item | Status |
|------|--------|
| Organization HTTP API routes (key CRUD) | Service layer complete; HTTP routes pending |
| Real KYC provider credentials (Youverify/Smile ID) | Abstraction ready; env config needed |
| Admin audit log UI | API ready; frontend UI pending |
| WAF / DDoS (production) | Platform-level (e.g. Vercel Firewall / Cloudflare) |
| Secrets manager integration | Env vars today; vault recommended for production |

---

**Document Version**: 2.0  
**Status**: As-built (August 2026)  
**Next Review**: Before production launch or after major auth/payment changes
