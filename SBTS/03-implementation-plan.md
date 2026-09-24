# ElectionsSentinel (ES360) — Implementation Plan

**Document Version:** 1.0  
**Date:** 27 August 2026  
**Classification:** Internal — Engineering Reference  

---

## Table of Contents

1. [Phase Summary & Timeline](#1-phase-summary--timeline)
2. [Phase 1 — Production Foundation & Testing Infrastructure](#2-phase-1--production-foundation--testing-infrastructure)
3. [Phase 2 — Data Integrity, PVT Completeness & Evidence Compliance](#3-phase-2--data-integrity-pvt-completeness--evidence-compliance)
4. [Phase 3 — Real-Time Operations, Notifications & Performance](#4-phase-3--real-time-operations-notifications--performance)
5. [Phase 4 — Offline Resilience, PWA & Field Readiness](#5-phase-4--offline-resilience-pwa--field-readiness)
6. [Phase 5 — Intelligence Maturity, Analytics & Reporting](#6-phase-5--intelligence-maturity-analytics--reporting)
7. [Phase 6 — Infrastructure, Deployment & Operational Readiness](#7-phase-6--infrastructure-deployment--operational-readiness)
8. [Phase 7 — Security Hardening & Compliance Audit](#8-phase-7--security-hardening--compliance-audit)
9. [Cross-Phase Dependency Map](#9-cross-phase-dependency-map)

---

## 1. Phase Summary & Timeline

| Phase | Name | Duration | Dependencies | Key Deliverables |
|-------|------|----------|--------------|-----------------|
| **Phase 1** | Production Foundation & Testing | 3 weeks | None | Test suite, migrations, API docs, health checks, structured logging |
| **Phase 2** | Data Integrity & PVT Completeness | 2 weeks | Phase 1 | Per-party scores, over-voting alerts, §84 certificates, GPS evidence |
| **Phase 3** | Real-Time Operations & Notifications | 2 weeks | Phase 1 | WebSocket gateway, notification system, SLA escalation, live dashboard |
| **Phase 4** | Offline Resilience & Field Readiness | 2 weeks | Phase 1, 3 | Service worker, bulk import, agent mobile UX |
| **Phase 5** | Intelligence, Analytics & Reporting | 2 weeks | Phase 1, 2 | Collation roll-ups, IReV comparison, constituency analytics, narrative clustering |
| **Phase 6** | Infrastructure & Deployment | 2 weeks | Phase 1–5 | Docker hardening, CI/CD, monitoring, backup, blue-green deploys |
| **Phase 7** | Security Hardening & Compliance | 1 week | Phase 1–6 | Penetration testing, CORS hardening, MFA encryption, security audit |

**Total Estimated Duration:** 14 weeks (3.5 months)

---

## 2. Phase 1 — Production Foundation & Testing Infrastructure

> **Goal:** Establish the testing, documentation, and operational foundation required for all subsequent phases. No production deployment should occur without Phase 1 complete.

---

### TASK-0101: Set Up Testing Framework & Configuration

**Priority:** P0 | **Effort:** 2 days | **Assignee:** Backend Engineer

**Description:**  
Install and configure Jest for the NestJS backend with TypeORM test utilities, test database configuration, and coverage reporting.

**Deliverables:**
- Install `@nestjs/testing`, `jest`, `ts-jest`, `supertest`
- Create `jest.config.ts` with module path aliases matching `tsconfig.json`
- Create test database configuration (`test.env`) pointing to a separate `sentinel360_test` DB
- Configure `npm run test`, `npm run test:cov`, `npm run test:e2e` scripts in `package.json`
- Add `beforeAll`/`afterAll` helpers for test DB setup/teardown

**Validation Criteria:**
- [ ] `npm run test` executes successfully with 0 tests (framework loads without error)
- [ ] `npm run test:cov` generates HTML coverage report in `coverage/`
- [ ] Test DB is created and destroyed cleanly during test lifecycle
- [ ] CI-compatible: tests run headlessly without manual intervention

---

### TASK-0102: Write Unit Tests for Auth Module

**Priority:** P0 | **Effort:** 3 days | **Assignee:** Backend Engineer

**Description:**  
Comprehensive unit tests for `AuthService` covering registration, login, MFA enrollment, MFA verification, JWT generation, and refresh token rotation.

**Deliverables:**
- `src/auth/__tests__/auth.service.spec.ts`
- Test cases:
  - Registration with valid/invalid/duplicate email
  - Login with correct/incorrect password
  - MFA enrollment flow for privileged roles
  - MFA verification with valid/invalid/expired TOTP codes
  - JWT token generation and payload validation
  - Refresh token creation and rotation

**Validation Criteria:**
- [ ] ≥ 90% line coverage for `auth.service.ts`
- [ ] All 12+ test cases pass
- [ ] MFA flow correctly blocks privileged roles without enrollment
- [ ] Refresh token rotation revokes previous token on reuse

---

### TASK-0103: Write Unit Tests for Core Domain Services

**Priority:** P0 | **Effort:** 5 days | **Assignee:** Backend Engineer

**Description:**  
Unit tests for all critical business logic services: Agents (deployment gating), Incidents (SLA computation, dedup), Results (double-entry verification, mathematical integrity), Evidence (hash verification, custody chain), Compliance (rule engine, occurrence generation), Finance (cap enforcement), Comms (silence period), Intelligence (language detection).

**Deliverables:**
- `src/agents/__tests__/agents.service.spec.ts`
- `src/incidents/__tests__/incidents.service.spec.ts`
- `src/incidents/__tests__/sla-policy.spec.ts`
- `src/results/__tests__/results.service.spec.ts`
- `src/evidence/__tests__/evidence.service.spec.ts`
- `src/compliance/__tests__/compliance-rules.service.spec.ts`
- `src/finance/__tests__/finance.service.spec.ts`
- `src/comms/__tests__/comms.service.spec.ts`
- `src/intelligence/__tests__/language-detector.spec.ts`

**Validation Criteria:**
- [ ] ≥ 85% line coverage across all tested services
- [ ] Agent deployment gating: test Certified + equipmentComplete required for status advancement
- [ ] SLA computation: test all 5 states (On track, At risk, Ack breached, Resolve breached, Resolved)
- [ ] Incident dedup: test same-geography + similar-title within 6 hours triggers flag
- [ ] Double-entry: test matching and mismatching verification entries
- [ ] Mathematical integrity: test `valid + rejected > accredited` detection
- [ ] Finance cap: test within-cap, over-cap-blocked, over-cap-with-approval scenarios
- [ ] Silence period: test outside-window, inside-window-blocked, inside-window-override
- [ ] Language detector: test all 5 languages + Mixed + Undetermined cases

---

### TASK-0104: Write Integration Tests for Critical API Flows

**Priority:** P0 | **Effort:** 3 days | **Assignee:** Backend Engineer

**Description:**  
End-to-end integration tests using `supertest` that exercise the full HTTP pipeline (auth → guard → controller → service → database → response) for the most critical flows.

**Deliverables:**
- `test/auth.e2e-spec.ts` — Register → Login → MFA → Access protected endpoint
- `test/results.e2e-spec.ts` — Create result → Double-entry verify → Check reconciliation status
- `test/evidence.e2e-spec.ts` — Upload file → Verify hash → Apply legal hold → Export with hash check
- `test/incidents.e2e-spec.ts` — Create incident → Check SLA → Acknowledge → Resolve
- `test/finance.e2e-spec.ts` — Preview cap → Record entry → Verify cap enforcement

**Validation Criteria:**
- [ ] All E2E tests pass against test database
- [ ] Tests exercise the full NestJS middleware pipeline (Helmet, CORS, Throttle, Validation, JWT, Roles)
- [ ] Tests verify RBAC: unauthorized role receives 403
- [ ] Tests verify geography scoping: out-of-scope write receives 403
- [ ] Tests clean up database state after each suite

---

### TASK-0105: Implement TypeORM Production Migrations

**Priority:** P0 | **Effort:** 2 days | **Assignee:** Backend Engineer

**Description:**  
Replace `synchronize: true` with proper TypeORM migration files for production deployments. Generate the initial migration from the current entity state, test migration up/down, and configure production to use migrations only.

**Deliverables:**
- `src/migrations/` directory with timestamped migration files
- Initial migration capturing all 19 entities and their indexes
- `data-source.ts` updated with migration paths for CLI usage
- `app.module.ts` updated: `synchronize: false` when `NODE_ENV=production`
- `npm run migration:generate`, `npm run migration:run`, `npm run migration:revert` scripts working

**Validation Criteria:**
- [ ] `npm run migration:run` against an empty database creates all tables correctly
- [ ] `npm run migration:revert` cleanly rolls back the last migration
- [ ] Application boots successfully with `synchronize: false` and migrations applied
- [ ] Seed data SQL executes successfully after migration
- [ ] No data loss when migrating from a `synchronize: true` database

---

### TASK-0106: Generate OpenAPI/Swagger Documentation

**Priority:** P1 | **Effort:** 2 days | **Assignee:** Backend Engineer

**Description:**  
Install `@nestjs/swagger` and annotate all controllers and DTOs with Swagger decorators to generate interactive API documentation at `/api/docs`.

**Deliverables:**
- Install `@nestjs/swagger`, `swagger-ui-express`
- Add `SwaggerModule.setup()` in `main.ts`
- Annotate all 15 controllers with `@ApiTags`, `@ApiOperation`, `@ApiResponse`
- Annotate all DTOs with `@ApiProperty` including descriptions and examples
- Configure bearer token authentication in Swagger UI

**Validation Criteria:**
- [ ] `http://localhost:4000/api/docs` renders Swagger UI
- [ ] All 60+ endpoints listed with correct HTTP methods, paths, and tags
- [ ] DTOs show all fields with types, descriptions, and example values
- [ ] Bearer token can be set once and used for all authenticated requests
- [ ] Response schemas accurately reflect actual API responses

---

### TASK-0107: Implement Structured Logging

**Priority:** P1 | **Effort:** 2 days | **Assignee:** Backend Engineer

**Description:**  
Replace the default NestJS console logger with a structured JSON logger (Winston or Pino) that includes correlation IDs, configurable log levels, and production-ready output.

**Deliverables:**
- Install `nestjs-pino` + `pino-pretty` (or Winston equivalent)
- Configure log levels via `LOG_LEVEL` environment variable
- Add correlation ID middleware (generate UUID per request, attach to all log entries)
- Add request/response logging (method, path, status, duration)
- Suppress health check logs to avoid noise
- Configure `pino-pretty` for local dev, raw JSON for production

**Validation Criteria:**
- [ ] All log output is valid JSON in production mode
- [ ] Each log entry includes: `timestamp`, `level`, `correlationId`, `module`, `message`
- [ ] Request logs include: `method`, `path`, `statusCode`, `durationMs`
- [ ] Log level is configurable via environment variable without code change
- [ ] No console.log() calls remain in production code

---

### TASK-0108: Add Health Check Endpoints

**Priority:** P1 | **Effort:** 1 day | **Assignee:** Backend Engineer

**Description:**  
Implement health check and readiness probe endpoints using `@nestjs/terminus` for orchestration systems (Kubernetes, ECS, load balancers).

**Deliverables:**
- Install `@nestjs/terminus`
- `GET /api/health` — liveness probe (always returns 200 if app is running)
- `GET /api/health/ready` — readiness probe (checks DB connection, storage accessibility)
- Health check module with TypeORM health indicator and custom storage health indicator

**Validation Criteria:**
- [ ] `GET /api/health` returns 200 with `{ status: 'ok' }`
- [ ] `GET /api/health/ready` returns 200 when DB is connected and storage is accessible
- [ ] `GET /api/health/ready` returns 503 when DB is down with `{ status: 'error', details: ... }`
- [ ] Health endpoints are excluded from JWT authentication
- [ ] Health endpoints are excluded from rate limiting

---

### TASK-0109: Harden CORS Configuration

**Priority:** P0 | **Effort:** 0.5 days | **Assignee:** Backend Engineer

**Description:**  
Replace the permissive `origin: true` CORS setting with environment-configurable origin whitelist.

**Deliverables:**
- Add `WEB_ORIGIN` environment variable (already exists in `.env.example`)
- Parse comma-separated origins from `WEB_ORIGIN` (support multiple domains for staging + production)
- Set `origin` to the parsed whitelist instead of `true`
- Add `Vary: Origin` header handling

**Validation Criteria:**
- [ ] Requests from whitelisted origins succeed with proper CORS headers
- [ ] Requests from non-whitelisted origins receive no CORS headers (browser blocks)
- [ ] Preflight OPTIONS requests return correct `Access-Control-Allow-*` headers
- [ ] Multiple origins supported via comma-separated `WEB_ORIGIN`

---

### TASK-0110: Implement Geography Scope Filtering on Read Operations

**Priority:** P0 | **Effort:** 2 days | **Assignee:** Backend Engineer

**Description:**  
Currently the geography scope guard only checks request bodies on create operations. Extend it to filter GET list results by the caller's assigned geography subtree.

**Deliverables:**
- Create a `GeographyScopeInterceptor` that filters query results based on the caller's `assignedGeographicUnitId`
- For national-scope roles: no filtering (return all)
- For scoped roles: filter by `geographicUnitId` matching the caller's assigned geography path prefix
- Apply to all list endpoints: agents, incidents, results, evidence, compliance tasks, governance cases

**Validation Criteria:**
- [ ] A State Director assigned to Lagos sees only Lagos-scoped records in all list endpoints
- [ ] A Super Admin sees all records across all states
- [ ] A PU Agent sees only their own PU's records
- [ ] Filter works with the materialized `path` column for subtree matching
- [ ] No performance regression: filter uses indexed columns

---

## 3. Phase 2 — Data Integrity, PVT Completeness & Evidence Compliance

> **Goal:** Complete the parallel vote tabulation feature set and evidence compliance capabilities needed for tribunal-grade evidence.

---

### TASK-0201: Implement Per-Party Candidate Vote Scores

**Priority:** P0 | **Effort:** 3 days | **Assignee:** Full-Stack Engineer

**Description:**  
Create a `ResultCandidateScore` entity that stores per-party vote counts linked to a `ResultForm`, replacing the current aggregate-only result capture.

**Deliverables:**
- New entity: `src/results/result-candidate-score.entity.ts`
  - Fields: `id`, `resultFormId` (FK), `partyName`, `candidateName`, `votes`, `createdAt`
  - Composite index on `(resultFormId, partyName)`
- Update `CreateResultDto` to accept `candidateScores: Array<{ partyName, candidateName, votes }>`
- Update `ResultsService.create()` to insert candidate scores alongside the form
- Update `ResultsService.findAll()` to include candidate scores in response
- Mathematical integrity validation: `SUM(candidate_votes) = valid_votes`
- Update double-entry verification to include per-party scores comparison
- Frontend: update results capture modal to include per-party score entry fields
- Frontend: update results table to show per-party breakdown on expand

**Validation Criteria:**
- [ ] Creating a result with 5 party scores persists all 5 entries linked to the form
- [ ] Sum-of-parties validation catches mismatch: `Σ(party.votes) ≠ validVotes`
- [ ] Double-entry verification compares party scores between operators
- [ ] GET `/api/results/:id` returns embedded `candidateScores` array
- [ ] Frontend modal renders dynamic party score input rows (add/remove)

---

### TASK-0202: Implement Proactive Over-Voting Alert System

**Priority:** P0 | **Effort:** 2 days | **Assignee:** Backend Engineer

**Description:**  
Add real-time over-voting detection that fires when a result form is created or verified, generating an alert entity that references §51(2) of the Electoral Act.

**Deliverables:**
- New entity: `src/results/overvoting-alert.entity.ts`
  - Fields: `id`, `resultFormId` (FK), `electionId`, `geographicUnitId`, `alertType` (over-accreditation / over-registration), `computedTotal`, `threshold`, `statutoryReference`, `status` (Open / Acknowledged / Dismissed), `createdAt`
- Detection logic in `ResultsService`: check `validVotes + rejectedVotes > accreditedVoters` on every create/verify
- Also check: `accreditedVoters > registeredVoters`
- Alert creation includes statutory reference: "Electoral Act 2026 §51(2)"
- `GET /api/results/overvoting-alerts` — list all alerts by election, filterable by status
- Frontend: over-voting badge on affected result row; alert banner in dashboard

**Validation Criteria:**
- [ ] Submitting a result where `valid + rejected > accredited` creates an alert
- [ ] Submitting a result where `accredited > registered` creates a separate alert
- [ ] Alert includes correct §51(2) statutory reference
- [ ] Dashboard summary includes `overvotingAlertCount`
- [ ] Alerts are scoped by election and geography

---

### TASK-0203: Generate Section 84 Digital Evidence Certificate

**Priority:** P1 | **Effort:** 3 days | **Assignee:** Full-Stack Engineer

**Description:**  
Implement a PDF certificate generator that produces Evidence Act §84-compliant digital evidence certificates for export with evidence items.

**Deliverables:**
- Install `pdfmake` or `@react-pdf/renderer` (server-side)
- New service: `src/evidence/certificate.service.ts`
- Certificate content per §84(2) requirements:
  1. **System Identification**: Platform name, version, deployment ID
  2. **Regular Use Attestation**: Statement that the system is used regularly for election monitoring activities
  3. **Data Supply Regularity**: Statement that evidence of this kind is regularly ingested by the system
  4. **Proper Functioning**: System uptime attestation for the relevant period
  5. **Hash Verification**: SHA-256 hash of the evidence item
  6. **Chain of Custody Summary**: All custody events in chronological order
  7. **Signatory**: Name, role, and digital signature of the certifying officer (Legal Team role)
- `POST /api/evidence/:id/certificate` — generates and returns PDF
- Certificate stored as a separate evidence item linked to the original
- Frontend: "Generate §84 Certificate" button in custody drawer

**Validation Criteria:**
- [ ] PDF generates with all 7 required sections populated
- [ ] Certificate includes the correct SHA-256 hash matching the stored hash
- [ ] Only Legal Team and DPO roles can generate certificates
- [ ] Certificate PDF is stored as an evidence item with its own SHA-256 hash
- [ ] Generated certificate is a valid PDF openable in standard viewers

---

### TASK-0204: Add GPS Coordinates to Evidence Capture

**Priority:** P1 | **Effort:** 1 day | **Assignee:** Full-Stack Engineer

**Description:**  
Capture GPS coordinates at evidence upload time using the browser Geolocation API and store them on the evidence item.

**Deliverables:**
- Add `latitude` (FLOAT, nullable) and `longitude` (FLOAT, nullable) columns to `EvidenceItem` entity
- Update `CreateEvidenceDto` to accept optional `latitude` and `longitude`
- Frontend: request geolocation permission on evidence upload; attach coordinates to upload request
- Display coordinates in evidence detail/custody drawer
- Include coordinates in §84 certificate (Task 0203)

**Validation Criteria:**
- [ ] Evidence uploaded with GPS shows coordinates in detail view
- [ ] Evidence uploaded without GPS (permission denied) saves null coordinates without error
- [ ] Coordinates are included in chain-of-custody export
- [ ] GPS format: decimal degrees (e.g., 6.5244° N, 3.3792° E)

---

### TASK-0205: Add `linkedRecordType` to Evidence Items

**Priority:** P1 | **Effort:** 0.5 days | **Assignee:** Backend Engineer

**Description:**  
Currently evidence items have `linkedRecordId` but no `linkedRecordType`, making it ambiguous whether the linked record is an incident, result, compliance task, etc.

**Deliverables:**
- Add `linked_record_type` VARCHAR column to `evidence_items` (nullable, for backward compatibility)
- Update `CreateEvidenceDto` to accept `linkedRecordType` (enum: 'Incident', 'ResultForm', 'ComplianceTask', 'GovernanceCase', 'Other')
- Update evidence queries to filter by record type when specified

**Validation Criteria:**
- [ ] New evidence items save with both `linkedRecordId` and `linkedRecordType`
- [ ] Existing evidence items with null `linkedRecordType` continue to work
- [ ] `GET /api/evidence?linkedRecordType=Incident` filters correctly

---

### TASK-0206: Implement Timezone-Aware Silence Period

**Priority:** P1 | **Effort:** 1 day | **Assignee:** Backend Engineer

**Description:**  
Fix the election silence service to use the election's configured `timezone` field instead of server local time for silence period computation.

**Deliverables:**
- Install `luxon` or use Node.js built-in `Intl.DateTimeFormat` for timezone conversion
- Update `ElectionSilenceService` to convert current time and polling date to the election's timezone
- Silence window: 24 hours before polling date midnight through end of polling day (23:59:59) in election timezone

**Validation Criteria:**
- [ ] Silence period is correctly computed for `Africa/Lagos` timezone
- [ ] Server running in UTC correctly identifies silence period for Lagos-time elections
- [ ] Silence period spans: `(pollingDate - 1 day) 00:00:00` to `pollingDate 23:59:59` in election timezone
- [ ] Edge case: midnight transition handled correctly

---

## 4. Phase 3 — Real-Time Operations, Notifications & Performance

> **Goal:** Enable live situation room operations with real-time data push, automated notifications, and SLA-driven escalation.

---

### TASK-0301: Implement WebSocket Gateway for Live Dashboard

**Priority:** P1 | **Effort:** 4 days | **Assignee:** Backend Engineer

**Description:**  
Add a NestJS WebSocket gateway (Socket.IO or native `ws`) that pushes live metric updates to connected dashboard clients, replacing HTTP polling.

**Deliverables:**
- Install `@nestjs/websockets`, `@nestjs/platform-socket.io` (or `ws`)
- New module: `src/realtime/realtime.module.ts`
- `RealtimeGateway` with:
  - `connection` handler: authenticate via JWT token in handshake
  - `dashboard:summary` event: broadcast when any dashboard-affecting entity changes
  - `incidents:update` event: broadcast on incident create/update
  - `results:update` event: broadcast on result create/verify
  - `agents:heartbeat` event: broadcast on agent sync status change
- Emit events from domain services using NestJS `EventEmitter2` pattern
- Frontend: connect Zustand dashboard store to WebSocket; fallback to HTTP polling if WS unavailable

**Validation Criteria:**
- [ ] WebSocket connection established with valid JWT
- [ ] Connection rejected with invalid/expired JWT
- [ ] Creating an incident broadcasts `incidents:update` to all connected clients within 2 seconds
- [ ] Dashboard store receives push updates without manual refresh
- [ ] Graceful fallback to 30-second HTTP polling when WebSocket unavailable
- [ ] Connection survives page navigation within the SPA

---

### TASK-0302: Implement Notification System

**Priority:** P1 | **Effort:** 3 days | **Assignee:** Full-Stack Engineer

**Description:**  
Create an in-app notification system that delivers alerts for SLA breaches, over-voting detections, compliance deadlines, and system events.

**Deliverables:**
- New entity: `src/notifications/notification.entity.ts`
  - Fields: `id`, `userId` (FK), `type` (SLA_BREACH, OVERVOTING, COMPLIANCE_DUE, SYSTEM), `title`, `body`, `linkedRecordType`, `linkedRecordId`, `read`, `createdAt`
- `NotificationService` with `create()`, `markRead()`, `markAllRead()`, `getUnread()`
- `GET /api/notifications` — list notifications for authenticated user
- `PATCH /api/notifications/:id/read` — mark as read
- `POST /api/notifications/mark-all-read` — bulk mark read
- WebSocket delivery: push new notifications via `notifications:new` event
- Frontend: notification bell icon in topbar with unread count badge; dropdown panel

**Validation Criteria:**
- [ ] SLA breach generates notification for incident owner and National Director
- [ ] Over-voting alert generates notification for National Director and Legal Team
- [ ] Compliance task due in 48 hours generates notification for task owner
- [ ] Unread count badge updates in real-time via WebSocket
- [ ] Clicking notification navigates to the relevant record

---

### TASK-0303: Implement Automated SLA Escalation

**Priority:** P1 | **Effort:** 2 days | **Assignee:** Backend Engineer

**Description:**  
Add a scheduled job that checks for SLA breaches on unresolved incidents and triggers escalation notifications and status changes.

**Deliverables:**
- Install `@nestjs/schedule`
- Create `SlaEscalationService` with a `@Cron('*/2 * * * *')` (every 2 minutes) job
- Job logic:
  1. Query all incidents with `status NOT IN ('Resolved', 'Closed')` and `resolvedAt IS NULL`
  2. Compute SLA state for each
  3. For newly breached incidents (state changed since last check):
     - Create notification for incident owner + National Director
     - Update incident `escalated` flag (new column)
     - Log audit event
- Prevent duplicate notifications for already-escalated incidents

**Validation Criteria:**
- [ ] Cron job runs every 2 minutes in production
- [ ] Newly breached Critical incident generates notification within 2 minutes of breach
- [ ] Already-escalated incidents do not generate duplicate notifications
- [ ] Audit trail records each escalation event
- [ ] Cron job is disabled in test environment

---

### TASK-0304: Implement Recurring Compliance Rule Scheduler

**Priority:** P1 | **Effort:** 1 day | **Assignee:** Backend Engineer

**Description:**  
Add a scheduled job that auto-generates `ComplianceTask` occurrences for `RECURRING` compliance rules.

**Deliverables:**
- Create `ComplianceSchedulerService` with a `@Cron('0 0 * * *')` (daily midnight) job
- Job logic:
  1. Query all active `ComplianceRule` with `triggerType = RECURRING`
  2. For each rule, check if a task for the current occurrence period already exists
  3. If not, generate a new `ComplianceTask` with calculated due date
- Add `last_generated_at` column to `ComplianceRule` for tracking

**Validation Criteria:**
- [ ] Daily cron generates tasks for monthly recurring rules when due
- [ ] No duplicate tasks generated for the same occurrence period
- [ ] Generated tasks inherit rule's `risk`, `authority`, and `statutory_reference`
- [ ] Retired/superseded rules do not generate new occurrences

---

### TASK-0305: Optimize Dashboard Aggregation Queries

**Priority:** P1 | **Effort:** 2 days | **Assignee:** Backend Engineer

**Description:**  
The dashboard currently runs aggregation queries on every request. Add caching and/or materialized views to handle national-scale data (176,846 PUs).

**Deliverables:**
- Option A (Redis cache): Cache dashboard summary and state readiness results with 30-second TTL; invalidate on relevant entity changes
- Option B (Materialized view): Create PostgreSQL materialized views for state-level aggregations; refresh via scheduled job
- Benchmark with synthetic data: insert 176,846 PU results and measure query time
- Target: dashboard summary response < 500ms with full national data

**Validation Criteria:**
- [ ] Dashboard summary API responds in < 500ms with 176,846+ result forms
- [ ] State readiness query responds in < 500ms with all 37 states populated
- [ ] Cache invalidation occurs within 30 seconds of new data
- [ ] No stale data served beyond the configured TTL

---

## 5. Phase 4 — Offline Resilience & Field Readiness

> **Goal:** Harden the offline-first capability for field deployment and add bulk operations for operational efficiency.

---

### TASK-0401: Implement PWA Service Worker for Asset Caching

**Priority:** P1 | **Effort:** 3 days | **Assignee:** Frontend Engineer

**Description:**  
Add a service worker to the Next.js frontend that caches the application shell (HTML, CSS, JS) for offline access. Currently only data is queued offline; the app shell requires connectivity to load.

**Deliverables:**
- Install `next-pwa` or implement custom service worker
- Cache strategy: "network first, cache fallback" for API calls; "cache first" for static assets
- Precache critical routes: `/login`, `/app` (main dashboard)
- Service worker registration in `layout.tsx`
- Manifest file (`manifest.json`) with app name, icons, theme color
- "Add to Home Screen" support for mobile deployment

**Validation Criteria:**
- [ ] App loads from cache when completely offline (after first visit)
- [ ] Static assets (CSS, JS, fonts) served from cache without network request
- [ ] API calls that fail offline are caught by the existing IndexedDB queue
- [ ] New version of the app detected on next online visit; user prompted to update
- [ ] PWA installable on Chrome/Edge/Safari mobile

---

### TASK-0402: Implement Bulk Agent Import (CSV/Excel)

**Priority:** P1 | **Effort:** 3 days | **Assignee:** Full-Stack Engineer

**Description:**  
Allow National Directors to upload a CSV or Excel file to bulk-create field agent records, with validation against the geography master data.

**Deliverables:**
- Install `csv-parse` or `papaparse` (server-side CSV parsing)
- New endpoint: `POST /api/agents/bulk-import` (multipart file upload)
- Expected CSV columns: `full_name, phone, state, lga, ward, polling_unit, assignment_type`
- Validation logic:
  - All required fields present
  - State/LGA/Ward/PU exist in `geographic_units` table
  - Phone format valid (Nigerian format)
  - No duplicate agents for the same PU in the same election
- Return: `{ imported: N, skipped: N, errors: [{row, field, message}] }`
- Frontend: upload modal with progress indicator; results summary with downloadable error report

**Validation Criteria:**
- [ ] CSV with 1,000 valid agents imports in < 30 seconds
- [ ] Invalid rows are skipped with specific error messages
- [ ] Geography validation catches non-existent PU codes
- [ ] Duplicate PU assignments within same election are flagged
- [ ] Imported agents have correct `electionId` and `geographicUnitId` from geography lookup

---

### TASK-0403: Implement Agent Heartbeat Endpoint

**Priority:** P1 | **Effort:** 2 days | **Assignee:** Full-Stack Engineer

**Description:**  
Create a lightweight heartbeat endpoint that field agents call periodically to report their connectivity and device status.

**Deliverables:**
- New endpoint: `POST /api/agents/:id/heartbeat`
  - Body: `{ batteryLevel?, gpsLatitude?, gpsLongitude?, appVersion? }`
- Updates agent `syncStatus` to 'Online (heartbeat Nm ago)'
- Stores last heartbeat timestamp
- Dashboard: agents without heartbeat for > 10 minutes shown as "Offline"
- Frontend: background heartbeat every 60 seconds when logged in as agent role

**Validation Criteria:**
- [ ] Heartbeat updates agent's `syncStatus` and `lastHeartbeatAt` timestamp
- [ ] Agents without heartbeat for > 10 minutes transition to "Offline" in dashboard
- [ ] Heartbeat request is lightweight (< 200 bytes request, < 100 bytes response)
- [ ] Heartbeat endpoint is rate-limited separately (1 per 30 seconds per agent)

---

### TASK-0404: Mobile-Optimized Field Agent View

**Priority:** P1 | **Effort:** 3 days | **Assignee:** Frontend Engineer

**Description:**  
Create a simplified, mobile-optimized view for PU/Collation Agent role users focused on their core tasks: submit results, report incidents, upload evidence.

**Deliverables:**
- Responsive layout optimized for 360px–414px viewport
- Quick-action cards: "Submit EC8A Result", "Report Incident", "Upload Evidence"
- Camera integration for evidence capture (use `<input type="file" capture="environment">`)
- Offline status indicator prominently displayed
- Sync queue status showing pending/conflict counts
- Large touch targets (minimum 48x48px) for field use with gloves

**Validation Criteria:**
- [ ] All three core actions accessible within 2 taps from the home screen
- [ ] Camera capture opens directly to rear camera for evidence photos
- [ ] Offline indicator visible at all times
- [ ] No horizontal scrolling on any 360px viewport
- [ ] Font sizes ≥ 16px to prevent auto-zoom on iOS

---

### TASK-0405: Enhance Offline Sync Conflict Resolution UI

**Priority:** P1 | **Effort:** 2 days | **Assignee:** Frontend Engineer

**Description:**  
Improve the conflict review UI in the Administration view to provide clearer context and easier resolution.

**Deliverables:**
- Show the original queued request data (what was attempted)
- Show the server's rejection reason
- Show the current server state of the record (if applicable)
- "Edit & Resubmit" action: open the record in edit mode with the queued data pre-filled
- "Discard" action with confirmation dialog
- Batch operations: "Retry All" and "Discard All" buttons

**Validation Criteria:**
- [ ] Conflict items show both the queued data and server error reason
- [ ] "Edit & Resubmit" opens the correct module's create/edit form
- [ ] "Discard" requires confirmation and removes item from IndexedDB
- [ ] Batch operations work correctly for 10+ conflict items
- [ ] Conflict count updates in topbar after resolution

---

## 6. Phase 5 — Intelligence, Analytics & Reporting

> **Goal:** Complete the analytical capabilities for results collation, intelligence maturity, and governance reporting.

---

### TASK-0501: Implement Ward/LGA/State Collation Roll-Up

**Priority:** P1 | **Effort:** 4 days | **Assignee:** Backend Engineer

**Description:**  
Aggregate PU-level results to ward, LGA, and state levels for collation comparison. This is the foundation for verifying official collation results.

**Deliverables:**
- New entity: `src/results/collation-summary.entity.ts`
  - Fields: `id`, `electionId`, `geographicUnitId`, `level` (Ward/LGA/State/National), `totalRegisteredVoters`, `totalAccreditedVoters`, `totalValidVotes`, `totalRejectedVotes`, `puCount`, `puReportedCount`, `reportingPercentage`, `computedAt`
- `CollationService` with `computeCollation(electionId, geographicUnitId)` method
- Aggregate queries using geography `path` prefix matching
- `GET /api/results/collation?electionId=X&geographicUnitId=Y` — returns collation summary
- `GET /api/results/collation/tree?electionId=X` — returns full collation tree (national → states → LGAs)
- Per-party aggregation from `ResultCandidateScore` (depends on TASK-0201)

**Validation Criteria:**
- [ ] Ward collation sums all PU results within the ward correctly
- [ ] LGA collation sums all ward collations within the LGA correctly
- [ ] State collation sums all LGA collations within the state correctly
- [ ] `reportingPercentage` accurately reflects PUs reported vs. total PUs in geography
- [ ] Per-party vote totals aggregate correctly at each level
- [ ] Collation handles partially reported wards/LGAs (some PUs missing)

---

### TASK-0502: Implement IReV Comparison Layer

**Priority:** P2 | **Effort:** 2 days | **Assignee:** Full-Stack Engineer

**Description:**  
Add the ability to manually enter or import IReV-published results for comparison against ES360-captured results.

**Deliverables:**
- New entity: `src/results/irev-result.entity.ts`
  - Fields: `id`, `resultFormId` (FK), `accreditedVoters`, `validVotes`, `rejectedVotes`, `source` ('Manual entry' | 'API import'), `enteredBy`, `createdAt`
- `POST /api/results/:id/irev-comparison` — enter IReV figures
- Update reconciliation service to include IReV comparison as layer 4
- Discrepancy: if ES360 and IReV figures differ by > 5%, flag as "IReV Divergence"
- Frontend: IReV entry form in reconciliation report; divergence highlighted in red

**Validation Criteria:**
- [ ] IReV figures stored separately from ES360 captured figures
- [ ] Reconciliation report now shows 4 layers (was 3): agent evidence, double-entry, arithmetic, IReV
- [ ] > 5% divergence flagged with clear visual indicator
- [ ] IReV data is optional; reconciliation works without it (reports "Not available")

---

### TASK-0503: Build Constituency Analytics Dashboard

**Priority:** P2 | **Effort:** 3 days | **Assignee:** Full-Stack Engineer

**Description:**  
Create a dedicated dashboard for governance casework analytics, enabling elected representatives to track mandate delivery.

**Deliverables:**
- `GET /api/governance/analytics?geographicUnitId=X` — returns aggregated casework metrics
  - Total cases by sector (Health, Education, Infrastructure, Energy, Agriculture, Security)
  - Cases by status (Open, In Progress, Resolved, Closed)
  - Average resolution time by sector
  - Overdue case count and percentage
  - Monthly trend (cases opened/resolved per month)
- Frontend: dedicated Governance Analytics view with charts (bar charts, donut charts, trend lines)
- Chart library: use native SVG or lightweight library (Chart.js via CDN)

**Validation Criteria:**
- [ ] Analytics correctly aggregate all governance cases for the selected geography
- [ ] Sector breakdown chart shows accurate counts
- [ ] Monthly trend chart shows last 12 months of data
- [ ] Overdue cases highlighted with appropriate urgency indicators
- [ ] Dashboard loads in < 2 seconds with 500+ cases

---

### TASK-0504: Implement Narrative Clustering for Intelligence Signals

**Priority:** P2 | **Effort:** 4 days | **Assignee:** Backend Engineer

**Description:**  
Group related intelligence signals into tracked "narratives" using text similarity and geographic proximity.

**Deliverables:**
- New entity: `src/intelligence/narrative.entity.ts`
  - Fields: `id`, `title`, `summary`, `status` (Emerging / Active / Countered / Dismissed), `signalCount`, `geographicScope`, `firstSeenAt`, `lastSeenAt`, `createdAt`
- New join entity: `src/intelligence/narrative-signal.entity.ts`
  - Fields: `narrativeId`, `signalId`
- Clustering logic: TF-IDF or keyword overlap scoring between new signal and existing open narratives
- If similarity > threshold: link to existing narrative
- If no match: create new narrative from signal
- `GET /api/intelligence/narratives` — list narratives with signal count
- `GET /api/intelligence/narratives/:id/signals` — list signals in a narrative
- Frontend: replace mock narrative UI with real data

**Validation Criteria:**
- [ ] Two signals about "fake polling unit relocation in Ikeja" cluster into one narrative
- [ ] Unrelated signals create separate narratives
- [ ] Narrative status tracks lifecycle: Emerging → Active → Countered
- [ ] Narrative summary auto-generates from first signal's content
- [ ] Dashboard shows active narrative count

---

### TASK-0505: Export & Reporting Module

**Priority:** P1 | **Effort:** 3 days | **Assignee:** Full-Stack Engineer

**Description:**  
Add data export capabilities for compliance reporting, tribunal preparation, and operational analysis.

**Deliverables:**
- `GET /api/export/results?electionId=X&format=csv` — export all results as CSV
- `GET /api/export/incidents?electionId=X&format=csv` — export all incidents as CSV
- `GET /api/export/finance?electionId=X&format=csv` — export finance ledger as CSV
- `GET /api/export/audit?electionId=X&format=csv` — export audit trail as CSV
- `GET /api/export/agents?electionId=X&format=csv` — export agent roster as CSV
- PDF report generation: `POST /api/export/election-report?electionId=X` — comprehensive election summary report
- Frontend: export buttons on each module's list view

**Validation Criteria:**
- [ ] CSV exports include all fields, properly escaped
- [ ] CSV downloads trigger browser download (Content-Disposition header)
- [ ] PDF election report includes: summary metrics, state readiness, incident summary, results summary
- [ ] Exports are restricted to National Director, Legal, and Super Admin roles
- [ ] Large exports (100,000+ records) complete without timeout (streaming)

---

## 7. Phase 6 — Infrastructure, Deployment & Operational Readiness

> **Goal:** Prepare the application for production deployment with robust infrastructure, CI/CD, monitoring, and disaster recovery.

---

### TASK-0601: Harden Docker Compose for Production

**Priority:** P0 | **Effort:** 2 days | **Assignee:** DevOps Engineer

**Description:**  
Update the Docker Compose configuration for production readiness with proper health checks, resource limits, and secrets management.

**Deliverables:**
- Multi-stage Dockerfiles for API and Web (build + runtime stages)
- Health checks for all services
- Resource limits (CPU, memory) for each container
- Environment variable management via `.env` files (not hardcoded)
- Named volumes for persistent data (PostgreSQL, MinIO)
- Network isolation: internal network for DB/storage, external for API/web
- Production-ready `docker-compose.prod.yml` overlay

**Validation Criteria:**
- [ ] `docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d` starts all services
- [ ] API container < 256MB memory at idle
- [ ] Web container < 128MB memory at idle
- [ ] Health checks correctly identify unhealthy containers
- [ ] Secrets not visible in `docker inspect` output

---

### TASK-0602: Implement CI/CD Pipeline

**Priority:** P0 | **Effort:** 3 days | **Assignee:** DevOps Engineer

**Description:**  
Set up automated CI/CD using GitHub Actions (or equivalent) for testing, building, and deploying.

**Deliverables:**
- `.github/workflows/ci.yml`:
  - Trigger: push to `main`, pull request to `main`
  - Steps: lint → unit tests → integration tests → build
  - PostgreSQL service container for integration tests
  - Coverage report upload as artifact
  - Fail pipeline on < 80% coverage
- `.github/workflows/deploy.yml`:
  - Trigger: tag push (`v*.*.*`)
  - Steps: build Docker images → push to registry → deploy to staging
  - Manual approval gate for production deployment
- Branch protection: require CI pass + 1 approval for merge to `main`

**Validation Criteria:**
- [ ] CI pipeline runs on every push and PR
- [ ] Pipeline fails on test failure or lint error
- [ ] Pipeline fails if coverage drops below 80%
- [ ] Docker images built and pushed to registry on tag
- [ ] Deployment to staging is automated; production requires manual approval

---

### TASK-0603: Set Up Monitoring & Alerting

**Priority:** P1 | **Effort:** 3 days | **Assignee:** DevOps Engineer

**Description:**  
Deploy Prometheus + Grafana for metrics collection and visualization, with alerting for critical system events.

**Deliverables:**
- Install `@willsoto/nestjs-prometheus` or `prom-client` for NestJS metrics
- Expose `/metrics` endpoint (Prometheus format)
- Key metrics (see Architecture doc §11.2):
  - `api_request_duration_seconds` (histogram)
  - `api_request_total` (counter by method, path, status)
  - `db_connection_pool_active` (gauge)
  - `incidents_sla_breached_total` (counter)
  - `results_overvoting_detected_total` (counter)
  - `evidence_hash_mismatch_total` (counter)
  - `offline_queue_pending_total` (gauge)
- Grafana dashboard with panels for all key metrics
- Alert rules: SLA breach, over-voting detected, hash mismatch, error rate > 5%, p99 latency > 2s

**Validation Criteria:**
- [ ] `/metrics` endpoint returns Prometheus-format metrics
- [ ] Grafana dashboard displays all key panels
- [ ] Alert fires within 5 minutes of SLA breach
- [ ] Alert fires immediately on evidence hash mismatch
- [ ] Historical metrics retained for at least 30 days

---

### TASK-0604: Implement Database Backup & Recovery

**Priority:** P0 | **Effort:** 2 days | **Assignee:** DevOps Engineer

**Description:**  
Set up automated PostgreSQL backup with point-in-time recovery capability.

**Deliverables:**
- Automated daily full backup (pg_dump or pgBackRest)
- WAL archiving for point-in-time recovery
- Backup storage: S3-compatible bucket (separate from evidence vault)
- Backup retention: 30 days for daily, 1 year for monthly
- Documented recovery procedure with estimated RTO/RPO
- Backup verification: weekly automated restore test to temporary instance

**Validation Criteria:**
- [ ] Daily backup completes successfully and is uploaded to S3
- [ ] Point-in-time recovery tested and documented
- [ ] Recovery from latest backup completes in < 30 minutes (RTO)
- [ ] Maximum data loss window: 5 minutes (RPO with WAL archiving)
- [ ] Weekly automated restore test passes without errors

---

### TASK-0605: Load Testing & Performance Benchmarks

**Priority:** P1 | **Effort:** 2 days | **Assignee:** Performance Engineer

**Description:**  
Conduct load testing to validate the system can handle election-day peak loads: 176,846 PUs reporting simultaneously.

**Deliverables:**
- Load testing tool: k6 or Artillery
- Test scenarios:
  1. **Burst result ingestion**: 10,000 concurrent result submissions in 60 seconds
  2. **Dashboard under load**: 500 concurrent dashboard viewers during result ingestion
  3. **Authentication storm**: 1,000 concurrent login attempts
  4. **Evidence upload**: 500 concurrent 5MB file uploads
- Performance targets:
  - Result ingestion: p99 < 2s, 0% error rate
  - Dashboard: p99 < 500ms
  - Login: p99 < 1s
  - Evidence upload: p99 < 10s
- Report with bottleneck analysis and optimization recommendations

**Validation Criteria:**
- [ ] All four scenarios execute without system crash
- [ ] Performance targets met for all scenarios
- [ ] Report identifies any bottlenecks with specific remediation recommendations
- [ ] Load test scripts committed to repository for repeatability

---

## 8. Phase 7 — Security Hardening & Compliance Audit

> **Goal:** Final security review and hardening before production deployment.

---

### TASK-0701: Encrypt MFA Secrets at Rest

**Priority:** P0 | **Effort:** 1 day | **Assignee:** Security Engineer

**Description:**  
Currently `mfa_secret` is stored as plaintext in the database. Encrypt it with application-level AES-256-GCM using a key from environment variables.

**Deliverables:**
- Create `src/common/crypto.service.ts` with `encrypt(plaintext)` and `decrypt(ciphertext)` methods
- Use AES-256-GCM with random IV per encryption
- Encryption key sourced from `MFA_ENCRYPTION_KEY` environment variable
- Update `AuthService` to encrypt on enrollment, decrypt on verification
- Migration to encrypt existing plaintext secrets

**Validation Criteria:**
- [ ] `mfa_secret` column contains only encrypted values (not readable TOTP secrets)
- [ ] TOTP verification still works correctly after encryption
- [ ] Encryption key rotation procedure documented
- [ ] Missing `MFA_ENCRYPTION_KEY` at startup throws clear error, not silent failure

---

### TASK-0702: Implement Refresh Token Rotation with Reuse Detection

**Priority:** P0 | **Effort:** 2 days | **Assignee:** Backend Engineer

**Description:**  
Implement proper refresh token rotation: each refresh produces a new pair (access + refresh), and the old refresh token is revoked. If a revoked token is reused (indicating theft), revoke all tokens for that user.

**Deliverables:**
- `POST /api/auth/refresh` endpoint
- On refresh: issue new access token + new refresh token; revoke the used refresh token
- On reuse of revoked token: revoke ALL refresh tokens for the user (force re-login)
- Add `family_id` column to `refresh_tokens` for tracking token lineage
- Frontend: Axios interceptor automatically refreshes on 401 with retry

**Validation Criteria:**
- [ ] Valid refresh token produces new access + refresh tokens
- [ ] Used refresh token returns 401 on second use
- [ ] Reused (revoked) refresh token triggers full session revocation for the user
- [ ] Frontend seamlessly refreshes tokens without user-visible interruption
- [ ] Concurrent requests during refresh don't cause race conditions

---

### TASK-0703: Security Audit & Penetration Testing Preparation

**Priority:** P0 | **Effort:** 3 days | **Assignee:** Security Engineer

**Description:**  
Conduct an internal security audit and prepare the application for external penetration testing.

**Deliverables:**
- OWASP Top 10 checklist review:
  1. **Injection**: Verify all queries use parameterized statements
  2. **Broken Auth**: Verify token lifecycle, MFA enforcement, password complexity
  3. **Sensitive Data Exposure**: Verify encryption at rest and in transit
  4. **XXE**: Verify no XML processing
  5. **Broken Access Control**: Verify RBAC on all endpoints, geography scoping
  6. **Security Misconfiguration**: Verify Helmet headers, CORS, error messages
  7. **XSS**: Verify React escaping, CSP headers
  8. **Insecure Deserialization**: Verify DTO validation, class-transformer settings
  9. **Known Vulnerabilities**: Run `npm audit` and remediate critical/high findings
  10. **Insufficient Logging**: Verify audit trail coverage
- Fix all identified vulnerabilities
- Document residual risks and mitigations
- Prepare scope document for external penetration test

**Validation Criteria:**
- [ ] All 10 OWASP categories reviewed with documented findings
- [ ] All critical and high vulnerabilities remediated
- [ ] `npm audit` returns 0 critical and 0 high vulnerabilities
- [ ] Penetration test scope document produced with system boundaries and credentials
- [ ] Security audit report signed off by technical lead

---

### TASK-0704: Implement Content Security Policy (CSP)

**Priority:** P1 | **Effort:** 1 day | **Assignee:** Frontend Engineer

**Description:**  
Configure a strict Content Security Policy to prevent XSS attacks, including inline script restrictions and trusted source whitelisting.

**Deliverables:**
- Configure Helmet CSP in `main.ts` with:
  - `default-src: 'self'`
  - `script-src: 'self'` (no `'unsafe-inline'` or `'unsafe-eval'`)
  - `style-src: 'self' 'unsafe-inline'` (required for Tailwind)
  - `img-src: 'self' data: blob:` (for camera captures and QR codes)
  - `connect-src: 'self' wss:` (for WebSocket connections)
  - `font-src: 'self'`
- Fix any CSP violations in the frontend code
- CSP reporting endpoint for violation monitoring (optional)

**Validation Criteria:**
- [ ] CSP header present on all responses
- [ ] No CSP violations in browser console during normal app usage
- [ ] Inline scripts blocked (if any accidentally added)
- [ ] Third-party script injection blocked
- [ ] Camera capture and QR code display still work under CSP

---

### TASK-0705: Final Pre-Launch Checklist

**Priority:** P0 | **Effort:** 1 day | **Assignee:** Tech Lead

**Description:**  
Execute the final pre-launch checklist to verify production readiness across all dimensions.

**Deliverables:**
- Checklist items:
  - [ ] All P0 tasks from Phases 1–7 completed and verified
  - [ ] Test suite passes with ≥ 80% coverage
  - [ ] CI/CD pipeline green on `main` branch
  - [ ] Database migrations tested against production-equivalent schema
  - [ ] INEC geography master data imported (or import procedure documented)
  - [ ] Demo accounts removed; real admin accounts created
  - [ ] MFA enforced for all privileged roles
  - [ ] CORS restricted to production domains
  - [ ] TLS/SSL configured on load balancer
  - [ ] Backup and recovery tested within last 7 days
  - [ ] Monitoring dashboards operational
  - [ ] Alert channels configured (email, SMS, Slack)
  - [ ] Incident response runbook documented
  - [ ] Load test passed at expected peak load
  - [ ] Security audit findings remediated
  - [ ] Evidence vault encryption verified
  - [ ] Audit trail retention policy configured
  - [ ] DNS and domain configuration complete
  - [ ] Rate limiting tuned for production traffic patterns
  - [ ] Error pages and 404 handling verified

**Validation Criteria:**
- [ ] All checklist items marked complete
- [ ] Sign-off from: Tech Lead, Security Lead, Operations Lead
- [ ] Launch communication prepared for all stakeholders

---

## 9. Cross-Phase Dependency Map

```mermaid
graph LR
    subgraph "Phase 1 (Foundation)"
        T0101["TASK-0101<br/>Test Framework"]
        T0102["TASK-0102<br/>Auth Tests"]
        T0103["TASK-0103<br/>Domain Tests"]
        T0104["TASK-0104<br/>E2E Tests"]
        T0105["TASK-0105<br/>Migrations"]
        T0106["TASK-0106<br/>Swagger"]
        T0107["TASK-0107<br/>Logging"]
        T0108["TASK-0108<br/>Health Checks"]
        T0109["TASK-0109<br/>CORS"]
        T0110["TASK-0110<br/>Geo Scope Filter"]
    end

    subgraph "Phase 2 (Data Integrity)"
        T0201["TASK-0201<br/>Party Scores"]
        T0202["TASK-0202<br/>Over-Voting Alerts"]
        T0203["TASK-0203<br/>§84 Certificate"]
        T0204["TASK-0204<br/>GPS Evidence"]
        T0205["TASK-0205<br/>Evidence Record Type"]
        T0206["TASK-0206<br/>TZ Silence Period"]
    end

    subgraph "Phase 3 (Real-Time)"
        T0301["TASK-0301<br/>WebSocket Gateway"]
        T0302["TASK-0302<br/>Notifications"]
        T0303["TASK-0303<br/>SLA Escalation"]
        T0304["TASK-0304<br/>Rule Scheduler"]
        T0305["TASK-0305<br/>Dashboard Perf"]
    end

    subgraph "Phase 4 (Field Ready)"
        T0401["TASK-0401<br/>Service Worker"]
        T0402["TASK-0402<br/>Bulk Import"]
        T0403["TASK-0403<br/>Agent Heartbeat"]
        T0404["TASK-0404<br/>Mobile Agent UX"]
        T0405["TASK-0405<br/>Conflict UI"]
    end

    subgraph "Phase 5 (Analytics)"
        T0501["TASK-0501<br/>Collation Roll-Up"]
        T0502["TASK-0502<br/>IReV Comparison"]
        T0503["TASK-0503<br/>Governance Analytics"]
        T0504["TASK-0504<br/>Narrative Clustering"]
        T0505["TASK-0505<br/>Export Module"]
    end

    subgraph "Phase 6 (Infrastructure)"
        T0601["TASK-0601<br/>Docker Prod"]
        T0602["TASK-0602<br/>CI/CD"]
        T0603["TASK-0603<br/>Monitoring"]
        T0604["TASK-0604<br/>DB Backup"]
        T0605["TASK-0605<br/>Load Testing"]
    end

    subgraph "Phase 7 (Security)"
        T0701["TASK-0701<br/>MFA Encryption"]
        T0702["TASK-0702<br/>Token Rotation"]
        T0703["TASK-0703<br/>Security Audit"]
        T0704["TASK-0704<br/>CSP"]
        T0705["TASK-0705<br/>Launch Checklist"]
    end

    %% Phase 1 internal
    T0101 --> T0102
    T0101 --> T0103
    T0102 --> T0104
    T0103 --> T0104

    %% Phase 1 → Phase 2
    T0105 --> T0201
    T0103 --> T0202

    %% Phase 1 → Phase 3
    T0107 --> T0301
    T0108 --> T0301
    T0103 --> T0303

    %% Phase 2 → Phase 5
    T0201 --> T0501
    T0201 --> T0502

    %% Phase 3 → Phase 4
    T0301 --> T0401
    T0302 --> T0303

    %% Phase 5 dependencies
    T0501 --> T0502

    %% Phase 6 dependencies
    T0104 --> T0602
    T0107 --> T0603
    T0108 --> T0603

    %% Phase 7 dependencies
    T0602 --> T0703
    T0603 --> T0705
    T0604 --> T0705
    T0703 --> T0705
