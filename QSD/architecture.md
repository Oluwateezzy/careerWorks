# Vescrow — System Architecture Document

**Version**: 2.0  
**Last Updated**: August 2026  
**Author**: Vescrow Engineering Team  
**Audience**: Backend Developers, Solutions Architects, DevOps Engineers

---

## 1. System Overview

Vescrow is a **modular monolithic** Go application providing escrow services for the Nigerian market. It manages the full lifecycle of buyer-seller transactions — from creation through payment, delivery, and fund release — with built-in dispute resolution and regulatory compliance.

### 1.1 Design Philosophy

| Principle | Application |
|-----------|-------------|
| **Modular Monolith** | Single binary with domain-separated modules for easy future decomposition |
| **Domain-Driven Design** | Code organized by business domain, not technical layer |
| **Dependency Inversion** | Repository interfaces defined alongside services, implementations separate |
| **Defence in Depth** | Multiple security layers: transport, auth, authorization, data, audit |
| **Financial Integrity** | Double-entry bookkeeping ensures every naira is accounted for |

### 1.2 Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         CLIENT LAYER (:3001)                             │
│   Next.js (mobile-first) — same-origin proxy /api/v1 → backend :3000    │
│   Access token in memory • HttpOnly cookies for refresh/access           │
└───────────────────────────┬─────────────────────────────────────────────┘
                            │ HTTPS (TLS 1.3 in production)
                            ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                     VESCROW APPLICATION (:3000)                          │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │                   MIDDLEWARE PIPELINE                               │  │
│  │  Recover → Logger → SecurityHeaders → CORS → APIKeyAuth →           │  │
│  │  RateLimit → Auth (JWT/cookie) → KYC Gate → TrustCheck →            │  │
│  │  Idempotency (money) → AdminMFA → Permissions → Handler             │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  ┌────────┐ ┌────────┐ ┌─────────┐ ┌────────┐ ┌─────────┐ ┌────────┐ │
│  │  Auth  │ │ Escrow │ │ Listing │ │ Payment│ │ Wallet  │ │ Dispute│ │
│  │  KYC   │ │        │ │         │ │ Webhook│ │ Ledger  │ │        │ │
│  └────────┘ └────────┘ └─────────┘ └────────┘ └─────────┘ └────────┘ │
│  ┌────────┐ ┌────────┐ ┌─────────┐ ┌────────┐ ┌─────────┐            │
│  │ Trust  │ │ Inbox  │ │  Payout │ │ Recon  │ │   Org   │            │
│  │ Appeals│ │ Notif. │ │  Outbox │ │ Worker │ │ B2B Keys│            │
│  └────────┘ └────────┘ └─────────┘ └────────┘ └─────────┘            │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │  Notification • Audit • Admin (KYC, audit-logs, appeals)            │  │
│  └────────────────────────────────────────────────────────────────────┘  │
└────────────────────┬────────────────────┬───────────────────────────────┘
                     │                    │
              ┌──────┴──────┐      ┌──────┴──────┐
              │ PostgreSQL  │      │    Redis     │
              │ 25+ tables  │      │ OTP, sessions│
              │ ledger      │      │ rate limits  │
              │ audit       │      │ idempotency  │
              └─────────────┘      └──────────────┘
```

---

## 2. Module Architecture

### 2.1 Module Dependency Map

```
                    ┌──────────┐
                    │  common  │ ◄──── Used by ALL modules
                    └──────────┘
                         ▲
    ┌────────────────────┼──────────────────────────────────┐
    │          │         │         │         │              │
┌───┴───┐ ┌───┴────┐ ┌───┴────┐ ┌──┴─────┐ ┌─┴──────┐ ┌────┴────┐
│ auth  │ │ escrow │ │listing │ │payment │ │ wallet │ │ dispute │
└───────┘ └───┬────┘ └───┬────┘ └────┬───┘ └────┬───┘ └────┬────┘
              │          │           │          │          │
              └──────────┴─────┬─────┴──────────┴──────────┘
                               │
              ┌────────────────┼────────────────┐
              │                │                │
         ┌────┴─────┐   ┌─────┴─────┐   ┌─────┴─────┐
         │  trust   │   │ notific.  │   │   audit   │
         └──────────┘   └─────┬─────┘   └─────┬─────┘
                              │               │
                         ┌────┴─────┐    ┌────┴─────┐
                         │  inbox   │    │  admin   │
                         └──────────┘    └────┬─────┘
                                              │
                         ┌──────────┐  ┌─────┴─────┐  ┌──────────────┐
                         │  payout  │  │    org    │  │reconciliation│
                         │  outbox  │  │  (B2B)    │  │   (daily)    │
                         └──────────┘  └───────────┘  └──────────────┘
```

### 2.2 Module Responsibilities

#### Internal Flow: `handler.go → service.go → repository.go → model.go`

Each domain module follows a consistent 4-layer pattern:

| Layer | File | Responsibility |
|-------|------|---------------|
| **Model** | `model.go` | GORM struct definitions, validation tags |
| **Repository** | `repository.go` | Database queries (GORM), no business logic |
| **Service** | `service.go` | Business logic, orchestration, calls repositories |
| **Handler** | `handler.go` | HTTP request/response, input parsing, calls services |
| **Routes** | `routes.go` | Fiber route registration with middleware |

Each domain module follows this 4-layer pattern unless noted (e.g. `payout/` and `reconciliation/` are worker-only).

| Module | Path | Key Responsibilities |
|--------|------|---------------------|
| `auth/` | Identity | Register, login, OTP, JWT sessions, profile, KYC submission |
| `escrow/` | Core | Escrow lifecycle, milestones, in-escrow messaging |
| `listing/` | Growth | Shareable product listings, claim flow, payment reservations |
| `payment/` | Money In | Paystack initialize/verify, webhook processing, failure logging |
| `wallet/` | Money | Balances, ledger, withdrawals, bank verification |
| `payout/` | Money Out | Async outbox worker for Paystack transfers |
| `dispute/` | Trust | Dispute cases, evidence, admin resolution |
| `trust/` | Trust | Trust scores, blocks, appeals, device tracking |
| `notification/` | Comms | Email (SendGrid), SMS (Twilio) |
| `notification/inbox/` | Comms | In-app notification inbox |
| `audit/` | Compliance | Immutable activity trail |
| `admin/` | Ops | KYC review, audit log API |
| `organization/` | B2B | Tenants, members, API keys (service layer) |
| `reconciliation/` | Finance | Daily ledger vs payment reconciliation |
| `kycprovider/` | Identity | Pluggable KYC provider abstraction |
| `middleware/` | Security | Auth, rate limit, trust, idempotency, MFA, API keys |

---

## 3. Data Flow Patterns

### 3.1 Escrow Creation & Funding Flow

```
Buyer                    Vescrow API                    Paystack
  │                          │                             │
  │  POST /escrows           │                             │
  │────────────────────────►│                             │
  │                          │ Validate + Create Draft     │
  │  201 { escrow }         │                             │
  │◄────────────────────────│                             │
  │                          │                             │
  │  PUT /escrows/:id/submit│                             │
  │────────────────────────►│                             │
  │                          │ → Notify Seller (email)     │
  │  200 OK                 │                             │
  │◄────────────────────────│                             │
  │                          │                             │
  │         [Seller accepts via PUT /escrows/:id/accept]   │
  │                          │                             │
  │  POST /payments/initialize                               │
  │  Idempotency-Key: <uuid>                                │
  │────────────────────────►│                             │
  │                          │  POST /transaction/init     │
  │                          │────────────────────────────►│
  │                          │  { authorization_url }      │
  │                          │◄────────────────────────────│
  │  200 { auth_url }       │                             │
  │◄────────────────────────│                             │
  │                          │                             │
  │  [User pays on Paystack]│                             │
  │                          │                             │
  │                          │  POST /webhook/paystack     │
  │                          │◄────────────────────────────│
  │                          │ Verify HMAC signature        │
  │                          │ Check idempotency            │
  │                          │ WalletService.FundEscrow()   │
  │                          │ Update payment → confirmed   │
  │                          │ Update escrow → funded       │
  │                          │ Create audit log             │
  │                          │ Notify both parties          │
  │                          │  200 OK (500 on failure →    │
  │                          │   webhook_failures table)    │
  │                          │────────────────────────────►│
```

### 3.2 Fund Release Flow

```
Buyer                    Vescrow API                    Wallet Module
  │                          │                             │
  │  PUT /escrows/:id/      │                             │
  │      approve-delivery   │                             │
  │────────────────────────►│                             │
  │                          │ Validate: buyer, status     │
  │                          │                             │
  │                          │ Calculate fee (3%)           │
  │                          │ amount = 1,000,000 kobo     │
  │                          │ fee    =    30,000 kobo     │
  │                          │ seller =   970,000 kobo     │
  │                          │                             │
  │                          │  BEGIN TRANSACTION           │
  │                          │────────────────────────────►│
  │                          │  Debit escrow wallet         │
  │                          │  Credit seller wallet        │
  │                          │  Credit platform_fee wallet  │
  │                          │  COMMIT                      │
  │                          │◄────────────────────────────│
  │                          │                             │
  │                          │ Update escrow → completed    │
  │                          │ Create audit log             │
  │                          │ Notify both parties          │
  │  200 OK                 │                             │
  │◄────────────────────────│                             │
```

### 3.3 Withdrawal Flow

```
Seller                   Vescrow API                    Paystack
  │                          │                             │
  │  POST /wallet/withdraw  │                             │
  │  Idempotency-Key: <uuid>│                             │
  │  { amount, bank_code,   │                             │
  │    account_number }     │                             │
  │────────────────────────►│                             │
  │                          │ Validate: KYC, trust,       │
  │                          │   balance, limits, frozen   │
  │                          │                             │
  │                          │ Debit seller wallet          │
  │                          │ Enqueue payout_outbox        │
  │                          │                             │
  │  202 { withdrawal }     │                             │
  │◄────────────────────────│                             │
  │                          │                             │
  │                          │ [Outbox worker → transfer]   │
  │                          │  POST /transfer              │
  │                          │────────────────────────────►│
  │                          │  { transfer_code, status }   │
  │                          │◄────────────────────────────│
  │                          │                             │
  │  200 { withdrawal }     │                             │
  │◄────────────────────────│                             │
  │                          │                             │
  │                          │ [Webhook: transfer.success]  │
  │                          │◄────────────────────────────│
  │                          │ Update withdrawal → complete │
  │                          │ Notify seller (email + SMS)  │
```

---

## 4. Security Architecture

> Full security design, threat model, and compliance references: **[security-architecture.md](./security-architecture.md)**

### 4.1 Security Layers (Summary)

```
Layer 1: TRANSPORT SECURITY
├── TLS 1.3 (production)
└── CORS origin whitelist

Layer 2: RATE LIMITING
├── Per-IP: 100 req/min (general)
├── Per-IP: 5 req/15min (login)
└── Per-User: 10 req/min (financial operations)

Layer 3: AUTHENTICATION
├── RS256 JWT (access token, 15 min)
├── HttpOnly cookies (access + refresh tokens)
├── Redis refresh session revocation
├── OTP verification (email + phone)
└── Argon2id password hashing

Layer 4: AUTHORIZATION
├── Role-Based Access Control (user/admin/super_admin)
├── Permission middleware (e.g. view_audit)
├── B2B API keys (X-API-Key, org-scoped)
├── KYC status gate (verified/approved)
├── TrustCheck on money paths
└── Resource ownership validation

Layer 5: DATA SECURITY
├── AES-256-GCM encryption (BVN, NIN, bank accounts)
├── GORM parameterized queries (SQL injection prevention)
├── API response PII masking
└── Webhook HMAC-SHA512 signature verification

Layer 6: FINANCIAL INTEGRITY
├── Double-entry bookkeeping (WalletService)
├── Idempotency keys (Redis + DB unique constraints)
├── Payout outbox (async transfers)
├── Daily reconciliation worker
├── Transaction atomicity (database transactions)
└── Withdrawal limits + cool-down periods

Layer 7: AUDIT & COMPLIANCE
├── Immutable audit trail (append-only, 5-year retention)
├── Admin audit log API
├── Webhook failure persistence
└── Admin MFA (TOTP) in production
```

### 4.2 JWT Token Flow

```
                  ┌──────────────┐
                  │    Login     │
                  └──────┬───────┘
                         │
                    ┌────▼────┐
                    │ Verify  │
                    │Password │
                    └────┬────┘
                         │
              ┌──────────┴──────────┐
              ▼                     ▼
     ┌───────────────┐    ┌────────────────┐
     │ Access Token  │    │ Refresh Token  │
     │ RS256, 15 min │    │ RS256, 30 days │
     │ (HttpOnly     │    │ (HttpOnly      │
     │  cookie +     │    │  cookie)       │
     │  JSON body)   │    │ Redis session  │
     └───────┬───────┘    └────────┬───────┘
             │                     │
             ▼                     │
     ┌───────────────┐             │
     │  API Requests │             │
     │  Authorization│             │
     │  Bearer xxx   │             │
     └───────┬───────┘             │
             │                     │
             │ Expired?            │
             ▼                     ▼
     ┌───────────────┐    ┌────────────────┐
     │  401 Returned │───►│ POST /refresh  │
     │               │    │ (auto via      │
     │               │    │  client)       │
     └───────────────┘    └────────────────┘
```

---

## 5. Database Architecture

### 5.1 Single Database Strategy

All domain tables reside in a single PostgreSQL instance (`vescrow_db`). This simplifies:
- Transaction management (ACID across all domains)
- Join queries (admin dashboard, reports)
- Backup and recovery
- Schema migrations

### 5.2 Table Relationships

```
users ─┬─< kyc_documents
       ├─< trust_profiles / trust_events / appeals
       ├─< escrow_listings (as creator)
       ├─< escrows (as buyer_id / seller_id)
       ├─< wallets → ledger_entries, withdrawals
       ├─< messages, disputes, audit_logs
       └─> organizations (optional organization_id)

organizations ─┬─< members
               └─< api_keys

escrow_listings ─┬─< payment_reservations → escrows
escrows ─┬─< escrow_milestones, escrow_payments, messages, disputes
         └─ optional organization_id, listing_id

payout_outbox ──> withdrawals (async processing)
webhook_failures ──> failed Paystack payloads
inbox_notifications ──> users
```

### 5.3 Indexing Strategy

| Table | Index | Type | Purpose |
|-------|-------|------|---------|
| users | email | UNIQUE | Login lookup |
| users | phone | UNIQUE | OTP verification |
| users | kyc_status | B-TREE | Admin KYC queue filtering |
| escrows | buyer_id | B-TREE | User's escrows lookup |
| escrows | seller_id | B-TREE | User's escrows lookup |
| escrows | status | B-TREE | State-based queries |
| escrows | (status, delivered_at) | COMPOSITE | Auto-release cron query |
| escrow_payments | provider_reference | UNIQUE | Webhook idempotency |
| escrow_payments | idempotency_key | UNIQUE | Payment idempotency |
| wallets | (user_id, wallet_type) | COMPOSITE UNIQUE | One wallet per type per user |
| ledger_entries | wallet_id | B-TREE | Transaction history |
| ledger_entries | reference | UNIQUE | Entry deduplication |
| audit_logs | (resource_type, resource_id) | COMPOSITE | Resource history lookup |
| audit_logs | created_at | B-TREE | Time-range queries |

---

## 6. Background Workers

| Worker | Module | Interval / Trigger | Purpose |
|--------|--------|-------------------|---------|
| Auto-release | `escrow/` | Cron | Release funds after inspection period |
| Pending acceptance expiry | `escrow/` | Cron | Expire unaccepted escrows |
| Pre-release reminders | `escrow/` | Cron | Notify buyer before auto-release |
| Reservation expiry | `listing/` | Cron | Release held listing slots |
| Payout outbox | `payout/` | Poll | Process async Paystack transfers |
| Daily reconciliation | `reconciliation/` | Daily | Detect ledger/payment drift |

Workers start in `cmd/server/main.go` and share the application database connection pool.

---

## 7. Infrastructure

### 7.1 Docker Compose Stack

```
┌─────────────────────────────────────────────────┐
│              Docker Bridge Network               │
│                                                   │
│  ┌─────────────┐  ┌────────────┐  ┌───────────┐ │
│  │   vescrow   │  │ vescrow_db │  │   redis   │ │
│  │   :3000     │  │   :5432    │  │   :6379   │ │
│  │             │  │            │  │           │ │
│  │  Go Binary  │  │ PostgreSQL │  │  Redis 7  │ │
│  │  (Alpine)   │  │  15-Alpine │  │  (Alpine) │ │
│  └──────┬──────┘  └─────┬──────┘  └─────┬─────┘ │
│         │               │               │        │
│         └───────────────┼───────────────┘        │
│                         │                         │
│              ┌──────────┴──────────┐              │
│              │   Docker Volumes    │              │
│              │ • postgres-data     │              │
│              │ • redis-data        │              │
│              └─────────────────────┘              │
└─────────────────────────────────────────────────┘
```

### 7.2 Resource Recommendations

| Environment | CPU | Memory | Storage |
|------------|-----|--------|---------|
| Development | 1 core | 512 MB | 1 GB |
| Staging | 2 cores | 1 GB | 10 GB |
| Production | 4 cores | 4 GB | 100 GB (SSD) |

---

## 8. Error Handling Strategy

### 8.1 Standardized API Response Format

```json
// Success
{
  "success": true,
  "message": "Escrow created successfully",
  "data": { ... }
}

// Error
{
  "success": false,
  "message": "Validation failed",
  "errors": [
    { "field": "amount", "message": "Amount must be at least 100 kobo" }
  ]
}

// Paginated
{
  "success": true,
  "message": "Escrows retrieved",
  "data": [ ... ],
  "meta": {
    "page": 1,
    "per_page": 20,
    "total": 150,
    "total_pages": 8
  }
}
```

### 8.2 HTTP Status Codes

| Code | Usage |
|------|-------|
| 200 | Successful operation |
| 201 | Resource created |
| 400 | Validation error, bad request |
| 401 | Missing or invalid authentication |
| 403 | Insufficient permissions or KYC level |
| 404 | Resource not found |
| 409 | Conflict (duplicate, invalid state transition, idempotency key reuse) |
| 422 | Unprocessable entity |
| 429 | Rate limit exceeded |
| 500 | Internal server error |

---

## 9. Monitoring & Observability (Future)

### 9.1 Recommended Stack

| Tool | Purpose |
|------|---------|
| Prometheus | Metrics collection |
| Grafana | Dashboards and visualization |
| Loki | Log aggregation |
| Sentry | Error tracking |
| Uptime Kuma | Health checks |

### 9.2 Key Metrics to Track

- Request rate (RPM) by endpoint
- Response time (p50, p95, p99)
- Error rate by status code
- Active escrow count by status
- Daily transaction volume (kobo)
- Wallet balance totals
- Withdrawal success/failure rate
- Database connection pool utilization
- Redis memory usage

---

## 10. Future Architecture Considerations

### 10.1 Microservices Migration Path

If Vescrow outgrows the monolith, each domain module maps cleanly to a microservice:

| Module | → Service | Database | Communication |
|--------|-----------|----------|---------------|
| `auth/` | Auth Service | auth_db | REST / gRPC |
| `escrow/` | Escrow Service | escrow_db | Kafka events |
| `payment/` | Payment Service | (shared with escrow) | Kafka events |
| `wallet/` | Wallet Service | wallet_db | Kafka events |
| `dispute/` | Dispute Service | dispute_db | Kafka events |
| `notification/` | Notification Service | — (stateless) | Kafka consumer |
| `audit/` | Audit Service | audit_db | Kafka consumer |

### 10.2 Scaling Strategy

| Phase | Strategy |
|-------|---------|
| **Phase 1** (now) | Single instance, vertical scaling |
| **Phase 2** | Read replicas for PostgreSQL, Redis cluster |
| **Phase 3** | Horizontal scaling with load balancer (stateless app) |
| **Phase 4** | Decompose into microservices as needed |

---

**Document Version**: 2.0  
**Status**: Approved  
**Next Review**: Before production launch
