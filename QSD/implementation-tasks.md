# Vescrow Backend — Implementation Task List

**Version**: 2.0
**Created**: June 2026
**Updated**: August 2026
**Total Tasks**: 112 (99 original + 13 new refactoring tasks)
**ID Format**: `VES-XXX`

> **Current State**: Phases 1–12 are implemented. Phase 13 introduces the architecture simplification (remove wallets & tiers).
> **Goal**: Refactor to direct escrow disbursement model, then complete production hardening.

---

## How to Use This Document

1. **Pick a phase** — they are ordered by dependency (Phase 1 first, then 2, etc.)
2. **Complete every task** in the phase — each has clear validation criteria
3. **Run the validation** — don't move to the next phase until all criteria pass
4. **Check the box** — mark `[x]` when done

### Milestones

| Milestone | Phases | What You Get |
|-----------|--------|-------------|
| **MVP Core** | 1–7 | Working auth, escrow lifecycle, Paystack payments, ledger — demoable with test keys |
| **Trust & Safety** | 8–12 | Disputes, notifications, KYC (BVN), audit trail, auto-release cron |
| **Architecture Refactor** | 13 | ⚡ Remove wallets & tiers, add direct disbursement, simplify KYC |
| **Production Ready** | 14–19 | Admin dashboard, security hardening, testing, CI/CD, deployment |

---

## Phase 1 — Project Scaffolding & Configuration

Set up the Go project skeleton, Docker infrastructure, and environment config.

---

### VES-001 · Initialize Go module

Create `go.mod` with module path `github.com/didihart/vescrow_backend` and Go 1.23+.

**Validation Criteria:**
- [ ] `go.mod` exists at project root
- [ ] `go mod tidy` runs without errors
- [ ] Module path is `github.com/didihart/vescrow_backend`

---

### VES-002 · Create project directory structure

Create all directories from the project structure: `cmd/server/`, `internal/config/`, `internal/database/`, `internal/middleware/`, `internal/common/`, `internal/auth/`, `internal/escrow/`, `internal/payment/`, `internal/wallet/`, `internal/dispute/`, `internal/notification/`, `internal/admin/`, `internal/audit/`, `keys/`.

**Validation Criteria:**
- [ ] All 14 directories exist
- [ ] Placeholder `.gitkeep` files added where needed

---

### VES-003 · Create config loader (`internal/config/config.go`)

Load all environment variables into a strongly-typed `Config` struct. Use `os.Getenv` with sensible defaults for development.

**Validation Criteria:**
- [ ] Struct covers: Port, Environment, DB (host/port/user/password/name/ssl_mode), Redis URL, JWT paths + expiry, Paystack keys, SendGrid, Twilio, EncryptionKey, Platform settings
- [ ] `config.Load()` returns a populated `Config` or panics with a clear message on missing required vars
- [ ] Defaults work for local development without `.env` file

---

### VES-004 · Create `.env.example`

Template with all environment variables documented.

**Validation Criteria:**
- [ ] Contains every variable from the README's Environment Variables section
- [ ] Has placeholder values and inline comments explaining each variable
- [ ] Does NOT contain real secrets

---

### VES-005 · Create `Dockerfile` (multi-stage build)

Stage 1: Build Go binary with `golang:1.23-alpine`. Stage 2: Run with `alpine:3.19` as non-root user `appuser`.

**Validation Criteria:**
- [ ] `docker build -t vescrow:test .` succeeds
- [ ] Final image size < 30 MB
- [ ] Runs as non-root user (`appuser`)
- [ ] Binary is statically linked (CGO_ENABLED=0)

---

### VES-006 · Create `docker-compose.yml` (dev stack)

Services: `app` (Go, port 3000), `vescrow_db` (PostgreSQL 15-alpine, port 5432), `redis` (Redis 7-alpine, port 6379). Named volumes for persistence.

**Validation Criteria:**
- [ ] `docker-compose up -d vescrow_db redis` starts both infra services
- [ ] Can connect to Postgres via `psql -h localhost -p 5432 -U vescrow_user -d vescrow_db`
- [ ] Can connect to Redis via `redis-cli -h localhost -p 6379 ping` → `PONG`
- [ ] Named volumes `pgdata` and `redisdata` created

---

### VES-007 · Create `.gitignore`

Ignore: `.env`, `keys/*.pem`, `*.exe`, `vendor/`, `tmp/`, IDE files, `coverage.out`.

**Validation Criteria:**
- [ ] `.env` file not tracked by git
- [ ] `keys/private.pem` and `keys/public.pem` not tracked
- [ ] `git status` does not show ignored files after adding secrets

---

### VES-008 · Generate RSA keys for development

Create `keys/private.pem` (2048-bit) and `keys/public.pem`.

**Validation Criteria:**
- [ ] Both files exist in `keys/` directory
- [ ] Files are valid PEM format
- [ ] `openssl rsa -in keys/private.pem -check` succeeds
- [ ] Public key matches private key

---

## Phase 2 — Database Connection, Models & Migrations

Connect to PostgreSQL via GORM, define all domain models, and run auto-migrations.

---

### VES-009 · Install core dependencies

Add GORM, PostgreSQL driver, Fiber, Redis client, UUID, JWT library, and Argon2 to `go.mod`.

**Validation Criteria:**
- [ ] `go mod tidy` succeeds
- [ ] `go.sum` contains entries for: `gorm.io/gorm`, `gorm.io/driver/postgres`, `github.com/gofiber/fiber/v2`, `github.com/redis/go-redis/v9`, `github.com/google/uuid`, `github.com/golang-jwt/jwt/v5`

---

### VES-010 · Create database connection (`internal/database/connect.go`)

Connect to PostgreSQL with GORM using config values. Implement retry logic (3 retries, 5s backoff). Set connection pool: MaxIdleConns=10, MaxOpenConns=100, ConnMaxLifetime=1h.

**Validation Criteria:**
- [ ] `database.Connect(cfg)` returns a `*gorm.DB` or error
- [ ] Logs successful connection with database name
- [ ] Retries on transient connection failures (3 attempts, 5s apart)
- [ ] Connection pool settings applied correctly

---

### VES-011 · Define Auth models (`internal/auth/model.go`)

Structs: `Role` (id/name/permissions), `User` (all fields from users table), `KYCDocument` (all fields from kyc_documents table).

**Validation Criteria:**
- [ ] All columns from database-schema.md `users` table are represented as struct fields
- [ ] All columns from database-schema.md `kyc_documents` table are represented
- [ ] GORM tags include: `gorm:"primaryKey"`, `gorm:"uniqueIndex"`, `gorm:"type:uuid;default:uuid_generate_v4()"`, proper foreign keys
- [ ] `json` tags mask sensitive fields (password_hash → `json:"-"`)
- [ ] `Role` has seed-compatible structure with ID, Name, Permissions

---

### VES-012 · Define Escrow models (`internal/escrow/model.go`)

Structs: `Escrow`, `EscrowMilestone`, `Message` — all fields from schema doc.

**Validation Criteria:**
- [ ] All amounts are `int64` (kobo)
- [ ] `Escrow.Status` has a string type with constants for all 13 states: `draft`, `pending_acceptance`, `accepted`, `funded`, `in_progress`, `delivered`, `completed`, `disputed`, `refunded`, `partial_release`, `cancelled`, `expired`, `rejected`
- [ ] GORM relationships: Escrow HasMany Milestones, Messages
- [ ] JSONB field for `Metadata` uses `datatypes.JSON`
- [ ] Timestamps: `funded_at`, `delivered_at`, `completed_at`, `expires_at` are nullable

---

### VES-013 · Define Payment model (`internal/payment/model.go`)

Struct: `EscrowPayment` — all fields from escrow_payments table.

**Validation Criteria:**
- [ ] Unique indexes on `provider_reference`, `paystack_reference`, `idempotency_key`
- [ ] JSONB field for `ProviderResponse`
- [ ] Foreign keys to `escrows` and `users` tables
- [ ] Status field with constants: `pending`, `confirmed`, `failed`, `refunded`

---

### VES-014 · Define Wallet & Ledger models (`internal/wallet/model.go`)

Structs: `Wallet`, `LedgerEntry`, `Withdrawal` — all fields from schema doc.

**Validation Criteria:**
- [ ] Wallet has composite unique constraint `(user_id, wallet_type)`
- [ ] LedgerEntry.Reference is unique
- [ ] All amounts are `int64`
- [ ] Wallet types: `buyer`, `seller`, `escrow`, `platform_fee`
- [ ] Ledger categories: `escrow_fund`, `escrow_release`, `escrow_refund`, `withdrawal`, `platform_fee`, `deposit`
- [ ] Withdrawal statuses: `pending`, `processing`, `completed`, `failed`, `reversed`

---

### VES-015 · Define Dispute models (`internal/dispute/model.go`)

Structs: `Dispute`, `DisputeEvidence`, `DisputeComment` — all fields from schema doc.

**Validation Criteria:**
- [ ] All foreign keys defined (escrow_id, opened_by, resolved_by, dispute_id, uploaded_by, author_id)
- [ ] Dispute HasMany Evidence, Comments
- [ ] Dispute statuses: `open`, `seller_responded`, `under_review`, `resolved`
- [ ] Resolution types: `full_refund`, `full_release`, `partial_split`

---

### VES-016 · Define Audit & Admin models (`internal/audit/model.go`, `internal/admin/model.go`)

`AuditLog` and `PlatformSetting` structs.

**Validation Criteria:**
- [ ] AuditLog has JSONB fields for `OldState` and `NewState`
- [ ] AuditLog actor_type: `user`, `admin`, `system`
- [ ] PlatformSetting.Key has unique index
- [ ] AuditLog resource_type: `user`, `escrow`, `wallet`, `dispute`, `payment`, `withdrawal`, `kyc`

---

### VES-017 · Create migration runner (`internal/database/migrate.go`)

AutoMigrate all models. Enable `uuid-ossp` PostgreSQL extension.

**Validation Criteria:**
- [ ] `database.RunMigrations(db)` creates all 15 tables
- [ ] Running it twice is idempotent (no errors on re-run)
- [ ] `uuid-ossp` extension enabled (`SELECT * FROM pg_extension WHERE extname = 'uuid-ossp'` returns a row)
- [ ] All indexes from database-schema.md created

---

### VES-018 · Create seeder (`internal/database/seed.go`)

Seed: 3 roles (`user`, `admin`, `super_admin`), 1 super admin user (from env), 2 system wallets (`escrow`, `platform_fee`), 10 platform settings.

**Validation Criteria:**
- [ ] `database.Seed(db, cfg)` creates seed data
- [ ] Idempotent — running twice doesn't create duplicates (uses `FirstOrCreate` pattern)
- [ ] Roles: user (id=1), admin (id=2), super_admin (id=3)
- [ ] System wallets: escrow wallet (user_id=NULL), platform_fee wallet (user_id=NULL)
- [ ] Platform settings match the 10 values from database-schema.md seed data

---

### VES-019 · Create entry point (`cmd/server/main.go`)

Wire up: load config → connect DB → run migrations → seed → print "Vescrow server ready".

**Validation Criteria:**
- [ ] `go run cmd/server/main.go` connects to DB and logs success
- [ ] Tables are visible in PostgreSQL: `\dt` shows all 15 tables
- [ ] Seed data present: `SELECT * FROM roles` returns 3 rows
- [ ] System wallets present: `SELECT * FROM wallets WHERE user_id IS NULL` returns 2 rows

---

## Phase 3 — Common Utilities & Middleware

Build the shared utility layer and HTTP middleware pipeline.

---

### VES-020 · Standardized API responses (`internal/common/response.go`)

Functions: `Success(c, status, message, data)`, `Error(c, status, message)`, `ValidationError(c, errors)`, `Paginated(c, message, data, meta)`.

**Validation Criteria:**
- [ ] All responses follow the format: `{success: bool, message: string, data?: any, errors?: array, meta?: object}`
- [ ] `Success` sets `success: true`
- [ ] `Error` sets `success: false`
- [ ] `Paginated` includes `meta` with `page`, `per_page`, `total`, `total_pages`

---

### VES-021 · Custom error types (`internal/common/errors.go`)

Define: `AppError` struct with Code, Message, Field. Sentinel errors: `ErrUnauthorized`, `ErrForbidden`, `ErrNotFound`, `ErrConflict`, `ErrValidation`, `ErrInternalServer`.

**Validation Criteria:**
- [ ] Errors implement Go `error` interface
- [ ] `ErrUnauthorized` → HTTP 401
- [ ] `ErrForbidden` → HTTP 403
- [ ] `ErrNotFound` → HTTP 404
- [ ] `ErrConflict` → HTTP 409
- [ ] `ErrValidation` → HTTP 400

---

### VES-022 · Input validation (`internal/common/validator.go`)

Use `go-playground/validator` for struct validation. Helper to translate validation errors into the `errors[]` array format.

**Validation Criteria:**
- [ ] Validates: `required`, `email`, `min`/`max` length, phone format (E.164), `uuid`
- [ ] Returns field-level errors like `{"field": "email", "message": "Must be a valid email"}`
- [ ] Custom tag for Nigerian phone format (`+234XXXXXXXXXX`)

---

### VES-023 · JWT utilities (`internal/common/jwt.go`)

Functions: `GenerateAccessToken(userID, role)`, `GenerateRefreshToken(userID)`, `ParseToken(tokenString)`. Load RSA keys from PEM files. Claims: `sub` (userID), `role`, `exp`, `iat`, `jti`.

**Validation Criteria:**
- [ ] Tokens are RS256 signed (not HMAC)
- [ ] Access token expires in configurable time (default 15 minutes)
- [ ] Refresh token expires in configurable time (default 30 days)
- [ ] `ParseToken` returns claims or error for expired/invalid tokens
- [ ] Each token has a unique `jti` (JWT ID)

---

### VES-024 · Password hashing (`internal/common/password.go`)

Functions: `HashPassword(password)` → `[]byte`, `VerifyPassword(hash, password)` → `bool`. Use Argon2id with params: time=1, memory=64MB, threads=4, keyLength=32.

**Validation Criteria:**
- [ ] `HashPassword("test")` returns different bytes each call (random salt)
- [ ] `VerifyPassword(hash, "test")` returns `true`
- [ ] `VerifyPassword(hash, "wrong")` returns `false`
- [ ] Argon2id parameters: time=1, memory=65536, threads=4, keyLength=32

---

### VES-025 · OTP utilities (`internal/common/otp.go`)

Functions: `GenerateOTP()` → 6-digit string, `StoreOTP(ctx, key, otp, expiry)` (Redis), `VerifyOTP(ctx, key, otp)` → bool (deletes on success). OTP expiry: 10 minutes.

**Validation Criteria:**
- [ ] OTPs are 6-digit numeric strings (e.g., `"483921"`)
- [ ] Stored in Redis with key format `otp:<channel>:<target>` and TTL of 10 minutes
- [ ] OTP is deleted from Redis after successful verification (single-use)
- [ ] `VerifyOTP` returns `false` for wrong OTP without deleting the correct one

---

### VES-026 · Pagination helper (`internal/common/pagination.go`)

Parse `page` and `per_page` from Fiber query params. Calculate offset. Return `PaginationMeta{Page, PerPage, Total, TotalPages}`.

**Validation Criteria:**
- [ ] Default `page=1`, `per_page=20`
- [ ] Maximum `per_page=100` (silently capped)
- [ ] Correctly calculates offset: `(page - 1) * per_page`
- [ ] Correctly calculates total pages: `ceil(total / per_page)`

---

### VES-027 · Auth middleware (`internal/middleware/auth.go`)

Extract Bearer token from `Authorization` header → parse JWT → inject `userID` and `role` into Fiber context locals.

**Validation Criteria:**
- [ ] Returns 401 JSON error if `Authorization` header missing
- [ ] Returns 401 JSON error if token is expired or invalid
- [ ] Sets `c.Locals("userID")` to the user's UUID string on success
- [ ] Sets `c.Locals("role")` to the user's role string on success

---

### VES-028 · Admin-only middleware (`internal/middleware/admin_only.go`)

Check `c.Locals("role")` is `admin` or `super_admin`.

**Validation Criteria:**
- [ ] Returns 403 for users with role `user`
- [ ] Allows users with role `admin` through
- [ ] Allows users with role `super_admin` through
- [ ] Error message: `"Admin access required"`

---

### VES-029 · KYC gate middleware (`internal/middleware/kyc_required.go`)

Accept a minimum tier parameter. Look up user's `kyc_tier` from database or JWT claims.

**Validation Criteria:**
- [ ] `KYCRequired("tier1")` blocks users with `kyc_tier = "none"`
- [ ] `KYCRequired("tier1")` allows users with `kyc_tier = "tier1"`, `"tier2"`, or `"tier3"`
- [ ] Returns 403 with message: `"KYC Tier 1 required for this operation"`
- [ ] Tier comparison is hierarchical: tier3 > tier2 > tier1 > none

---

### VES-030 · Rate limiter middleware (`internal/middleware/rate_limiter.go`)

Redis-backed sliding window rate limiter. Configurable per-route: key (IP or userID), max requests, window duration.

**Validation Criteria:**
- [ ] Returns 429 with `Retry-After` header when limit exceeded
- [ ] Error message: `"Rate limit exceeded. Try again in X seconds."`
- [ ] General API: 100 req/min per IP
- [ ] Auth routes: 5 req/15min per IP
- [ ] Counter resets correctly after window expires

---

### VES-031 · CORS middleware (`internal/middleware/cors.go`)

Whitelist origins from config `ALLOWED_ORIGINS`. Allow methods: GET, POST, PUT, DELETE. Allow headers: Authorization, Content-Type.

**Validation Criteria:**
- [ ] Blocks requests from non-whitelisted origins in production
- [ ] Allows all origins in development mode (`ENVIRONMENT=development`)
- [ ] Preflight `OPTIONS` requests handled correctly
- [ ] `Access-Control-Allow-Credentials: true` for cookie-based refresh tokens

---

### VES-032 · Wire Fiber app in main.go

Create Fiber app with config (Body limit 10MB, JSON encoder). Register global middleware: CORS → Rate Limiter → Logger. Start server on configured port.

**Validation Criteria:**
- [ ] `go run cmd/server/main.go` starts HTTP server on port 3000
- [ ] `curl http://localhost:3000/api/v1/health` returns a response (even if placeholder)
- [ ] CORS headers present in response
- [ ] Fiber body limit set to 10MB

---

## Phase 4 — Authentication Module

Full auth flow: register, login, OTP, JWT, password management, profile.

---

### VES-033 · Auth repository (`internal/auth/repository.go`)

Interface + GORM implementation. Methods: `CreateUser`, `FindByEmail`, `FindByPhone`, `FindByID`, `UpdateUser`, `CreateKYCDocument`, `FindKYCDocumentsByUser`.

**Validation Criteria:**
- [ ] All methods tested with actual DB queries
- [ ] `FindByEmail` returns `nil, nil` for non-existent users (not an error)
- [ ] `CreateUser` returns the created user with generated UUID
- [ ] `UpdateUser` only updates non-zero fields (partial update)

---

### VES-034 · Auth service (`internal/auth/service.go`)

Business logic: `Register`, `Login`, `VerifyEmail`, `VerifyPhone`, `ResendOTP`, `RefreshToken`, `Logout`, `ForgotPassword`, `ResetPassword`, `ChangePassword`, `GetProfile`, `UpdateProfile`.

**Validation Criteria:**
- [ ] Registration creates the user + a `seller` type wallet automatically
- [ ] Registration sends email OTP via notification service
- [ ] Login returns access token in JSON body
- [ ] Login sets refresh token as HTTP-only secure cookie
- [ ] Cannot login if email not verified → returns `"Please verify your email first"`
- [ ] Cannot login if user is suspended/banned → returns appropriate error
- [ ] Password change requires correct current password
- [ ] Forgot password always returns 200 (prevents email enumeration)

---

### VES-035 · Auth handler (`internal/auth/handler.go`)

HTTP handlers for all 12 auth endpoints. Parse request bodies, call service, return standardized responses.

**Validation Criteria:**
- [ ] Request body validation with clear field-level error messages
- [ ] Phone numbers masked in responses: `+234803****567`
- [ ] Email masked in responses: `s***@example.com`
- [ ] Password hash never returned in any response
- [ ] All responses use standardized response format from VES-020

---

### VES-036 · Auth routes (`internal/auth/routes.go`)

Register all auth routes under `/api/v1/auth`. Apply rate limiting to login/register/OTP routes.

**Validation Criteria:**
- [ ] `POST /api/v1/auth/register` → reachable, no auth required
- [ ] `POST /api/v1/auth/login` → reachable, no auth required
- [ ] `POST /api/v1/auth/verify-email` → reachable, no auth required
- [ ] `POST /api/v1/auth/verify-phone` → reachable, no auth required
- [ ] `POST /api/v1/auth/resend-otp` → reachable, no auth required
- [ ] `POST /api/v1/auth/refresh` → reachable, uses refresh token cookie
- [ ] `POST /api/v1/auth/logout` → requires JWT
- [ ] `POST /api/v1/auth/forgot-password` → no auth required
- [ ] `POST /api/v1/auth/reset-password` → no auth required (uses reset token)
- [ ] `POST /api/v1/auth/change-password` → requires JWT
- [ ] `GET  /api/v1/auth/profile` → requires JWT
- [ ] `PUT  /api/v1/auth/profile` → requires JWT
- [ ] Rate limiter active on login, register, OTP endpoints (5 req/15min per IP)

---

### VES-037 · End-to-end auth flow test

Manual or script test of the full authentication flow.

**Validation Criteria:**
- [ ] Register with valid data → 201 with user data (masked PII)
- [ ] Verify email with correct OTP → 200 success
- [ ] Login with correct credentials → 200 with `access_token` + `Set-Cookie: refresh_token`
- [ ] Access `GET /auth/profile` with valid token → 200 with user profile
- [ ] Refresh token → new `access_token` returned
- [ ] Logout → subsequent requests with old access token still work (stateless JWT) but refresh is invalidated
- [ ] Login with wrong password → 401

---

## Phase 5 — Escrow Transaction Module

The core business logic: create, submit, accept/reject, deliver, approve escrows.

---

### VES-038 · Escrow state machine (`internal/escrow/state_machine.go`)

Define valid state transitions as a map. Function: `CanTransition(currentState, targetState) bool`. Define all 13 states as constants.

**Validation Criteria:**
- [ ] Valid transitions match the state machine in README:
  - `draft` → `pending_acceptance`, `cancelled`
  - `pending_acceptance` → `accepted`, `rejected`, `expired`, `cancelled`
  - `accepted` → `funded`, `cancelled`
  - `funded` → `in_progress`
  - `in_progress` → `delivered`
  - `delivered` → `completed`, `disputed`
  - `disputed` → `completed`, `refunded`, `partial_release`
- [ ] `CanTransition("draft", "pending_acceptance")` → `true`
- [ ] `CanTransition("completed", "draft")` → `false`
- [ ] Terminal states have no outgoing transitions: `completed`, `refunded`, `cancelled`, `expired`, `rejected`, `partial_release`

---

### VES-039 · Escrow repository (`internal/escrow/repository.go`)

Methods: `Create`, `FindByID`, `FindByUserID` (paginated, with role + status filters), `Update`, `CreateMilestone`, `FindMilestonesByEscrow`, `UpdateMilestone`, `CreateMessage`, `FindMessagesByEscrow`.

**Validation Criteria:**
- [ ] `FindByUserID` returns escrows where user is buyer OR seller
- [ ] Pagination works: offset, limit, total count returned
- [ ] Status filter: `?status=funded` returns only funded escrows
- [ ] Role filter: `?role=buyer` returns only escrows where user is buyer
- [ ] Messages ordered by `created_at ASC`

---

### VES-040 · Escrow service (`internal/escrow/service.go`)

Business logic for all escrow operations.

**Validation Criteria:**
- [ ] `CreateEscrow` rejects if `buyer_id == seller_id`
- [ ] `CreateEscrow` looks up seller by email and creates escrow with correct `seller_id`
- [ ] Fee calculation: `amount_kobo * platform_fee_percent / 100`
- [ ] Milestones sum validated: `SUM(milestone.amount_kobo) == escrow.amount_kobo`
- [ ] `SubmitToSeller` sets `expires_at` to `now + 72 hours`
- [ ] Every state transition validated by state machine (VES-038)
- [ ] `ApproveDelivery` calls wallet service to release funds
- [ ] `ExtendInspection` caps total inspection at max 30 days
- [ ] Only escrow buyer can: create, submit, fund, approve delivery, dispute, extend
- [ ] Only escrow seller can: accept, reject, mark delivered

---

### VES-041 · Escrow handler (`internal/escrow/handler.go`)

HTTP handlers for all 14 escrow endpoints.

**Validation Criteria:**
- [ ] Create requires JWT + KYC tier >= 1
- [ ] All state-change endpoints validate the actor (buyer vs seller)
- [ ] Error responses include which role is required (e.g., "Only the buyer can approve delivery")
- [ ] Escrow details include milestones, payment status, timeline data

---

### VES-042 · Escrow routes (`internal/escrow/routes.go`)

Register all routes under `/api/v1/escrows`.

**Validation Criteria:**
- [ ] `POST   /api/v1/escrows` → create (JWT + KYC)
- [ ] `GET    /api/v1/escrows` → list (JWT)
- [ ] `GET    /api/v1/escrows/:id` → details (JWT)
- [ ] `PUT    /api/v1/escrows/:id` → update draft (JWT)
- [ ] `DELETE /api/v1/escrows/:id` → cancel draft (JWT)
- [ ] `PUT    /api/v1/escrows/:id/submit` → submit to seller (JWT)
- [ ] `PUT    /api/v1/escrows/:id/accept` → seller accepts (JWT)
- [ ] `PUT    /api/v1/escrows/:id/reject` → seller rejects (JWT)
- [ ] `POST   /api/v1/escrows/:id/fund` → init payment (JWT)
- [ ] `PUT    /api/v1/escrows/:id/deliver` → mark delivered (JWT)
- [ ] `PUT    /api/v1/escrows/:id/approve-delivery` → approve + release (JWT)
- [ ] `PUT    /api/v1/escrows/:id/dispute` → raise dispute (JWT)
- [ ] `PUT    /api/v1/escrows/:id/extend-inspection` → extend (JWT)
- [ ] `GET    /api/v1/escrows/:id/messages` → get messages (JWT)
- [ ] `POST   /api/v1/escrows/:id/messages` → send message (JWT)
- [ ] `GET    /api/v1/escrows/:id/milestones` → get milestones (JWT)
- [ ] `PUT    /api/v1/escrows/:id/milestones/:mid/complete` → complete (JWT)
- [ ] `PUT    /api/v1/escrows/:id/milestones/:mid/approve` → approve (JWT)

---

### VES-043 · End-to-end escrow lifecycle test

Test the full lifecycle from creation to completion.

**Validation Criteria:**
- [ ] Create escrow → 201, status = `draft`
- [ ] Submit to seller → status = `pending_acceptance`
- [ ] Seller accepts → status = `accepted`
- [ ] Buyer funds (via Paystack mock/test) → status = `funded`
- [ ] Seller marks delivered → status = `delivered`, `delivered_at` set
- [ ] Buyer approves delivery → status = `completed`, funds in seller wallet
- [ ] Seller wallet balance increased by `amount - fee`
- [ ] Platform fee wallet balance increased by `fee`

---

## Phase 6 — Payment Module (Paystack Integration)

Integrate Paystack for payment collection and webhook processing.

---

### VES-044 · Paystack API client (`internal/payment/paystack.go`)

HTTP client wrapper for Paystack API endpoints.

**Validation Criteria:**
- [ ] `InitializeTransaction(email, amountKobo, reference, metadata, callbackURL)` → calls `POST https://api.paystack.co/transaction/initialize`
- [ ] `VerifyTransaction(reference)` → calls `GET https://api.paystack.co/transaction/verify/:reference`
- [ ] `ListBanks()` → calls `GET https://api.paystack.co/bank`
- [ ] `ResolveAccountNumber(accountNumber, bankCode)` → calls `GET https://api.paystack.co/bank/resolve`
- [ ] `CreateTransferRecipient(name, accountNumber, bankCode)` → calls `POST https://api.paystack.co/transferrecipient`
- [ ] `InitiateTransfer(amountKobo, recipientCode, reason, reference)` → calls `POST https://api.paystack.co/transfer`
- [ ] Secret key passed as `Authorization: Bearer sk_test_xxx`
- [ ] HTTP timeout set to 30 seconds
- [ ] Proper error handling for non-200 Paystack responses

---

### VES-045 · Payment repository (`internal/payment/repository.go`)

Methods: `Create`, `FindByReference`, `FindByEscrowID`, `UpdateStatus`, `FindByIdempotencyKey`.

**Validation Criteria:**
- [ ] `FindByReference` returns payment by `paystack_reference`
- [ ] `FindByIdempotencyKey` prevents duplicate payment records
- [ ] `UpdateStatus` updates status and sets `paid_at` timestamp on confirmation

---

### VES-046 · Payment service (`internal/payment/service.go`)

Initialize payments, verify transactions, and handle Paystack webhooks.

**Validation Criteria:**
- [ ] `InitializePayment` generates unique reference format: `VESC_REF_<uuid>`
- [ ] `InitializePayment` creates an idempotency key per escrow+user combination
- [ ] `HandleWebhook` verifies HMAC-SHA512 signature using `PAYSTACK_WEBHOOK_SECRET`
- [ ] Webhook rejects requests with invalid signature (returns 401)
- [ ] Idempotency: processing same webhook twice does NOT create duplicate ledger entries
- [ ] On `charge.success`: payment → `confirmed`, escrow → `funded`, credit escrow wallet, create ledger entries
- [ ] On `transfer.success`: withdrawal → `completed`, notify user
- [ ] On `transfer.failed`: withdrawal → `failed`, reverse wallet debit, notify user

---

### VES-047 · Payment handler (`internal/payment/handler.go`)

HTTP handlers for payment endpoints.

**Validation Criteria:**
- [ ] `POST /payments/initialize` → requires JWT, returns `{authorization_url, reference}`
- [ ] `GET /payments/verify/:reference` → requires JWT, returns payment status
- [ ] `POST /payments/webhook/paystack` → NO JWT (public), uses HMAC signature verification
- [ ] Webhook always returns 200 to Paystack (even on internal errors, but logs them)

---

### VES-048 · Payment routes (`internal/payment/routes.go`)

Register routes under `/api/v1/payments`.

**Validation Criteria:**
- [ ] `POST /api/v1/payments/initialize` → JWT protected
- [ ] `GET  /api/v1/payments/verify/:reference` → JWT protected
- [ ] `POST /api/v1/payments/webhook/paystack` → public (no auth middleware)

---

### VES-049 · Paystack integration test

Using Paystack test keys.

**Validation Criteria:**
- [ ] Initialize with test key → receive valid Paystack `authorization_url`
- [ ] Verify with known test reference → get transaction details from Paystack
- [ ] Simulated webhook with valid HMAC signature → processed correctly
- [ ] Simulated webhook with invalid signature → rejected
- [ ] Duplicate webhook with same reference → no duplicate ledger entries

---

## Phase 7 — Wallet & Double-Entry Ledger

Internal wallet system with financial integrity guarantees.

---

### VES-050 · Ledger engine (`internal/wallet/ledger.go`)

Core double-entry bookkeeping engine.

**Validation Criteria:**
- [ ] `RecordEntry(tx, walletID, escrowID, entryType, amountKobo, category, description)` creates a ledger entry
- [ ] Debit = reduce wallet balance, Credit = increase wallet balance
- [ ] `balance_after_kobo` snapshot always matches the wallet's actual balance post-operation
- [ ] Rejects debit if wallet balance would go negative (returns error)
- [ ] Generates unique reference per entry (format: `LEDGER_<uuid>`)
- [ ] All operations run within a GORM transaction (caller provides `tx`)

---

### VES-051 · Wallet repository (`internal/wallet/repository.go`)

Methods: `CreateWallet`, `FindByUserAndType`, `FindByID`, `UpdateBalance`, `GetLedgerEntries`, `CreateWithdrawal`, `FindWithdrawalsByUser`, `UpdateWithdrawal`.

**Validation Criteria:**
- [ ] `UpdateBalance` uses atomic SQL: `gorm.Expr("balance_kobo + ?", amount)` — no race conditions
- [ ] `FindByUserAndType("user-uuid", "seller")` returns the user's seller wallet
- [ ] `GetLedgerEntries` is paginated and ordered by `created_at DESC`
- [ ] `FindWithdrawalsByUser` is paginated

---

### VES-052 · Wallet service (`internal/wallet/service.go`)

Wallet operations and fund management.

**Validation Criteria:**
- [ ] `GetBalance(userID)` returns wallet balance, currency, frozen status
- [ ] `FundEscrow(escrowID, amountKobo)` → credit escrow system wallet + create ledger entries
- [ ] `ReleaseToSeller(escrowID)` creates exactly 3 ledger entries in one DB transaction:
  1. Debit escrow wallet (full amount)
  2. Credit seller wallet (amount - fee)
  3. Credit platform_fee wallet (fee)
- [ ] After release: `SUM(credits) == SUM(debits)` for the escrow
- [ ] `RefundToBuyer(escrowID)` → debit escrow wallet, credit buyer wallet
- [ ] `RequestWithdrawal` validates: sufficient balance, daily limit (₦5,000,000 = 500000000 kobo), per-tx limit (₦2,000,000 = 200000000 kobo), minimum ₦1,000 (100000 kobo)
- [ ] First-ever withdrawal has 24-hour cooldown (checks user's withdrawal history)

---

### VES-053 · Wallet handler (`internal/wallet/handler.go`)

HTTP handlers for wallet endpoints.

**Validation Criteria:**
- [ ] `GET /wallet` → returns `{balance_kobo, currency, is_frozen}`
- [ ] `GET /wallet/transactions` → paginated ledger entries
- [ ] `POST /wallet/withdraw` → requires JWT + KYC, returns withdrawal record
- [ ] `GET /wallet/withdrawals` → paginated withdrawal history
- [ ] `GET /wallet/banks` → list of Nigerian banks from Paystack
- [ ] `POST /wallet/verify-account` → NUBAN account resolution via Paystack

---

### VES-054 · Wallet routes (`internal/wallet/routes.go`)

Register routes under `/api/v1/wallet`.

**Validation Criteria:**
- [ ] All 6 endpoints registered with correct HTTP methods
- [ ] All endpoints require JWT
- [ ] `POST /wallet/withdraw` additionally requires KYC gate (tier >= 1)

---

### VES-055 · Ledger integrity test

Validate double-entry bookkeeping invariants.

**Validation Criteria:**
- [ ] After a full escrow lifecycle: `SELECT SUM(CASE WHEN entry_type='credit' THEN amount_kobo ELSE 0 END) - SUM(CASE WHEN entry_type='debit' THEN amount_kobo ELSE 0 END) FROM ledger_entries` = `0`
- [ ] Concurrent withdrawal attempts don't result in negative wallet balance
- [ ] Every ledger entry has a unique reference
- [ ] Running balance snapshots are accurate across all entries

---

## Phase 8 — Dispute Resolution Module

Evidence-based disputes with admin arbitration.

---

### VES-056 · Dispute repository (`internal/dispute/repository.go`)

Methods: `Create`, `FindByID`, `FindByEscrowID`, `FindByUser`, `FindAll`, `Update`, `CreateEvidence`, `FindEvidenceByDispute`, `CreateComment`, `FindCommentsByDispute`.

**Validation Criteria:**
- [ ] `FindAll` supports filtering by status (for admin dashboard)
- [ ] `FindByUser` returns disputes where user is buyer OR seller of the related escrow
- [ ] Evidence and comments ordered by `created_at ASC`
- [ ] `FindAll` is paginated

---

### VES-057 · Dispute service (`internal/dispute/service.go`)

Dispute lifecycle management.

**Validation Criteria:**
- [ ] `OpenDispute` validates escrow is in `delivered` state → rejects otherwise with 409
- [ ] `OpenDispute` transitions escrow → `disputed`
- [ ] `UploadEvidence` saves file metadata (URL, type, description)
- [ ] `SubmitSellerResponse` transitions dispute → `seller_responded`
- [ ] `AddComment` allows buyer, seller, and admin to participate
- [ ] `ResolveDispute("full_refund")` → debit escrow wallet → credit buyer wallet
- [ ] `ResolveDispute("full_release")` → debit escrow → credit seller (minus fee) + credit platform_fee
- [ ] `ResolveDispute("partial_split", buyerAmount, sellerAmount)` → split per specified amounts
- [ ] Partial split validates: `buyerAmount + sellerAmount == escrow.amount_kobo`
- [ ] All resolutions create audit logs

---

### VES-058 · Dispute handler (`internal/dispute/handler.go`)

HTTP handlers for dispute endpoints.

**Validation Criteria:**
- [ ] `POST /disputes` → only buyer can open (validates against escrow)
- [ ] `POST /disputes/:id/respond` → only seller can respond
- [ ] `PUT  /disputes/:id/resolve` → only admin can resolve
- [ ] `POST /disputes/:id/evidence` → buyer + seller can upload
- [ ] `POST /disputes/:id/comment` → buyer + seller + admin can comment
- [ ] `GET  /disputes` → user sees only their disputes
- [ ] `GET  /disputes/:id` → only escrow participants + admin can view

---

### VES-059 · Dispute routes (`internal/dispute/routes.go`)

Register routes under `/api/v1/disputes`.

**Validation Criteria:**
- [ ] All endpoints require JWT
- [ ] `PUT /disputes/:id/resolve` additionally requires admin middleware

---

### VES-060 · Dispute flow end-to-end test

Test the full dispute lifecycle.

**Validation Criteria:**
- [ ] Buyer opens dispute on `delivered` escrow → escrow status = `disputed`
- [ ] Attempting to open dispute on non-delivered escrow → 409 error
- [ ] Both parties upload evidence → evidence visible in dispute details
- [ ] Seller responds → dispute status = `seller_responded`
- [ ] Admin resolves as `partial_split` → correct wallet credits to both parties
- [ ] Escrow status updated to `partial_release`
- [ ] Audit log records the resolution with admin's user ID

---

## Phase 9 — Notification Service

Email (SendGrid) and SMS (Twilio) notifications.

---

### VES-061 · Notification service interface (`internal/notification/service.go`)

Define `NotificationService` interface.

**Validation Criteria:**
- [ ] Interface methods: `SendEmail(to, subject, templateName, data)`, `SendSMS(phone, message)`, `NotifyEscrowEvent(escrowID, event, recipients)`
- [ ] Implementation can be swapped (e.g., mock for testing, real for production)
- [ ] Interface lives in a package that other modules can import without circular dependencies

---

### VES-062 · SendGrid email client (`internal/notification/email.go`)

Integrate SendGrid API for transactional emails.

**Validation Criteria:**
- [ ] Sends emails via SendGrid REST API (`POST https://api.sendgrid.com/v3/mail/send`)
- [ ] Falls back gracefully in development: logs email content to stdout instead of sending
- [ ] Handles SendGrid API errors without crashing the app (logs error, returns it)
- [ ] From address configured via `SENDGRID_FROM_EMAIL` and `SENDGRID_FROM_NAME`

---

### VES-063 · Twilio SMS client (`internal/notification/sms.go`)

Integrate Twilio API for SMS delivery.

**Validation Criteria:**
- [ ] Sends SMS via Twilio REST API
- [ ] Falls back gracefully in development: logs SMS content to stdout
- [ ] Validates E.164 phone format before sending (e.g., `+2348031234567`)
- [ ] From number configured via `TWILIO_PHONE_NUMBER`

---

### VES-064 · Email templates (`internal/notification/templates/`)

HTML email templates using Go `html/template`.

**Validation Criteria:**
- [ ] Template: `welcome.html` — welcome message after registration
- [ ] Template: `email_otp.html` — OTP for email verification
- [ ] Template: `phone_otp.html` — OTP for phone verification (SMS fallback text)
- [ ] Template: `escrow_created.html` — seller notification of new escrow
- [ ] Template: `escrow_funded.html` — both parties notified of funding
- [ ] Template: `delivery_notification.html` — buyer notified of delivery
- [ ] Template: `inspection_reminder.html` — 24h before auto-release warning
- [ ] Template: `dispute_opened.html` — both parties notified
- [ ] Template: `dispute_resolved.html` — both parties notified with resolution
- [ ] Template: `withdrawal_completed.html` — withdrawal success confirmation
- [ ] All templates use data injection (`{{.UserName}}`, `{{.Amount}}`, etc.)

---

### VES-065 · Integrate notifications across modules

Wire notification service into auth, escrow, wallet, and dispute modules.

**Validation Criteria:**
- [ ] Registration → sends email OTP
- [ ] Escrow submission → notifies seller by email
- [ ] Escrow funded → notifies both parties
- [ ] Delivery marked → notifies buyer
- [ ] Dispute opened → notifies other party
- [ ] Dispute resolved → notifies both parties
- [ ] Withdrawal completed → notifies user by email + SMS
- [ ] 24h before auto-release → sends inspection reminder email
- [ ] All notification failures are logged but don't crash the main operation

---

## Phase 10 — Audit Logging Module

Immutable, append-only audit trail for regulatory compliance.

---

### VES-066 · Audit service (`internal/audit/service.go`)

Append-only audit logger.

**Validation Criteria:**
- [ ] `LogAction(actorID, actorType, action, resourceType, resourceID, oldState, newState, ipAddress, userAgent)` creates an `AuditLog` record
- [ ] No `Update` or `Delete` methods exist on the service — INSERT only
- [ ] `actorType` validated: must be `user`, `admin`, or `system`
- [ ] `action` uses dot notation: `escrow.created`, `escrow.funded`, `user.registered`, etc.

---

### VES-067 · Audit repository (`internal/audit/repository.go`)

INSERT-only data access.

**Validation Criteria:**
- [ ] `Create(log)` → INSERT only
- [ ] `FindByResource(resourceType, resourceID)` → paginated, ordered by `created_at DESC`
- [ ] `FindByActor(actorID)` → paginated
- [ ] `FindAll(filters)` → paginated, supports date range filter (`from`, `to`)
- [ ] No `Update()` or `Delete()` methods defined

---

### VES-068 · Integrate audit logging across modules

Add audit log calls to all significant operations.

**Validation Criteria:**
- [ ] `user.registered` — logged on new registration
- [ ] `user.login` — logged on successful login
- [ ] `escrow.created` — logged with escrow details
- [ ] `escrow.status_changed` — logged on every state transition with old + new state
- [ ] `payment.confirmed` — logged with Paystack reference
- [ ] `wallet.withdrawal_requested` — logged with amount
- [ ] `wallet.withdrawal_completed` — logged on success
- [ ] `dispute.opened` — logged with dispute reason
- [ ] `dispute.resolved` — logged with resolution + admin ID
- [ ] `kyc.approved` / `kyc.rejected` — logged with admin ID
- [ ] `admin.user_suspended` / `admin.user_banned` — logged
- [ ] IP address and user agent captured from request context

---

## Phase 11 — KYC Verification Module

Tiered KYC with BVN/NIN verification and document management.

---

### VES-069 · PII encryption utilities (`internal/common/encryption.go`)

AES-256-GCM encryption for sensitive data at rest.

**Validation Criteria:**
- [ ] `Encrypt(plaintext, key)` → returns base64-encoded ciphertext
- [ ] `Decrypt(ciphertext, key)` → returns original plaintext
- [ ] `Decrypt(Encrypt("12345678901", key), key)` returns `"12345678901"`
- [ ] Same plaintext produces different ciphertext each call (random 12-byte nonce)
- [ ] Key must be exactly 32 bytes (derived from hex-encoded env var)
- [ ] Returns clear error if key is wrong length

---

### VES-070 · KYC service methods (extend `internal/auth/service.go`)

BVN/NIN verification and document management.

**Validation Criteria:**
- [ ] `VerifyBVN(userID, bvn)` → validates 11-digit format, encrypts, stores, promotes to Tier 1
- [ ] `VerifyNIN(userID, nin)` → validates 11-digit format, encrypts, stores
- [ ] `SubmitDocument(userID, docType, docURL, selfieURL)` → creates KYC document record with `status = "pending"`
- [ ] `GetKYCStatus(userID)` → returns tier, status, BVN linked flag, NIN linked flag, documents list
- [ ] Tier promotion logic:
  - BVN alone → `tier1`
  - BVN + NIN + approved government ID → `tier2`
  - Tier 2 + address verification + selfie → `tier3`
- [ ] Transaction limit enforcement per tier (Tier 1: ₦50,000, Tier 2: ₦500,000, Tier 3: unlimited)

---

### VES-071 · KYC admin endpoints (in admin handler)

Admin KYC management.

**Validation Criteria:**
- [ ] `GET  /admin/kyc/submissions` → list pending KYC documents (paginated)
- [ ] `GET  /admin/kyc/submissions/:id` → view submission details
- [ ] `PUT  /admin/kyc/submissions/:id/approve` → sets `status = "approved"`, upgrades user's `kyc_tier`, sets `verified_by`, `verified_at`
- [ ] `PUT  /admin/kyc/submissions/:id/reject` → sets `status = "rejected"`, requires `rejection_reason`
- [ ] Audit log created for every admin KYC action (VES-068)
- [ ] Only admin/super_admin can access these endpoints

---

### VES-072 · KYC routes

Register KYC routes.

**Validation Criteria:**
- [ ] `POST /api/v1/kyc/verify-bvn` → JWT required
- [ ] `POST /api/v1/kyc/verify-nin` → JWT required
- [ ] `POST /api/v1/kyc/submit-document` → JWT required
- [ ] `GET  /api/v1/kyc/status` → JWT required
- [ ] Admin KYC endpoints under `/api/v1/admin/kyc/` → JWT + admin middleware

---

## Phase 12 — Auto-Release Cron Jobs

Background workers for time-based state transitions.

---

### VES-073 · Auto-release worker

Background goroutine that runs every hour to auto-release funds.

**Validation Criteria:**
- [x] Runs automatically every hour via `time.Ticker`
- [x] Query: escrows with `status = 'delivered'` AND `delivered_at + inspection_period_days < NOW()`
- [x] Skips escrows with active disputes (`disputed` status)
- [x] For each qualifying escrow: transition → `completed`, release funds via wallet service
- [x] Creates audit log with `actor_type = "system"`, action = `escrow.auto_released`
- [x] Sends notification to both buyer and seller
- [x] Errors in one escrow don't block processing of others

---

### VES-074 · Pending-acceptance expiry worker

Expire escrows stuck in `pending_acceptance` beyond their `expires_at` time.

**Validation Criteria:**
- [x] Query: escrows with `status = 'pending_acceptance'` AND `expires_at < NOW()`
- [x] Transitions matching escrows → `expired`
- [x] Buyer notified of expiry
- [x] Creates audit log with `actor_type = "system"`, action = `escrow.expired`

---

### VES-075 · Auto-release notification sequence

Pre-release reminder notifications.

**Validation Criteria:**
- [x] 24 hours before auto-release → email reminder to buyer
- [x] 12 hours before auto-release → SMS reminder to buyer
- [x] Notifications contain: escrow title, amount, countdown time, link to approve/dispute
- [x] No duplicate reminders sent (track reminder state or use idempotent checks)

---

## Phase 13 — Architecture Simplification (Remove Wallets & Tiers)

> ⚡ **REFACTORING PHASE**: This phase restructures the existing codebase based on CTO/management feedback. It removes user wallets, removes KYC tiers, and introduces direct escrow-to-seller-bank disbursement via Paystack Transfer API.
>
> **Pre-requisite**: Phases 1–12 complete. All existing tests pass before starting.

---

### VES-100 · Rename wallet module → ledger module (`internal/wallet/` → `internal/ledger/`)

Rename the wallet package to `ledger`. This module now manages only system-level accounts and double-entry bookkeeping.

**Validation Criteria:**
- [ ] Package renamed from `wallet` to `ledger`
- [ ] All import paths updated across the codebase (`internal/wallet` → `internal/ledger`)
- [ ] `go build ./...` succeeds with no import errors
- [ ] Git history preserved (use `git mv`)

---

### VES-101 · Replace Wallet model with SystemAccount model

Replace the `Wallet` struct with `SystemAccount`. Remove user-facing wallet types. Only system accounts remain.

**Validation Criteria:**
- [ ] `SystemAccount` struct: `id`, `account_type` (UNIQUE: `escrow`, `platform_fee`), `balance_kobo`, `currency`, `is_frozen`, timestamps
- [ ] Removed `UserID` field — system accounts have no user association
- [ ] Removed `WalletType` enum values: `buyer`, `seller` — only `escrow` and `platform_fee` remain
- [ ] Renamed `WalletType` to `AccountType`
- [ ] `LedgerEntry.WalletID` renamed to `LedgerEntry.AccountID`, references `SystemAccount`
- [ ] Database migration auto-creates exactly 2 system accounts via seed: `escrow`, `platform_fee`
- [ ] `Withdrawal` model **deleted entirely**
- [ ] `WithdrawalStatus` enum **deleted entirely**
- [ ] `go test ./internal/ledger/...` compiles (tests may fail until updated in VES-110)

---

### VES-102 · Remove user wallet creation from auth registration

The auth service currently creates buyer/seller wallets on user registration. Remove this.

**Validation Criteria:**
- [ ] `auth/service.go` → `Register()` no longer calls wallet/ledger service to create user wallets
- [ ] No `CreateWallet` or `GetOrCreateWallet` calls for user-scoped wallets remain in auth module
- [ ] User registration returns successfully without wallet creation
- [ ] Existing auth tests pass (mocks for wallet service removed)

---

### VES-103 · Remove KYC tier system

Replace the tiered KYC model with a single BVN verification gate.

**Validation Criteria:**
- [ ] `KYCTier` enum removed from `auth/enum.go` — no more `tier1/tier2/tier3`
- [ ] `User.KYCTier` field removed from `auth/model.go`
- [ ] `KYCStatus` values simplified: `none`, `pending`, `verified` (remove `approved`, `rejected` for user-level status)
- [ ] `User.NINHash` field removed — only BVN verification remains
- [ ] `VerifyBVN()` now sets `kyc_status = verified` directly (no tier promotion)
- [ ] `VerifyNIN()` method removed from auth service
- [ ] All BVN/NIN routes updated: remove `/verify-nin` endpoint
- [ ] KYC middleware simplified: check `kyc_status == 'verified'` instead of tier levels
- [ ] Existing KYC tests updated for new model

---

### VES-104 · Add seller bank details to escrow acceptance flow

When a seller accepts escrow terms, they must provide bank account details for fund disbursement.

**Validation Criteria:**
- [ ] `Escrow` model gets new fields: `SellerBankCode`, `SellerAccountNumber`, `SellerAccountName`, `DisbursementStatus`, `DisbursementReference`, `DisbursementRecipientCode`, `DisbursedAt`
- [ ] `AcceptEscrowRequest` DTO requires `bank_code` and `account_number`
- [ ] `AcceptEscrow()` service method validates bank details via Paystack NUBAN resolution API
- [ ] On successful resolution, stores `seller_bank_code`, `seller_account_number`, `seller_account_name` on the escrow
- [ ] Rejects acceptance if bank verification fails (400 error)
- [ ] `DisbursementStatus` defaults to `pending`
- [ ] Acceptance without bank details returns 400 validation error

---

### VES-105 · Refactor escrow release to use Paystack Transfer

Replace wallet-credit-based release with Paystack Transfer API disbursement to seller's bank.

**Validation Criteria:**
- [ ] `ApproveDelivery()` now: records ledger entries (debit escrow, credit seller account, credit fee account) + initiates Paystack Transfer
- [ ] Paystack Transfer flow: create/get transfer recipient → initiate transfer → set `disbursement_status = 'processing'`
- [ ] `payment/service.go` gets new method: `InitiateTransfer(recipientCode, amountKobo, reference)` calling Paystack Transfer API
- [ ] `payment/service.go` gets new method: `CreateTransferRecipient(accountNumber, bankCode, name)` calling Paystack API
- [ ] Transfer reference stored in `escrow.disbursement_reference`
- [ ] Recipient code stored in `escrow.disbursement_recipient_code`
- [ ] Escrow transitions to `completed` with `disbursement_status = 'processing'`
- [ ] If Paystack Transfer initiation fails, escrow still marked `completed` but `disbursement_status = 'failed'` (retry mechanism needed later)

---

### VES-106 · Add Paystack Transfer webhook handling

Handle `transfer.success` and `transfer.failed` webhooks from Paystack.

**Validation Criteria:**
- [ ] Webhook handler recognizes `transfer.success` and `transfer.failed` events
- [ ] `transfer.success` → update escrow `disbursement_status = 'completed'`, set `disbursed_at`
- [ ] `transfer.failed` → update escrow `disbursement_status = 'failed'`
- [ ] Idempotency: processing same webhook twice has no side effects
- [ ] Audit log created for both success and failure
- [ ] Notification sent to seller on successful transfer
- [ ] Notification sent to seller + admin on failed transfer

---

### VES-107 · Refactor dispute resolution to use Paystack Refund & Transfer

Update dispute resolution to use Paystack Refund API (for buyer) and Paystack Transfer (for seller) instead of wallet credits.

**Validation Criteria:**
- [ ] `full_refund` → records ledger entries + initiates Paystack Refund to buyer's original payment method
- [ ] `full_release` → records ledger entries + initiates Paystack Transfer to seller's bank
- [ ] `partial_split` → records ledger entries + Paystack Refund (buyer portion) + Paystack Transfer (seller portion)
- [ ] `payment/service.go` gets new method: `InitiateRefund(transactionReference, amountKobo)` calling Paystack Refund API
- [ ] Partial split validates: `buyer_amount + seller_amount == escrow.amount_kobo`
- [ ] All resolutions create audit logs with disbursement details
- [ ] Dispute resolution returns `disbursement_status` in response

---

### VES-108 · Update auto-release worker for direct disbursement

The auto-release cron worker currently releases funds to seller's wallet. Update it to use Paystack Transfer.

**Validation Criteria:**
- [ ] Auto-release worker calls the refactored `ApproveDelivery()` flow (ledger + Paystack Transfer)
- [ ] No references to user wallets or `wallet.CreditWallet()` remain in worker code
- [ ] Worker handles Paystack Transfer failures gracefully (logs error, marks disbursement as failed, continues to next escrow)
- [ ] Audit log action remains `escrow.auto_released`
- [ ] Worker tests updated to use new mock interfaces

---

### VES-109 · Remove wallet-related endpoints and routes

Remove all wallet/withdrawal HTTP endpoints and routes.

**Validation Criteria:**
- [ ] Removed endpoints: `GET /api/v1/wallet/balance`, `POST /api/v1/wallet/withdraw`, `GET /api/v1/wallet/transactions`, `GET /api/v1/wallet/withdrawals`
- [ ] `wallet/handler.go` deleted or gutted — only ledger admin query endpoints remain (if any)
- [ ] `wallet/routes.go` updated — no user-facing wallet routes
- [ ] All references to wallet handlers removed from main router registration
- [ ] Verify bank account endpoint moved to escrow routes: `POST /api/v1/escrows/verify-bank-account`
- [ ] Bank list endpoint moved to escrow routes: `GET /api/v1/escrows/banks`

---

### VES-110 · Update all tests for new architecture

Update unit tests and integration tests to reflect the wallet-less, tier-less architecture.

**Validation Criteria:**
- [ ] All wallet/withdrawal test files deleted or rewritten for ledger-only tests
- [ ] Ledger integrity test updated: tests system accounts only, no user wallets
- [ ] Escrow lifecycle test updated: approve-delivery now checks Paystack Transfer mock
- [ ] Auth registration test: no wallet creation expected
- [ ] KYC tests: no tier promotion, only BVN verification → `verified`
- [ ] Dispute tests: resolution uses Paystack Refund/Transfer mocks
- [ ] `go test ./...` passes with zero failures
- [ ] Coverage for critical paths (escrow, ledger, payment) > 80%

---

### VES-111 · Database migration script

Create a migration to transition existing database from wallet-based to system-account-based schema.

**Validation Criteria:**
- [ ] Migration renames `wallets` table to `system_accounts` (or creates new + migrates data)
- [ ] Migration removes all user-wallet rows, keeps only `escrow` and `platform_fee` system accounts
- [ ] Migration renames `ledger_entries.wallet_id` to `ledger_entries.account_id`
- [ ] Migration drops `withdrawals` table
- [ ] Migration removes `kyc_tier` column from `users` table
- [ ] Migration removes `nin_hash` column from `users` table
- [ ] Migration adds new columns to `escrows`: `seller_bank_code`, `seller_account_number`, `seller_account_name`, `disbursement_status`, `disbursement_reference`, `disbursement_recipient_code`, `disbursed_at`
- [ ] Migration is reversible (down migration provided)
- [ ] Migration runs successfully on a fresh database via `AutoMigrate`

---

### VES-112 · End-to-end refactor verification

Verify the complete refactored system works end-to-end.

**Validation Criteria:**
- [ ] `go build ./...` succeeds
- [ ] `go vet ./...` clean
- [ ] `go test ./...` all pass
- [ ] No references to `wallet.Wallet`, `WalletType`, `KYCTier`, `Withdrawal`, `WithdrawalStatus` remain in source code
- [ ] Grep for `"wallet"` in handler/service/route files returns zero user-wallet hits (only ledger/system-account references)
- [ ] Manual test: create escrow → submit → accept (with bank) → fund → deliver → approve → verify Paystack Transfer mock called
- [ ] Manual test: dispute → resolve full_refund → verify Paystack Refund mock called
- [ ] Ledger integrity: `SUM(credits) - SUM(debits) = 0`

---

## Phase 14 — Admin Module

Admin dashboard, user management, and platform settings.

---

### VES-076 · Admin service (`internal/admin/service.go`)

Admin business logic.

**Validation Criteria:**
- [ ] `GetDashboardMetrics()` returns: total users, active escrows by status, total volume (kobo), pending KYC count, open disputes count, revenue (platform_fee system account balance)
- [ ] `ListUsers(page, perPage, filters)` → paginated, filterable by `status`, `role`, `kyc_status`
- [ ] `GetUserDetails(userID)` → full profile + escrow count
- [ ] `SuspendUser(userID)` → sets `status = "suspended"`, creates audit log
- [ ] `UnsuspendUser(userID)` → sets `status = "active"`, creates audit log
- [ ] `BanUser(userID)` → sets `status = "banned"`, requires `super_admin` role
- [ ] `GetAllEscrows(filters)` → admin view with status + date + disbursement_status filters
- [ ] `GetAllDisputes(filters)` → admin view with status filter
- [ ] `GetPlatformSettings()` → returns all key-value settings
- [ ] `UpdatePlatformSettings(key, value)` → updates setting, requires `super_admin`
- [ ] `GetRevenueReport(from, to)` → total fees, daily breakdown

---

### VES-077 · Admin handler (`internal/admin/handler.go`)

HTTP handlers for admin endpoints.

**Validation Criteria:**
- [ ] `GET  /admin/dashboard` → dashboard metrics
- [ ] `GET  /admin/users` → paginated user list
- [ ] `GET  /admin/users/:id` → user details
- [ ] `PUT  /admin/users/:id/suspend` → suspend user
- [ ] `PUT  /admin/users/:id/unsuspend` → unsuspend user
- [ ] `PUT  /admin/users/:id/ban` → ban user (super_admin only)
- [ ] `GET  /admin/escrows` → all escrows (with disbursement_status filter)
- [ ] `GET  /admin/disputes` → all disputes
- [ ] `GET  /admin/audit-logs` → audit logs (super_admin only)
- [ ] `GET  /admin/settings` → platform settings
- [ ] `PUT  /admin/settings` → update settings (super_admin only)
- [ ] `GET  /admin/revenue` → revenue report
- [ ] All responses use standardized format

---

### VES-078 · Admin routes (`internal/admin/routes.go`)

Register routes under `/api/v1/admin`.

**Validation Criteria:**
- [ ] All routes require JWT + admin middleware
- [ ] `PUT /admin/users/:id/ban` requires super_admin check
- [ ] `GET /admin/audit-logs` requires super_admin check
- [ ] `PUT /admin/settings` requires super_admin check
- [ ] Regular admin users get 403 on super_admin-only endpoints

---

## Phase 15 — Health Check & Graceful Shutdown

Production readiness endpoints.

---

### VES-079 · Health check endpoint

`GET /api/v1/health` — checks PostgreSQL and Redis connectivity.

**Validation Criteria:**
- [ ] Returns 200 `{"status": "healthy", "database": "connected", "redis": "connected"}` when all services OK
- [ ] Returns 503 `{"status": "unhealthy", ...}` with details when any dependency is down
- [ ] Checks are lightweight (ping, not full query)
- [ ] Endpoint requires no authentication

---

### VES-080 · API version endpoint

`GET /api/v1/version` — returns app metadata.

**Validation Criteria:**
- [ ] Returns: app version, Go version, build time
- [ ] Version and build time embedded at compile time via `-ldflags`
- [ ] Endpoint requires no authentication

---

### VES-081 · Graceful shutdown

Handle `SIGINT`/`SIGTERM` signals for clean server shutdown.

**Validation Criteria:**
- [ ] `kill -TERM <pid>` triggers clean shutdown
- [ ] Log message: `"Shutting down server..."` and then `"Server shutdown complete"`
- [ ] In-flight requests complete before shutdown (up to 30s timeout)
- [ ] Database connection pool closed
- [ ] Redis connection closed

---

## Phase 16 — Security Hardening

PII masking, security headers, input sanitization.

---

### VES-082 · API response PII masking (`internal/common/masking.go`)

Utility functions to mask sensitive data in API responses.

**Validation Criteria:**
- [ ] `MaskPhone("+2348031234567")` → `"+234803****567"`
- [ ] `MaskEmail("seun.adebayo@example.com")` → `"s***@example.com"`
- [ ] `MaskBVN("22233344455")` → `"****4455"`
- [ ] `MaskAccountNumber("0123456789")` → `"****6789"`
- [ ] All API responses use masked versions — full values never returned

---

### VES-083 · Request body size limits

Configure Fiber body limits.

**Validation Criteria:**
- [ ] Default JSON body limit: 1 MB
- [ ] File upload body limit: 10 MB (on upload endpoints)
- [ ] Oversized requests return 413 `"Request body too large"`

---

### VES-084 · Security headers middleware

Add security headers to all responses.

**Validation Criteria:**
- [ ] `X-Content-Type-Options: nosniff` present in every response
- [ ] `X-Frame-Options: DENY` present in every response
- [ ] `Strict-Transport-Security: max-age=31536000; includeSubDomains` in production only
- [ ] `X-Request-ID: <uuid>` generated per request for tracing

---

### VES-085 · Input sanitization

Strip dangerous content from all text inputs.

**Validation Criteria:**
- [ ] HTML tags stripped from: escrow titles, descriptions, messages, dispute content, comments
- [ ] `<script>alert('xss')</script>` stripped to `alert('xss')`
- [ ] Normal text with special characters preserved (e.g., `₦1,000 & more`)
- [ ] Applied as a utility function, called in handlers before passing to services

---

## Phase 17 — Structured Logging

Production-grade JSON logging.

---

### VES-086 · Structured logger setup

Configure `log/slog` or `zerolog` for structured logging.

**Validation Criteria:**
- [ ] Development mode: colored, human-readable log output
- [ ] Production mode (`ENVIRONMENT=production`): JSON-formatted logs to stdout
- [ ] Each log entry includes: timestamp, level, module name
- [ ] Supports log levels: DEBUG, INFO, WARN, ERROR
- [ ] Module context: `auth`, `escrow`, `payment`, `wallet`, `dispute`, `admin`, `audit`

---

### VES-087 · Request logging middleware

Log every HTTP request with performance data.

**Validation Criteria:**
- [ ] Every request produces a log line with: method, path, status code, latency (ms), client IP, user agent, request ID
- [ ] Sensitive data never logged: passwords, tokens, full PII
- [ ] Latency measured accurately (start → response sent)
- [ ] Log level: INFO for success (2xx), WARN for client errors (4xx), ERROR for server errors (5xx)

---

## Phase 18 — Testing

Unit tests, integration tests, and coverage.

---

### VES-088 · Unit tests: common utilities

Test JWT, password, OTP, pagination, validation, encryption.

**Validation Criteria:**
- [ ] `go test ./internal/common/...` passes
- [ ] Coverage > 80% for `internal/common` package
- [ ] Tests: JWT generation, parsing, expiry detection, invalid token handling
- [ ] Tests: password hash and verify (correct + incorrect)
- [ ] Tests: OTP generation format (6 digits), storage, verification, single-use
- [ ] Tests: encryption roundtrip, wrong key rejection
- [ ] Tests: pagination offset calculation, total pages calculation

---

### VES-089 · Unit tests: escrow state machine

Test all valid and invalid state transitions.

**Validation Criteria:**
- [ ] `go test ./internal/escrow/...` — state machine tests pass
- [ ] Every valid transition has a test case asserting `true`
- [ ] Every invalid transition (e.g., `completed → draft`) has a test case asserting `false`
- [ ] All 13 states covered
- [ ] Terminal states verified: no outgoing transitions

---

### VES-090 · Integration tests: auth flow

Test against real PostgreSQL + Redis.

**Validation Criteria:**
- [ ] `go test -tags=integration ./internal/auth/...` passes
- [ ] Test: register → verify email → login → get profile → refresh → change password
- [ ] Test: login with wrong password → 401
- [ ] Test: login before email verification → rejected
- [ ] Test: duplicate registration → error
- [ ] Uses test database (not production)

---

### VES-091 · Integration tests: escrow lifecycle

Full escrow lifecycle with wallet operations.

**Validation Criteria:**
- [ ] Test: create → submit → accept → fund → deliver → approve → completed
- [ ] Wallet balances verified at each step
- [ ] Ledger integrity verified: `SUM(credits) == SUM(debits)` = 0
- [ ] Disbursement status transitions verified
- [ ] Test: create with buyer == seller → rejected
- [ ] Test: invalid state transition → rejected

---

### VES-092 · Integration tests: dispute flow

Full dispute lifecycle.

**Validation Criteria:**
- [ ] Test: open dispute on delivered escrow → success
- [ ] Test: open dispute on non-delivered escrow → rejected
- [ ] Test: upload evidence → visible in dispute details
- [ ] Test: admin resolves as partial_split → correct Paystack Refund/Transfer disbursements

---

### VES-093 · Idempotency tests

Test payment and webhook idempotency.

**Validation Criteria:**
- [ ] Same Paystack webhook processed twice → only one set of ledger entries created
- [ ] Same payment initialized twice with same idempotency key → returns existing payment, no duplicate
- [ ] Ledger entry reference uniqueness enforced

---

### VES-094 · Test coverage report

Generate and review test coverage.

**Validation Criteria:**
- [ ] `go test -coverprofile=coverage.out ./...` generates report
- [ ] `go tool cover -html=coverage.out` opens viewable HTML report
- [ ] Overall coverage > 70%
- [ ] Critical paths (auth, escrow state machine, ledger, payment) > 80%

---

## Phase 19 — CI/CD & Production Deployment

Continuous integration, Docker builds, and production readiness.

---

### VES-095 · GitHub Actions CI pipeline (`.github/workflows/ci.yml`)

Automated testing on push/PR.

**Validation Criteria:**
- [ ] Triggers on: push to `main`, pull requests to `main`
- [ ] Steps: checkout → setup Go → `go vet ./...` → `go test ./...` → `go build ./...`
- [ ] Lint with `golangci-lint` (uses `golangci/golangci-lint-action`)
- [ ] Pipeline passes for clean code
- [ ] Pipeline fails on test failures or lint errors

---

### VES-096 · GitHub Actions CD pipeline (`.github/workflows/deploy.yml`)

Automated deployment on merge to main.

**Validation Criteria:**
- [ ] Triggers on: push to `main` (after CI passes)
- [ ] Steps: build Docker image → push to `ghcr.io` → deploy to VPS via SSH
- [ ] Image tagged with both `latest` and commit SHA
- [ ] Matches the deployment guide specification
- [ ] Uses GitHub Secrets for: `PROD_HOST`, `PROD_USER`, `SSH_PRIVATE_KEY`

---

### VES-097 · Production docker-compose (`docker-compose.prod.yml`)

Production stack with Traefik, SSL, hardened config.

**Validation Criteria:**
- [ ] Includes: Traefik reverse proxy, Vescrow app, PostgreSQL, Redis
- [ ] HTTPS with Let's Encrypt auto-renewal via Traefik
- [ ] PostgreSQL tuned: `max_connections=200`, `shared_buffers=1GB`
- [ ] Redis requires password authentication
- [ ] Matches the production spec in deployment-guide.md

---

### VES-098 · Database backup cron

Daily automated pg_dump with retention policy.

**Validation Criteria:**
- [ ] Script: daily `pg_dump` compressed with gzip
- [ ] Retention: delete backups older than 30 days
- [ ] Backup permissions: `chmod 600`
- [ ] Optional: upload to S3-compatible storage
- [ ] Matches the backup policy in deployment-guide.md

---

### VES-099 · Production checklist verification

Final verification of all production requirements.

**Validation Criteria:**
- [ ] `ENVIRONMENT=production` set
- [ ] Strong, unique `DB_PASSWORD` (not default)
- [ ] Strong `ENCRYPTION_KEY` (32 bytes, hex-encoded)
- [ ] Paystack live keys configured (`sk_live_xxx`)
- [ ] HTTPS configured and working
- [ ] Database backups running daily
- [ ] Log aggregation configured (stdout JSON → log shipper)
- [ ] CORS restricted to production domains only
- [ ] 4096-bit RSA keys generated (not 2048-bit dev keys)
- [ ] Health check endpoint responding correctly

---

## Summary

| Phase | Name | Task IDs | Count |
|-------|------|----------|-------|
| 1 | Project Scaffolding | VES-001 → VES-008 | 8 |
| 2 | Database & Models | VES-009 → VES-019 | 11 |
| 3 | Utilities & Middleware | VES-020 → VES-032 | 13 |
| 4 | Authentication | VES-033 → VES-037 | 5 |
| 5 | Escrow Module | VES-038 → VES-043 | 6 |
| 6 | Payment (Paystack) | VES-044 → VES-049 | 6 |
| 7 | Wallet & Ledger | VES-050 → VES-055 | 6 |
| 8 | Disputes | VES-056 → VES-060 | 5 |
| 9 | Notifications | VES-061 → VES-065 | 5 |
| 10 | Audit Logging | VES-066 → VES-068 | 3 |
| 11 | KYC Verification | VES-069 → VES-072 | 4 |
| 12 | Auto-Release Cron | VES-073 → VES-075 | 3 |
| **13** | **⚡ Architecture Simplification** | **VES-100 → VES-112** | **13** |
| 14 | Admin Module | VES-076 → VES-078 | 3 |
| 15 | Health & Shutdown | VES-079 → VES-081 | 3 |
| 16 | Security Hardening | VES-082 → VES-085 | 4 |
| 17 | Structured Logging | VES-086 → VES-087 | 2 |
| 18 | Testing | VES-088 → VES-094 | 7 |
| 19 | CI/CD & Deployment | VES-095 → VES-099 | 5 |
| **Total** | | **VES-001 → VES-112** | **112** |

---

> **Architecture Refactor milestone**: Phase 13 (VES-100 → VES-112) must be completed before Phases 14-19. It removes user wallets and KYC tiers, replacing them with direct escrow disbursement via Paystack Transfer API.

> **MVP milestone**: After completing **Phases 1–7** (VES-001 → VES-055), you'll have a working escrow platform with authentication, full transaction lifecycle, Paystack payments, and ledger operations. This is enough to demo and test with real users using Paystack test keys.

---

**Document Version**: 2.0
**Last Updated**: August 2026
