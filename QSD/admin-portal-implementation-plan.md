# Vescrow Admin Portal — Implementation Plan

**Task ID**: `VESC-ADMIN-2026-001`
**Created**: 2026-09-20 (Revised: separate application)
**Status**: Draft — Awaiting Approval

---

## Architecture Overview

The admin portal is a **completely separate Next.js application** — its own project, its own `node_modules`, its own dev server, its own deployment target.

```
vescrow/
├── vescrow_backend/        ← Go/Fiber API   → port 3000
├── vescrow_frontend/       ← User app        → port 3001
├── vescrow_admin/          ← Admin portal    → port 3002  [NEW]
└── docsplan/
```

### Why Separate Application?

| Concern | Benefit |
|---------|---------|
| **Security isolation** | Admin bundle never ships to end-users. Different CSP, different cookie domain |
| **Independent deployment** | Admin can be deployed internally / behind VPN; user app stays public |
| **Clean codebase** | No role-conditional rendering clutter in user app |
| **Independent CI/CD** | Admin deploys can be gated separately from user-facing releases |
| **Port separation** | Admin on `:3002`, user on `:3001`, backend on `:3000` |

### Communication

```
vescrow_admin (port 3002)
        │
        │ HTTP/HTTPS (JWT Bearer in Authorization header)
        │ via Next.js API proxy rewrites (/api/* → backend)
        ▼
vescrow_backend (port 3000)
        │
        ├── /api/v1/auth/*
        ├── /api/v1/admin/*   ← All admin endpoints
        ├── /api/v1/disputes/*
        └── /api/v1/kyc/*
```

---

## System Analysis — What Already Exists

### ✅ Backend: Already Built & Usable

| Endpoint | Location | Status |
|----------|----------|--------|
| `POST /auth/login` | `auth/handler.go` | ✅ Ready |
| `POST /auth/logout` | `auth/handler.go` | ✅ Ready |
| `GET /admin/kyc/submissions` | `admin/handler.go` | ✅ Ready |
| `GET /admin/kyc/submissions/:id` | `admin/handler.go` | ✅ Ready |
| `PUT /admin/kyc/submissions/:id/approve` | `admin/handler.go` | ✅ Ready |
| `PUT /admin/kyc/submissions/:id/reject` | `admin/handler.go` | ✅ Ready |
| `GET /admin/audit-logs` | `admin/audit_handler.go` | ✅ Ready |
| `PUT /disputes/:id/resolve` | `dispute/handler.go` | ✅ Ready |
| `GET /disputes` (admins see all) | `dispute/handler.go` | ✅ Partial |

### ❌ Backend: 13 Missing Endpoints

All documented in README but not yet in `admin/handler.go`:

| Endpoint | Priority |
|----------|----------|
| `GET /admin/dashboard` | 🔴 Critical |
| `GET /admin/users` | 🔴 Critical |
| `GET /admin/users/:id` | 🔴 Critical |
| `PUT /admin/users/:id/suspend` | 🟠 High |
| `PUT /admin/users/:id/unsuspend` | 🟠 High |
| `PUT /admin/users/:id/ban` | 🟠 High |
| `GET /admin/escrows` | 🔴 Critical |
| `GET /admin/escrows/:id` | 🟠 High |
| `GET /admin/disputes` | 🔴 Critical |
| `GET /admin/withdrawals` | 🟠 High |
| `GET /admin/revenue` | 🔴 Critical |
| `GET /admin/settings` | 🟡 Medium |
| `PUT /admin/settings` | 🟡 Medium |

### ✅ Frontend Reference (vescrow_frontend)

The admin app will **not** share code with `vescrow_frontend` but will mirror its conventions:
- Same design language: dark slate base, CSS custom properties
- Same dependency choices: Next.js 16, Zustand, Lucide icons, Framer Motion
- Same API response contract: `{ success, message, data }` envelope
- Types duplicated (or extracted to a shared `@vescrow/types` package in future)

---

## Open Questions

> [!IMPORTANT]
> **Admin Login**: Should admins use the same `/api/v1/auth/login` endpoint (role-checked post-login), or should a dedicated `/api/v1/admin/auth/login` endpoint exist that enforces admin-only access at the backend? **Recommended: reuse existing login + validate `role` client-side before granting portal access. Simpler, no backend change needed.**

> [!IMPORTANT]
> **Backend CORS**: The backend currently has a CORS config. It needs to allow `http://localhost:3002` (admin dev) and the admin production domain. Confirm which admin domain will be used in production.

> [!WARNING]
> **Admin MFA**: `AdminMFARequired()` middleware is active on all `/admin/*` routes. This requires admin users to have 2FA set up. Does the admin portal need a TOTP 2FA enrollment & verification flow? **If yes, this is a Phase 2 addition.**

> [!NOTE]
> **Shared Types**: TypeScript interfaces for `User`, `Escrow`, `Dispute`, `AuditLog`, etc. will be duplicated in `vescrow_admin`. In the future, these can be extracted to a monorepo package. For now, keeping them co-located in each app is simpler.

---

## Phase 1: Backend API Completion

**Goal**: Implement all 13 missing admin endpoints in the Go backend.
**Estimated effort**: 3–4 days
**Files**: `vescrow_backend/internal/admin/`

---

### Task 1.1 — Service & Repository Layer

#### [NEW] `vescrow_backend/internal/admin/service.go`
#### [NEW] `vescrow_backend/internal/admin/repository.go`

The current `admin` package only has a handler that calls into `auth.AuthService`. We need dedicated service + repository files for the new admin-specific business logic.

**`AdminRepository`** — raw DB queries:
```go
type AdminRepository struct { db *gorm.DB }

func (r *AdminRepository) CountUsers(ctx context.Context, filters UserFilters) (int64, error)
func (r *AdminRepository) ListUsers(ctx context.Context, filters UserFilters, page, perPage int) ([]auth.User, int64, error)
func (r *AdminRepository) GetUserByID(ctx context.Context, id uuid.UUID) (*auth.User, error)
func (r *AdminRepository) UpdateUserStatus(ctx context.Context, id uuid.UUID, status auth.UserStatus, reason string, adminID uuid.UUID) error
func (r *AdminRepository) ListEscrows(ctx context.Context, filters EscrowFilters, page, perPage int) ([]escrow.Escrow, int64, error)
func (r *AdminRepository) GetEscrowByID(ctx context.Context, id uuid.UUID) (*escrow.Escrow, error)
func (r *AdminRepository) GetDashboardMetrics(ctx context.Context) (*DashboardMetrics, error)
func (r *AdminRepository) GetRevenueReport(ctx context.Context, from, to time.Time) (*RevenueReport, error)
func (r *AdminRepository) ListDisputes(ctx context.Context, filters DisputeFilters, page, perPage int) ([]dispute.Dispute, int64, error)
func (r *AdminRepository) ListWithdrawals(ctx context.Context, filters WithdrawalFilters, page, perPage int) ([]wallet.Withdrawal, int64, error)
func (r *AdminRepository) GetSettings(ctx context.Context) ([]PlatformSetting, error)
func (r *AdminRepository) UpsertSetting(ctx context.Context, key, value string) error
```

**Validation Criteria**:
- [ ] All repository methods use parameterized queries (GORM)
- [ ] Service and repository files compile without errors

---

### Task 1.2 — Dashboard Endpoint

**File**: `vescrow_backend/internal/admin/handler.go` [MODIFY]

**Implement**: `GET /api/v1/admin/dashboard`

```go
type DashboardMetrics struct {
    // Users
    TotalUsers         int64 `json:"total_users"`
    ActiveUsers        int64 `json:"active_users"`
    SuspendedUsers     int64 `json:"suspended_users"`
    NewUsersToday      int64 `json:"new_users_today"`
    NewUsersThisMonth  int64 `json:"new_users_this_month"`
    KYCPendingCount    int64 `json:"kyc_pending_count"`

    // Escrows
    TotalEscrows        int64 `json:"total_escrows"`
    ActiveEscrows       int64 `json:"active_escrows"`
    DisputedEscrows     int64 `json:"disputed_escrows"`
    CompletedEscrows    int64 `json:"completed_escrows"`
    TotalVolumeKobo     int64 `json:"total_volume_kobo"`
    VolumeThisMonthKobo int64 `json:"volume_this_month_kobo"`

    // Revenue (from ledger_entries WHERE category = 'platform_fee')
    TotalRevenueKobo     int64 `json:"total_revenue_kobo"`
    RevenueThisMonthKobo int64 `json:"revenue_this_month_kobo"`
    RevenueThisWeekKobo  int64 `json:"revenue_this_week_kobo"`

    // Disputes
    TotalDisputes     int64   `json:"total_disputes"`
    OpenDisputes      int64   `json:"open_disputes"`
    ResolvedDisputes  int64   `json:"resolved_disputes"`
    AvgResolutionDays float64 `json:"avg_resolution_days"`

    // Withdrawals
    PendingWithdrawals          int64 `json:"pending_withdrawals"`
    PendingWithdrawalVolumeKobo int64 `json:"pending_withdrawal_volume_kobo"`

    // 30-day chart data
    DailyVolumeChart  []DailyMetric `json:"daily_volume_chart"`
    DailyRevenueChart []DailyMetric `json:"daily_revenue_chart"`
}

type DailyMetric struct {
    Date       string `json:"date"`
    AmountKobo int64  `json:"amount_kobo"`
    Count      int64  `json:"count"`
}
```

**Validation Criteria**:
- [ ] 200 with all fields populated
- [ ] 401 without token, 403 for `user` role token
- [ ] Volume from `escrows WHERE status IN ('completed', 'partial_release')`
- [ ] Revenue from `ledger_entries WHERE category = 'platform_fee'`
- [ ] Daily chart has exactly 30 entries (fill 0 for days with no activity)

---

### Task 1.3 — User Management Endpoints

**File**: `vescrow_backend/internal/admin/handler.go` [MODIFY]

#### `GET /api/v1/admin/users`
Params: `page`, `per_page`, `status`, `kyc_status`, `search`, `role`, `sort_by`, `sort_dir`
Response: paginated users with role, kyc_status, wallet balance sum, escrow count

#### `GET /api/v1/admin/users/:id`
Returns: full user + KYC documents + buyer/seller wallet balances + last 5 escrows + last 10 audit logs

#### `PUT /api/v1/admin/users/:id/suspend`
Body: `{ "reason": "string" }` (required, min 10 chars)
- Sets `status = 'suspended'`, writes audit log, sends notification to user

#### `PUT /api/v1/admin/users/:id/unsuspend`
- Sets `status = 'active'`, writes audit log

#### `PUT /api/v1/admin/users/:id/ban`
Requires `super_admin` role via `middleware.RequirePermission("ban_user")`
Body: `{ "reason": "string" }` (required)
- Sets `status = 'banned'`, freezes all user wallets (`is_frozen = true`), writes audit log

**Validation Criteria**:
- [ ] Pagination correct
- [ ] Search does ILIKE on name, email, phone
- [ ] User detail includes wallet balances
- [ ] Suspend/ban writes to audit_logs with actor_id
- [ ] Ban requires super_admin → 403 for admin role
- [ ] Notifications dispatched (fire-and-forget goroutine)

---

### Task 1.4 — Escrow Management Endpoints

**File**: `vescrow_backend/internal/admin/handler.go` [MODIFY]

#### `GET /api/v1/admin/escrows`
Params: `page`, `per_page`, `status`, `min_amount_kobo`, `max_amount_kobo`, `date_from`, `date_to`, `search`
Response: paginated escrows with buyer/seller name + email

#### `GET /api/v1/admin/escrows/:id`
Returns: full escrow + buyer + seller + milestones + payments + dispute (if any) + ledger entries + last 20 audit logs

**Validation Criteria**:
- [ ] All filters combine with AND logic
- [ ] Detail endpoint preloads all relations in single query (use GORM `Preload`)

---

### Task 1.5 — Dispute Management Endpoint

**File**: `vescrow_backend/internal/admin/handler.go` [MODIFY]

#### `GET /api/v1/admin/disputes`
Params: `page`, `per_page`, `status`, `date_from`, `date_to`, `search`
Response: paginated disputes with `days_open` computed field, buyer/seller names from joined escrow

> This is a dedicated admin endpoint with full filters — distinct from the user-facing `GET /disputes`.

**Validation Criteria**:
- [ ] `days_open = CURRENT_DATE - opened_at` included in response
- [ ] Status filter works
- [ ] Search matches dispute reason or escrow title

---

### Task 1.6 — Withdrawals Endpoint

**File**: `vescrow_backend/internal/admin/handler.go` [MODIFY]

#### `GET /api/v1/admin/withdrawals`
Params: `page`, `per_page`, `status`, `date_from`, `date_to`
Response: paginated withdrawals with user name + email joined from users table

**Validation Criteria**:
- [ ] Status filter works
- [ ] User info joined correctly

---

### Task 1.7 — Revenue & Settings Endpoints

**File**: `vescrow_backend/internal/admin/handler.go` [MODIFY]

#### `GET /api/v1/admin/revenue`
Params: `period` (7d/30d/90d/1y), `date_from`, `date_to`

```go
type RevenueReport struct {
    TotalRevenueKobo       int64         `json:"total_revenue_kobo"`
    TotalVolumeKobo        int64         `json:"total_volume_kobo"`
    TransactionCount       int64         `json:"transaction_count"`
    AverageTransactionKobo int64         `json:"average_transaction_kobo"`
    DailyBreakdown         []DailyMetric `json:"daily_breakdown"`
}
```

#### `GET /api/v1/admin/settings`
Returns all `platform_settings` rows as `{ [key]: value }` map

#### `PUT /api/v1/admin/settings`
Requires `super_admin`. Validates keys against whitelist. Creates audit log per changed key.

Settable keys: `platform_fee_percent`, `default_inspection_days`, `max_inspection_days`, `daily_withdrawal_limit_kobo`, `per_tx_withdrawal_limit_kobo`, `kyc_tier1_limit_kobo`, `kyc_tier2_limit_kobo`

**Validation Criteria**:
- [ ] Revenue sourced from `ledger_entries WHERE category = 'platform_fee'`
- [ ] Settings PUT validates key whitelist
- [ ] Settings PUT requires super_admin → 403 for admin
- [ ] Changes audited

---

### Task 1.8 — Update Routes & Tests

#### [MODIFY] `vescrow_backend/internal/admin/routes.go`

Register all new endpoints:
```go
adminGroup.Get("/dashboard", handler.GetDashboard)
adminGroup.Get("/users", handler.ListUsers)
adminGroup.Get("/users/:id", handler.GetUser)
adminGroup.Put("/users/:id/suspend", handler.SuspendUser)
adminGroup.Put("/users/:id/unsuspend", handler.UnsuspendUser)
adminGroup.Put("/users/:id/ban", middleware.RequirePermission("ban_user"), handler.BanUser)
adminGroup.Get("/escrows", handler.ListEscrows)
adminGroup.Get("/escrows/:id", handler.GetEscrow)
adminGroup.Get("/disputes", handler.ListDisputes)
adminGroup.Get("/withdrawals", handler.ListWithdrawals)
adminGroup.Get("/revenue", handler.GetRevenue)
adminGroup.Get("/settings", handler.GetSettings)
adminGroup.Put("/settings", middleware.RequirePermission("update_settings"), handler.UpdateSettings)
```

#### [MODIFY] `vescrow_backend/internal/admin/admin_test.go`

Add tests for all new endpoints covering 401, 403, 200 cases.

Also update **`vescrow_backend/internal/middleware/cors.go`** (or equivalent CORS config) to allow `http://localhost:3002` as an allowed origin for development.

**Validation Criteria**:
- [ ] `go test ./internal/admin/...` passes
- [ ] `go build ./...` compiles cleanly
- [ ] CORS allows requests from port 3002

---

## Phase 2: Admin Application Scaffolding & Auth

**Goal**: Bootstrap the `vescrow_admin` Next.js project, configure it, and build the auth layer.
**Estimated effort**: 2–3 days

---

### Task 2.1 — Project Bootstrap

#### [NEW] `vescrow_admin/` directory

```bash
# Scaffold in the monorepo root
cd /Users/oluwatobiloba/Desktop/qsd/vescrow
npx -y create-next-app@latest vescrow_admin \
  --typescript --eslint --no-tailwind --no-src-dir \
  --no-app --import-alias "@/*"
```

Then **manually restructure** to use App Router with src/:
```
vescrow_admin/
├── src/
│   ├── app/
│   │   ├── layout.tsx
│   │   ├── globals.css
│   │   ├── (auth)/
│   │   │   └── login/
│   │   │       └── page.tsx
│   │   └── (portal)/
│   │       ├── layout.tsx          ← Admin shell
│   │       ├── dashboard/page.tsx
│   │       ├── users/
│   │       │   ├── page.tsx
│   │       │   └── [id]/page.tsx
│   │       ├── kyc/page.tsx
│   │       ├── escrows/
│   │       │   ├── page.tsx
│   │       │   └── [id]/page.tsx
│   │       ├── disputes/
│   │       │   ├── page.tsx
│   │       │   └── [id]/page.tsx
│   │       ├── withdrawals/page.tsx
│   │       ├── reports/page.tsx
│   │       ├── audit-logs/page.tsx
│   │       └── settings/page.tsx
│   ├── components/
│   │   ├── ui/                     ← Primitives (Button, Card, Badge, Input, Modal, Table)
│   │   ├── layout/                 ← AdminSidebar, AdminTopbar
│   │   └── admin/                  ← Domain-specific components
│   ├── lib/
│   │   ├── api/
│   │   │   ├── client.ts           ← HTTP client (fetch wrapper + JWT)
│   │   │   └── admin.ts            ← All admin API functions
│   │   ├── types.ts                ← TypeScript interfaces
│   │   ├── constants.ts            ← Routes, status configs
│   │   └── utils.ts                ← formatCurrency, formatDate, cn()
│   ├── stores/
│   │   ├── auth-store.ts
│   │   └── admin-store.ts
│   └── middleware.ts               ← Route guard
├── package.json
├── next.config.ts
├── tsconfig.json
└── .env.local
```

#### [NEW] `vescrow_admin/package.json`
```json
{
  "name": "vescrow_admin",
  "version": "0.1.0",
  "scripts": {
    "dev": "next dev -p 3002",
    "build": "next build",
    "start": "next start -p 3002",
    "lint": "eslint"
  },
  "dependencies": {
    "next": "16.3.1",
    "react": "19.2.8",
    "react-dom": "19.2.8",
    "zustand": "^5.0.15",
    "lucide-react": "^1.31.0",
    "recharts": "^2.15.0",
    "clsx": "^2.1.1",
    "date-fns": "^4.4.0",
    "framer-motion": "^13.1.0",
    "react-hot-toast": "^2.6.0",
    "js-cookie": "^3.0.8"
  },
  "devDependencies": {
    "@tailwindcss/postcss": "^4.3.3",
    "tailwindcss": "^4.3.3",
    "typescript": "^5",
    "@types/react": "^19",
    "@types/react-dom": "^19",
    "@types/node": "^20",
    "@types/js-cookie": "^3.0.6"
  }
}
```

> `recharts` is added specifically for the analytics charts (line/bar charts on the dashboard and reports pages).

#### [NEW] `vescrow_admin/next.config.ts`
```typescript
const apiOrigin = process.env.API_ORIGIN || 'http://localhost:3000';

const nextConfig = {
  async rewrites() {
    return [{ source: '/api/:path*', destination: `${apiOrigin}/api/:path*` }];
  },
};
export default nextConfig;
```

#### [NEW] `vescrow_admin/.env.local`
```bash
API_ORIGIN=http://localhost:3000
NEXT_PUBLIC_APP_NAME=Vescrow Admin
NEXT_PUBLIC_USER_APP_URL=http://localhost:3001
```

**Validation Criteria**:
- [ ] `npm install` runs without errors
- [ ] `npm run dev` starts on port 3002 without errors
- [ ] TypeScript compiles cleanly (`npx tsc --noEmit`)
- [ ] Proxy: `curl http://localhost:3002/api/v1/auth/login` reaches backend

---

### Task 2.2 — Design System (Admin CSS)

#### [NEW] `vescrow_admin/src/app/globals.css`

Admin-specific design tokens. Same dark slate foundation as `vescrow_frontend` but with **purple/indigo as the admin accent** (instead of emerald) so it's visually distinct from the user app:

```css
@import "tailwindcss";

:root {
  /* Admin accent — Indigo/Purple (distinct from user app's emerald) */
  --admin-accent: #7C3AED;
  --admin-accent-light: #A78BFA;
  --admin-accent-glow: rgba(124, 58, 237, 0.25);

  /* Status colors */
  --success: #10B981;
  --warning: #F59E0B;
  --danger: #EF4444;
  --info: #3B82F6;

  /* Background */
  --bg-main: #020617;
  --bg-elevated: rgba(15, 23, 42, 0.9);
  --bg-card: rgba(15, 23, 42, 0.85);
  --bg-sidebar: #080F1E;

  /* Text */
  --text-primary: #F8FAFC;
  --text-secondary: #94A3B8;
  --text-tertiary: #64748B;

  /* Border */
  --border: rgba(255, 255, 255, 0.1);
  --border-subtle: rgba(255, 255, 255, 0.06);

  /* Radius */
  --radius: 14px;
  --radius-lg: 16px;
  --radius-full: 9999px;
}
```

Plus admin-specific utility classes: `.admin-card`, `.admin-glass`, `.status-badge`, `.sla-badge`, `.admin-table`, `.admin-sidebar`, `.admin-topbar`, animation keyframes identical to `vescrow_frontend`.

**Validation Criteria**:
- [ ] CSS variables available globally
- [ ] Inter font loaded from Google Fonts
- [ ] Purple accent visible in buttons and active states

---

### Task 2.3 — TypeScript Types & API Client

#### [NEW] `vescrow_admin/src/lib/types.ts`

Full type definitions mirroring the backend API spec:

```typescript
// Reuse same types from vescrow_frontend (copy & maintain independently):
// User, UserRole, UserStatus, KYCStatus, KYCDocument
// Escrow, EscrowStatus, EscrowMilestone, EscrowPayment
// Dispute, DisputeStatus, DisputeResolution, DisputeEvidence, DisputeComment
// Wallet, LedgerEntry, Withdrawal
// AuditLog, PaginatedResponse<T>, ApiResponse<T>, PaginationMeta

// Admin-specific additions:
export interface AdminDashboardMetrics {
  total_users: number; active_users: number; suspended_users: number;
  new_users_today: number; new_users_this_month: number; kyc_pending_count: number;
  total_escrows: number; active_escrows: number; disputed_escrows: number;
  completed_escrows: number; total_volume_kobo: number; volume_this_month_kobo: number;
  total_revenue_kobo: number; revenue_this_month_kobo: number; revenue_this_week_kobo: number;
  total_disputes: number; open_disputes: number; resolved_disputes: number;
  avg_resolution_days: number;
  pending_withdrawals: number; pending_withdrawal_volume_kobo: number;
  daily_volume_chart: DailyMetric[];
  daily_revenue_chart: DailyMetric[];
}

export interface DailyMetric { date: string; amount_kobo: number; count: number }

export interface AdminUserDetail extends User {
  kyc_documents: KYCDocument[];
  wallets: { buyer_balance_kobo: number; seller_balance_kobo: number };
  recent_escrows: Escrow[];
  recent_audit_logs: AuditLog[];
  escrow_count: number;
}

export interface KYCSubmissionAdmin {
  id: string; user_id: string; user_name: string; user_email: string;
  document_type: string; document_number: string;
  document_url: string; selfie_url: string;
  status: 'pending' | 'approved' | 'rejected';
  rejection_reason?: string;
  verified_by?: string; verified_at?: string;
  created_at: string;
  days_waiting: number;
}

export interface AdminEscrowDetail extends Escrow {
  payments: EscrowPayment[];
  audit_logs: AuditLog[];
}

export interface RevenueReport {
  total_revenue_kobo: number; total_volume_kobo: number;
  transaction_count: number; average_transaction_kobo: number;
  daily_breakdown: DailyMetric[];
}

export interface AdminDisputeItem extends Dispute {
  days_open: number;
  escrow_title: string; escrow_amount_kobo: number;
  buyer_name: string; seller_name: string;
}

export interface AdminFilters {
  page: number; per_page: number;
  search?: string; status?: string; date_from?: string; date_to?: string;
}
```

#### [NEW] `vescrow_admin/src/lib/api/client.ts`

HTTP client identical in pattern to `vescrow_frontend`:
- Fetch wrapper with base path `/api/v1`
- `Authorization: Bearer <token>` header injection
- Token stored in module-level variable (set after login)
- `401` auto-redirects to `/login`
- Standard error class `AdminApiError`

#### [NEW] `vescrow_admin/src/lib/api/admin.ts`

All admin API calls:
```typescript
// Auth
export const adminLogin = (email: string, password: string) => ...

// Dashboard
export const getDashboard = (): Promise<AdminDashboardMetrics> => ...

// Users
export const getUsers = (filters: AdminFilters & { role?: string; kyc_status?: string }) => ...
export const getUser = (id: string): Promise<AdminUserDetail> => ...
export const suspendUser = (id: string, reason: string) => ...
export const unsuspendUser = (id: string) => ...
export const banUser = (id: string, reason: string) => ...

// KYC
export const getKYCQueue = (status?: string): Promise<KYCSubmissionAdmin[]> => ...
export const approveKYC = (id: string) => ...
export const rejectKYC = (id: string, reason: string) => ...

// Escrows
export const getEscrows = (filters: AdminFilters & { status?: string }) => ...
export const getEscrow = (id: string): Promise<AdminEscrowDetail> => ...

// Disputes
export const getDisputes = (filters: AdminFilters & { status?: string }) => ...
export const resolveDispute = (id: string, dto: ResolveDisputeDTO) => ...

// Withdrawals
export const getWithdrawals = (filters: AdminFilters & { status?: string }) => ...

// Revenue
export const getRevenue = (period: string, from?: string, to?: string): Promise<RevenueReport> => ...

// Audit
export const getAuditLogs = (page: number, perPage?: number) => ...

// Settings
export const getSettings = (): Promise<Record<string, string>> => ...
export const updateSettings = (settings: Record<string, string>) => ...
```

**Validation Criteria**:
- [ ] All API functions compile without TypeScript errors
- [ ] API client correctly sets Authorization header
- [ ] 401 response triggers redirect to `/login`

---

### Task 2.4 — Zustand Stores

#### [NEW] `vescrow_admin/src/stores/auth-store.ts`

Admin auth store (pattern identical to `vescrow_frontend` version):
- `user`, `isAuthenticated`, `isLoading` state
- `login(email, password)` — calls API, validates role is `admin|super_admin`, stores user
- `logout()` — calls API, clears state, redirects to `/login`
- `restoreSession()` — calls `/auth/refresh`, hydrates profile
- Persisted to `sessionStorage` under key `vescrow_admin_session`

#### [NEW] `vescrow_admin/src/stores/admin-store.ts`

Single comprehensive store for all admin data:
```typescript
// State slices
metrics: AdminDashboardMetrics | null
users: AdminUserDetail[] ; userPagination: PaginationMeta | null
selectedUser: AdminUserDetail | null
escrows: Escrow[]; escrowPagination: PaginationMeta | null
selectedEscrow: AdminEscrowDetail | null
kycQueue: KYCSubmissionAdmin[]; kycFilter: 'all' | 'pending' | 'approved' | 'rejected'
disputes: AdminDisputeItem[]; disputePagination: PaginationMeta | null
withdrawals: Withdrawal[]; withdrawalPagination: PaginationMeta | null
revenueReport: RevenueReport | null; revenuePeriod: '7d' | '30d' | '90d' | '1y'
settings: Record<string, string>
auditLogs: AuditLog[]; auditPagination: PaginationMeta | null
isLoading: boolean; error: string | null

// Actions (all call API functions from admin.ts)
fetchMetrics, fetchUsers, fetchUser, suspendUser, unsuspendUser, banUser,
fetchKYCQueue, approveKYC, rejectKYC,
fetchEscrows, fetchEscrow,
fetchDisputes, resolveDispute,
fetchWithdrawals,
fetchRevenue,
fetchAuditLogs,
fetchSettings, updateSettings
```

**Validation Criteria**:
- [ ] Auth store persists login state across page refresh
- [ ] Admin store does NOT persist (always fresh from API)
- [ ] Login with `user` role throws error: "You do not have admin access"

---

### Task 2.5 — Route Guard & Admin Login

#### [NEW] `vescrow_admin/src/middleware.ts`

Protect all `/(portal)/*` routes:
```typescript
export const config = { matcher: ['/(portal)/:path*'] }

// If no valid session → redirect to /login
// If session but role = 'user' → redirect to /login with ?error=unauthorized
// If role = 'admin' | 'super_admin' → allow
```

#### [NEW] `vescrow_admin/src/app/(auth)/login/page.tsx`

Dedicated admin login page — premium dark design:
- Vescrow logo + "Admin Portal" label
- Email + Password fields (same field styles as `vescrow_frontend`)
- Calls `/api/v1/auth/login`
- Validates `user.role === 'admin' || 'super_admin'` — shows error if not
- On success: sets auth store, redirects to `/dashboard`
- Optional: TOTP input field shown if `user.is_2fa_enabled === true` after credential check
- "Back to main site" link → `NEXT_PUBLIC_USER_APP_URL`

**Design note**: Dark slate background `#020617`, purple accent `#7C3AED`, centered card with glass effect, subtle purple glow behind card.

**Validation Criteria**:
- [ ] Login with admin credentials → redirects to `/dashboard`
- [ ] Login with user credentials → shows "Admin access required" error
- [ ] Form validates email + password not empty
- [ ] Page not accessible if already logged in (redirect to `/dashboard`)

---

### Task 2.6 — Admin Layout Shell

#### [NEW] `vescrow_admin/src/app/(portal)/layout.tsx`

Server-side session check + renders `<AdminSidebar>` + `<AdminTopbar>` + `<main>`.

#### [NEW] `vescrow_admin/src/components/layout/AdminSidebar.tsx`

```
┌─────────────────────┐
│  🔒 Vescrow  [ADMIN]│
├─────────────────────┤
│  📊 Dashboard       │
│  👥 Users           │
│  🗂 KYC Review  [3] │← pending badge
│  📋 Escrows         │
│  ⚖️  Disputes   [2] │← open badge
│  💸 Withdrawals     │
│  📈 Reports         │
│  ─────────────────  │
│  📝 Audit Logs*     │← super_admin only
│  ⚙️  Settings*      │← super_admin only
├─────────────────────┤
│  [Avatar] Name      │
│  [Role badge]       │
│  [Logout]           │
└─────────────────────┘
```

- Active route: left border `var(--admin-accent)` + background highlight
- Badges refresh from `metrics.kyc_pending_count` and `metrics.open_disputes` on mount
- Collapsible on mobile → slide-out drawer

#### [NEW] `vescrow_admin/src/components/layout/AdminTopbar.tsx`

- Dynamic breadcrumb / page title
- Search input (searches escrow IDs, user emails globally — future enhancement)
- Environment pill (dev/staging/prod) based on `NODE_ENV`
- Admin user avatar + name + role badge + dropdown (profile link, logout)

**Validation Criteria**:
- [ ] Sidebar shows correct nav items per role (audit/settings hidden for `admin`)
- [ ] Active route is highlighted
- [ ] KYC and dispute badges show real counts from dashboard metrics
- [ ] Mobile sidebar works as drawer
- [ ] Topbar shows correct page title for each route
- [ ] Logout calls auth store, clears token, redirects to `/login`

---

## Phase 3: Analytics & Metrics Dashboard

**Goal**: Build the main admin dashboard with live KPIs and charts.
**Estimated effort**: 2–3 days

### Task 3.1 — Dashboard Page

#### [NEW] `vescrow_admin/src/app/(portal)/dashboard/page.tsx`

**Section 1 — Revenue KPI Row**

4 stat cards using a new `<AdminStatCard>` component:
| Card | Value | Trend |
|------|-------|-------|
| Total Revenue | `total_revenue_kobo` → ₦ | — |
| This Month Revenue | `revenue_this_month_kobo` | % vs last month (if calculable) |
| Total Volume | `total_volume_kobo` | — |
| This Month Volume | `volume_this_month_kobo` | — |

**Section 2 — Operations KPI Row**

| Card | Value | Alert |
|------|-------|-------|
| Total Users | `total_users` + "+N today" badge | — |
| Active Escrows | `active_escrows` | — |
| Open Disputes | `open_disputes` | Red if > 0 |
| KYC Pending | `kyc_pending_count` | Amber if > 0 |

**Section 3 — Charts**

Two-column chart grid using `recharts`:
- **Left**: `LineChart` — 30-day daily volume (from `daily_volume_chart`). X-axis: dates, Y-axis: ₦ amount
- **Right**: `BarChart` — 30-day daily revenue (from `daily_revenue_chart`). Shows platform fee collected per day

Both charts: dark background, purple/indigo gridlines, custom tooltip showing ₦ value + count.

**Section 4 — Recent Activity (3 columns)**

Column 1 — Pending KYC (last 5):
- User name, document type, submitted N days ago
- [Approve] [Reject] quick action buttons → open inline modal

Column 2 — Open Disputes (oldest first, last 5):
- Escrow title, amount, days open, SLA badge
- [Review] button → navigates to `/disputes/:id`

Column 3 — Recent Escrows (last 5):
- Title, buyer name, amount, status badge
- Click navigates to `/escrows/:id`

**Section 5 — Alerts Panel** (full width, below columns)

Dynamically computed alerts shown as dismissible banner cards:
- 🚨 Disputes open > 7 days (SLA breach) — red
- ⚠️ Pending withdrawals older than 24h — amber
- ℹ️ High-value (>₦500k) disputed escrows — blue

**Auto-refresh**: Metrics fetched on mount, then `setInterval` every 5 minutes.

**Validation Criteria**:
- [ ] Dashboard loads with single `fetchMetrics()` call
- [ ] All KPI values formatted correctly (kobo → ₦ with commas)
- [ ] Charts render with 30 data points each
- [ ] Recent KYC quick-approve works → updates KYC queue
- [ ] Alerts show correct items
- [ ] Auto-refresh every 5 min (no full page reload)
- [ ] Responsive at 1280px and 1440px

---

### Task 3.2 — Revenue Reports Page

#### [NEW] `vescrow_admin/src/app/(portal)/reports/page.tsx`

**Filter Row**: Period tabs (7d / 30d / 90d / 1y) + Custom range date pickers + Export CSV button

**Metric Cards**: Total Revenue, Total Platform Fees, Total Volume, Transaction Count, Avg Transaction Value, Revenue vs Prior Period (% badge)

**Charts**:
- Daily revenue bar chart (recharts `BarChart`)
- Cumulative volume area chart (recharts `AreaChart`)

**CSV Export**: Client-side CSV generation from `daily_breakdown` array.
Columns: `Date, Transactions, Volume (NGN), Revenue (NGN)`

**Validation Criteria**:
- [ ] Period tabs update all metrics + charts
- [ ] Custom date range pickers work
- [ ] % comparison shows correct direction (↑ green / ↓ red)
- [ ] CSV download generates correct file with naira values (kobo ÷ 100)
- [ ] Empty state when no data in range

---

## Phase 4: User & Escrow Management

**Goal**: Complete user management and escrow oversight pages.
**Estimated effort**: 4 days

### Task 4.1 — Users List Page

#### [NEW] `vescrow_admin/src/app/(portal)/users/page.tsx`

**Filter Bar**: Search input (debounced 300ms), Status dropdown, KYC Status dropdown, Role dropdown, Sort selector

**Data Table** (`<AdminDataTable>` component):
| Column | Content |
|--------|---------|
| User | Avatar initials + Name + Email |
| Phone | `****` masked last 4 |
| Role | `admin` / `super_admin` / `user` badge |
| KYC | KYC status badge |
| Status | `active` / `suspended` / `banned` badge |
| Joined | e.g. "3 days ago" |
| Escrows | Count |
| Actions | View · Suspend/Unsuspend · Ban (super_admin) |

**Pagination**: 20 per page, page controls at bottom

**Row Actions**:
- Suspend → `<AdminConfirmModal>` with required reason text input (min 20 chars)
- Ban → `<AdminConfirmModal>` with extra "type email to confirm" field (super_admin only)
- Unsuspend → single-click confirm

**Validation Criteria**:
- [ ] Search debounced correctly
- [ ] Filter params sent to API correctly
- [ ] Suspend modal blocks submit until reason ≥ 20 chars
- [ ] Ban modal blocks submit until typed email matches `user.email`
- [ ] After action: row status badge updates immediately (optimistic)
- [ ] Pagination changes page correctly

---

### Task 4.2 — User Detail Page

#### [NEW] `vescrow_admin/src/app/(portal)/users/[id]/page.tsx`

**Header**: Avatar + Name + Email + Status badge + Role badge + account action buttons

**Tabbed Content** (5 tabs):

**Tab 1 — Profile**
Full user info. Action buttons context-aware:
- Active user: `[Suspend]` `[Ban (super_admin)]`
- Suspended user: `[Unsuspend]` `[Ban (super_admin)]`
- Banned user: read-only

**Tab 2 — KYC Documents**
List of all submitted documents. For each:
- Document type, submission date, status badge
- For `pending` docs: `[Approve]` `[Reject with reason]` buttons
- `[View Document]` and `[View Selfie]` links (open in new tab)

**Tab 3 — Escrows**
Table: escrow title, role (Buyer/Seller badge), status, amount, partner name, created date. Each row links to `/escrows/:id`.

**Tab 4 — Wallet**
- Buyer wallet balance card + Seller wallet balance card
- Recent 10 ledger transactions table (entry_type badge, amount ₦, description, date)

**Tab 5 — Audit Trail**
Last 20 audit events on this user. Shows actor (admin name or "System"), action, timestamp. Read-only.

**Validation Criteria**:
- [ ] Tabs render correct content
- [ ] KYC approve/reject immediately updates document status in tab
- [ ] Suspend reflects status in Profile tab header immediately
- [ ] Ban confirmation requires email match
- [ ] Escrow links navigate to correct escrow detail
- [ ] Wallet balances shown in ₦ (from kobo)

---

### Task 4.3 — KYC Review Queue

#### [NEW] `vescrow_admin/src/app/(portal)/kyc/page.tsx`

**Layout**: Left panel (30%) + Right panel (70%) — two-column split on desktop, stacked on tablet

**Left — Queue List**
Filter tabs: All / Pending / Approved / Rejected
List items sorted: pending oldest first (SLA priority order)
Each item shows:
- User name + email
- Document type label
- `N days ago` submission time
- SLA urgency badge: green (<3d), amber (3–6d), red (7d+ = breach)
Click → loads right panel

**Right — Review Panel**
- User info header (name, email, current KYC tier)
- Document details: type, document number (decrypted from backend)
- Side-by-side image viewer: Document image | Selfie image (with zoom on click)
- `[✅ Approve]` button (green, full-width) — single confirm step
- `[❌ Reject]` button (red) → opens inline dropdown:
  - Preset reasons: "Document blurry", "Document expired", "Name mismatch", "Selfie mismatch", "Suspected fraud", "Other"
  - Free-text field for custom reason
  - `[Confirm Rejection]` submit button
- After approve/reject: item removed from queue, next item auto-selected

**Pending counter badge** in sidebar nav updates immediately after each action.

**Validation Criteria**:
- [ ] Queue sorted oldest pending first
- [ ] Document images load from backend URLs (cross-origin if storage is S3)
- [ ] Approve removes item from queue immediately (optimistic)
- [ ] Reject requires non-empty reason
- [ ] After action, next queue item auto-selected
- [ ] SLA badge colors correct: green/amber/red
- [ ] Sidebar badge updates after each action

---

### Task 4.4 — Escrow List Page

#### [NEW] `vescrow_admin/src/app/(portal)/escrows/page.tsx`

**Filter Bar**: Search, Status multi-select, Amount range (₦, converted to kobo for API), Date range

**Table**:
| Column | Content |
|--------|---------|
| ID | 8-char truncated UUID + copy icon |
| Title | Truncated (max 30 chars) |
| Buyer | Name + email |
| Seller | Name + email or `—` if pending |
| Amount | ₦ formatted |
| Status | Status badge (all 14 statuses) |
| Created | Relative date |
| Actions | `[View]` → detail page |

**Validation Criteria**:
- [ ] Amount range input accepts ₦ values and converts to kobo for API
- [ ] Status multi-select sends comma-delimited values to API
- [ ] UUID copy button works
- [ ] Pagination functional (20 per page)

---

### Task 4.5 — Escrow Detail Page (Admin)

#### [NEW] `vescrow_admin/src/app/(portal)/escrows/[id]/page.tsx`

**Header**: Escrow title, amount, status badge, created date, funded date

**Collapsible Sections**:

1. **Parties** — Buyer card (name, email, KYC status, link to `/users/:id`) + Seller card (same or "Pending invitation")
2. **Timeline** — Visual step-by-step state machine: each status as a node with timestamp and actor (buyer/seller/system)
3. **Milestones** — Table if milestones exist
4. **Payment Record** — Paystack reference, amount, status, paid_at
5. **Ledger Entries** — All debit/credit movements on this escrow (from admin escrow detail endpoint)
6. **Messages** — Read-only chat thread between buyer and seller
7. **Dispute** — If `status = 'disputed'`: shows dispute summary + `[Go to Dispute →]` button
8. **Audit Trail** — Last 20 admin/system actions on this escrow

**Validation Criteria**:
- [ ] All sections load from single API call (`getEscrow(id)`)
- [ ] Timeline nodes show correct timestamps
- [ ] Ledger entries show debit/credit badge and ₦ amount
- [ ] "Go to Dispute" button links to correct dispute ID
- [ ] Audit trail is read-only

---

## Phase 5: Dispute Management

**Goal**: Full arbitration workflow — queue, review, evidence viewer, resolution.
**Estimated effort**: 3 days

### Task 5.1 — Dispute Queue Page

#### [NEW] `vescrow_admin/src/app/(portal)/disputes/page.tsx`

**Filter Tabs**: All / Open / Seller Responded / Under Review / Resolved (with count badge on each)

**Table**:
| Column | Content |
|--------|---------|
| ID | 8-char UUID |
| Escrow | Title + ₦ amount |
| Buyer | Name |
| Seller | Name |
| Reason | Truncated (40 chars) |
| Status | Status badge |
| Opened | Date |
| Age | `N days` |
| SLA | Urgency badge (green/amber/red) |

**Default Sort**: Oldest open disputes first (SLA priority)

**SLA Logic**: Green (<3 days), Amber (3–6 days), Red (≥7 days = breach)

**Validation Criteria**:
- [ ] SLA badge colors based on `days_open`
- [ ] Filter tabs each show correct count in badge
- [ ] Default sort shows oldest open first
- [ ] Row click → navigate to `/disputes/:id`

---

### Task 5.2 — Dispute Detail & Resolution Page

#### [NEW] `vescrow_admin/src/app/(portal)/disputes/[id]/page.tsx`

**3-Column Layout** (desktop), stacked (tablet/mobile):

**Column 1 — Case Summary** (25% width)
- Status badge + SLA indicator
- Escrow info card (title, amount, link to `/escrows/:id`)
- Dispute details (category, reason, description)
- Opened by (buyer name) + opened date
- Days open

**Column 2 — Evidence & Thread** (45% width)

*Evidence Gallery*:
- Grid of uploaded evidence files
- Images → click opens lightbox (`<ImageLightbox>` component)
- PDFs/documents → download link with file icon
- Each item: uploader badge (Buyer/Seller), filename, upload date

*Discussion Thread*:
- Chronological comments, oldest first
- Buyer comments: left-aligned, blue bubble
- Seller comments: right-aligned, gray bubble
- Admin comments: center, purple bubble with "Admin" badge
- Admin comment input box at bottom (visible only if dispute is NOT resolved)
  - Textarea + `[Post Comment]` button
  - Calls `POST /disputes/:id/comment`

**Column 3 — Resolution Panel** (30% width)

*If dispute is Open or Seller Responded*:

Escrow amount summary:
```
Total escrow amount: ₦XX,XXX.XX
└── Buyer gets: ₦ [input]
└── Seller gets: ₦ [input]
    Sum must equal total ↑
```

Resolution type selector (radio cards):
- 🔴 **Full Refund** — all funds returned to buyer
- 🟢 **Full Release** — all funds released to seller
- 🟡 **Split** — custom split (activates both amount inputs)

Resolution notes textarea (required, min 50 chars)

`[Resolve Dispute]` button → opens `<AdminConfirmModal>`:
- Summary: "You are about to [resolution] ₦[amount] for escrow [title]"
- Type `RESOLVE` to confirm
- Submit → calls `PUT /disputes/:id/resolve`

*If dispute is Resolved* (read-only):
- Outcome card: resolution type badge, amounts, resolution notes
- Resolved by: admin name + timestamp

**Validation Criteria**:
- [ ] Evidence lightbox opens correctly for images
- [ ] PDFs open in new tab
- [ ] Split: buyer + seller amounts must equal `escrow.amount_kobo` exactly (enforced in real time)
- [ ] Resolution notes enforces min 50 chars
- [ ] "RESOLVE" confirmation is case-sensitive exact match
- [ ] After resolve: status badge updates without full reload
- [ ] Admin comment posts and appears in thread immediately (optimistic)
- [ ] Resolved disputes show read-only panel with no resolution form

---

### Task 5.3 — Withdrawals Monitoring Page

#### [NEW] `vescrow_admin/src/app/(portal)/withdrawals/page.tsx`

Read-only monitoring dashboard (Paystack handles actual transfers):

**Filters**: Status (pending / processing / completed / failed / reversed), Date range

**Table**:
| Column | Content |
|--------|---------|
| User | Name + email |
| Amount | ₦ |
| Bank | Bank name + `****XXXX` account |
| Status | Status badge |
| Requested | Date |
| Completed | Date or `—` |
| Failure | Tooltip icon if failed |

**Validation Criteria**:
- [ ] Status filter works
- [ ] Failed entries show failure reason in tooltip on hover
- [ ] All amounts in ₦ (kobo ÷ 100)

---

### Task 5.4 — Audit Logs Page

#### [NEW] `vescrow_admin/src/app/(portal)/audit-logs/page.tsx`

**Access guard**: If `user.role !== 'super_admin'` → show `<AccessDenied>` component (locked icon + message + link back to dashboard)

**Filters**: Actor type (user/admin/system), Action search, Resource type, Date range

**Table** (50 per page):
| Column | Content |
|--------|---------|
| Timestamp | `YYYY-MM-DD HH:mm:ss` |
| Actor | Name + role badge or "System" |
| Action | Monospace code: `dispute.resolved`, `kyc.approved` |
| Resource | Type + linkable short ID |
| IP | Masked: `192.168.x.x` |
| Changes | Expandable `[+]` toggle showing old → new state diff |

**Validation Criteria**:
- [ ] `admin` role sees `<AccessDenied>`, NOT the logs
- [ ] Pagination 50 per page
- [ ] State diff expandable for complex actions
- [ ] Resource IDs link to correct detail pages (escrow, user, dispute)
- [ ] Date filter works correctly

---

### Task 5.5 — Platform Settings Page

#### [NEW] `vescrow_admin/src/app/(portal)/settings/page.tsx`

**Access guard**: `super_admin` only. Admin role sees locked read-only view.

**Settings Sections**:

**💰 Fee Configuration**
- Platform fee (%) — `platform_fee_percent`
- Incoming fee (%) — `incoming_fee_percent`
- Outgoing fee (%) — `outgoing_fee_percent`

**🏦 Withdrawal Limits**
- Daily limit (₦) — `daily_withdrawal_limit_kobo` (display in ₦, store in kobo)
- Per-transaction limit (₦) — `per_tx_withdrawal_limit_kobo`

**🔐 KYC Transaction Limits** (CBN-aligned)
- Tier 1 max (₦) — `kyc_tier1_limit_kobo`
- Tier 2 max (₦) — `kyc_tier2_limit_kobo`

**📅 Escrow Defaults**
- Default inspection period (days) — `default_inspection_days`
- Maximum inspection period (days) — `max_inspection_days`

**UX**:
- Unsaved fields highlighted with amber left border
- Single `[Save Changes]` button (bottom of page)
- On save: confirmation dialog → calls `PUT /admin/settings`
- Success: toast + settings reloaded from API
- Audit log entry created automatically

**Validation Criteria**:
- [ ] `admin` role sees read-only inputs (disabled, locked icon)
- [ ] `super_admin` can edit all fields
- [ ] Fee inputs: 0–100 range, max 2 decimal places
- [ ] Limit inputs: positive integers only (₦ values)
- [ ] Unsaved change indicator appears correctly on any edit
- [ ] Save creates audit log (verifiable in audit logs page)

---

## Phase 6: Polish, Components & QA

**Goal**: Shared component library, loading/error states, responsiveness, final testing.
**Estimated effort**: 2 days

### Task 6.1 — Admin UI Component Library

#### [NEW] `vescrow_admin/src/components/ui/`

Mirror key components from `vescrow_frontend` (adapted for admin):
- `Button` — primary (purple), secondary, danger, ghost variants
- `Card` — elevated, glass variants
- `Badge` — all escrow statuses + KYC + dispute + user statuses
- `Input` — text, number, search, with error state
- `Modal` — accessible dialog with backdrop
- `Select` — custom dropdown
- `Tabs` — horizontal with animated underline

#### [NEW] `vescrow_admin/src/components/admin/`

Admin-specific components:
- `AdminStatCard` — KPI card with value, label, icon, optional trend badge
- `AdminDataTable` — Generic paginated sortable table with row actions
- `AdminFilterBar` — Reusable filter row (search + status + date range)
- `AdminConfirmModal` — Destructive action modal with optional type-to-confirm field
- `AdminSLABadge` — Color-coded urgency badge based on days_open
- `AdminEmptyState` — Empty queue / no results illustration + message
- `AccessDenied` — Super-admin-only page gate component
- `ImageLightbox` — Full-screen image viewer for evidence
- `StatusBadge` — Unified status badge for escrow/dispute/withdrawal/user statuses

### Task 6.2 — Loading & Error States

For every data-fetching page:
- Skeleton screens while loading (matching table/card layout)
- Error boundary component with retry button
- Empty state when filter returns 0 results
- Network error toast with retry action

### Task 6.3 — Responsive Breakpoints

Admin is primarily desktop-focused but must work on tablet:
- **1440px+**: Full 3-column layouts
- **1280px**: Standard desktop, all features
- **1024px**: Sidebar collapses to icon-only, content adjusts
- **< 1024px**: Sidebar becomes hamburger drawer, tables scroll horizontally

### Task 6.4 — Full Integration Testing

End-to-end smoke test checklist:
- [ ] `npm run dev` on `vescrow_admin` starts on port 3002
- [ ] Admin login with test admin credentials → dashboard loads with real data
- [ ] Dashboard KPIs match backend DB (verify counts match `psql` queries)
- [ ] KYC queue: approve → user KYC status updated → confirmed via backend
- [ ] Dispute resolve → escrow status updated → ledger entries created
- [ ] User suspend → user gets 401/403 on next API call from user app
- [ ] Settings update → `platform_settings` table updated in DB
- [ ] Audit logs: every admin action visible with correct actor
- [ ] `admin` role: audit-logs and settings pages show Access Denied
- [ ] Logout → session cleared → redirect to `/login` → back button blocked
- [ ] `npm run build` completes without errors
- [ ] `npx tsc --noEmit` passes

---

## Summary

| Phase | Scope | Backend | Frontend | Effort |
|-------|-------|---------|----------|--------|
| **1. Backend API** | 13 missing endpoints + tests | ✅ Go | — | 3–4 days |
| **2. App Setup & Auth** | Scaffold, types, stores, login, shell | — | ✅ Next.js | 2–3 days |
| **3. Analytics Dashboard** | KPIs, charts, revenue reports | — | ✅ Next.js | 2–3 days |
| **4. User & Escrow Mgmt** | Users, KYC queue, escrows | — | ✅ Next.js | 3–4 days |
| **5. Dispute Management** | Dispute queue, resolution, withdrawals, audit, settings | — | ✅ Next.js | 3 days |
| **6. Polish & QA** | Components, skeletons, responsive, E2E | — | ✅ Next.js | 2 days |
| **Total** | **29 tasks** | **~4 days** | **~13 days** | **~17 days** |

---

## File Change Manifest

### vescrow_backend (Go)

| File | Change |
|------|--------|
| `internal/admin/service.go` | [NEW] Business logic |
| `internal/admin/repository.go` | [NEW] DB queries |
| `internal/admin/handler.go` | [MODIFY] Add 13 handlers |
| `internal/admin/routes.go` | [MODIFY] Register all endpoints |
| `internal/admin/admin_test.go` | [MODIFY] Extend tests |
| `internal/middleware/cors.go` | [MODIFY] Allow port 3002 |

### vescrow_admin (New Next.js App)

| File | Change |
|------|--------|
| `package.json` | [NEW] |
| `next.config.ts` | [NEW] with API proxy |
| `.env.local` | [NEW] |
| `src/app/globals.css` | [NEW] Admin design tokens |
| `src/lib/types.ts` | [NEW] |
| `src/lib/api/client.ts` | [NEW] |
| `src/lib/api/admin.ts` | [NEW] |
| `src/lib/constants.ts` | [NEW] |
| `src/lib/utils.ts` | [NEW] |
| `src/middleware.ts` | [NEW] Route guard |
| `src/stores/auth-store.ts` | [NEW] |
| `src/stores/admin-store.ts` | [NEW] |
| `src/app/layout.tsx` | [NEW] |
| `src/app/(auth)/login/page.tsx` | [NEW] |
| `src/app/(portal)/layout.tsx` | [NEW] Admin shell |
| `src/app/(portal)/dashboard/page.tsx` | [NEW] |
| `src/app/(portal)/users/page.tsx` | [NEW] |
| `src/app/(portal)/users/[id]/page.tsx` | [NEW] |
| `src/app/(portal)/kyc/page.tsx` | [NEW] |
| `src/app/(portal)/escrows/page.tsx` | [NEW] |
| `src/app/(portal)/escrows/[id]/page.tsx` | [NEW] |
| `src/app/(portal)/disputes/page.tsx` | [NEW] |
| `src/app/(portal)/disputes/[id]/page.tsx` | [NEW] |
| `src/app/(portal)/withdrawals/page.tsx` | [NEW] |
| `src/app/(portal)/reports/page.tsx` | [NEW] |
| `src/app/(portal)/audit-logs/page.tsx` | [NEW] |
| `src/app/(portal)/settings/page.tsx` | [NEW] |
| `src/components/layout/AdminSidebar.tsx` | [NEW] |
| `src/components/layout/AdminTopbar.tsx` | [NEW] |
| `src/components/ui/` | [NEW] 8 primitive components |
| `src/components/admin/` | [NEW] 9 admin-specific components |
