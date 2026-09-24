# ISO 27001 Portal — Mock / Placeholder Removal & Real Integration Plan

**Surface:** Compliance Portal — ISO 27001 Framework  
**Goal:** Catalog every mock contract, hardcoded identity, placeholder stub, and demo-mode artefact still active in the ISO 27001 portal; provide a phased implementation plan with unique task IDs and validation criteria for replacing each with production-grade integration.  
**Date:** 2026-08-10  
**Depends on:** [ISO27001_Portal_Redesign_Tasks.md](./ISO27001_Portal_Redesign_Tasks.md) (Phases 0–5 = Done)

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Severity Classification](#2-severity-classification)
3. [Finding Catalogue](#3-finding-catalogue)
   - Category A: Demo Mode / API Mock Layer
   - Category B: Hardcoded Identity & Fallback Values
   - Category C: Placeholder Empty-State Tabs (No Real Backend)
   - Category D: CISO Role Gate Hardcoded to Email
   - Category E: Clause 7 "Demo CISO" Fallback
4. [Phased Task Plan](#4-phased-task-plan)
5. [Task ID Quick-Reference Index](#5-task-id-quick-reference-index)
6. [File Map](#6-file-map)
7. [Verification Plan](#7-verification-plan)

---

## 1. Executive Summary

After the ISO 27001 Portal Redesign (Phases 0–5, all **Done**), the portal's **Annex A scoring engine**, **SoA ownership**, **Overview metrics**, and **secondary tab generators** have been replaced with real data or honest empty states. However, several systemic mock patterns remain:

| Category | Count | Risk |
|----------|-------|------|
| **A** — Demo API mock layer (`demo-api-mock.ts`, `demo-ciso-session.ts`) | 2 modules + `api.ts` interceptor | 🔴 Critical — intercepts ALL API calls in demo mode; in-memory data never touches real DB |
| **B** — Hardcoded identity fallbacks (`"Ismail Muhammad"`, `"Demo CISO"`, `ciso@gtbank.com`) | 11 occurrences across 8 files | 🟡 Medium — leaks specific names/emails into production UI |
| **C** — Tabs backed by empty-state only (no real API) | 1 tab (`ActivitiesTab`) | 🟢 Low — honest empty state, but no pathway to populate |
| **D** — CISO role gate hardcoded to `ciso@gtbank.com` email | 8 files, 8 occurrences | 🟡 Medium — breaks RBAC for any non-GTBank institution |
| **E** — Clause 7 "Demo CISO" string fallback | 4 occurrences in `clause-7.tsx` | 🟡 Medium — visible to end-users when `user.name` is empty |

**Total unique findings:** 24 remediations across 5 categories.

---

## 2. Severity Classification

| Level | Meaning | Remediation Urgency |
|-------|---------|---------------------|
| 🔴 **Critical** | Mock data silently replaces real API; production users may see fake data without knowing | Sprint 1 |
| 🟡 **Medium** | Hardcoded values leak into UI or break multi-tenant RBAC | Sprint 2 |
| 🟢 **Low** | Honest "coming soon" or empty state; no data fabrication | Backlog |

---

## 3. Finding Catalogue

### Category A: Demo Mode / API Mock Layer

> **Architecture:** When `useDemoStore().isDemoMode === true`, the `ApiClient.request()` method in `api.ts` intercepts HTTP requests and routes them to `demo-api-mock.ts` which returns in-memory seed data. This layer covers assessments, remediation, workforce, threats, reports, and dashboard endpoints. A companion `demo-ciso-session.ts` bootstraps a fake auth session.

#### MOCK-A-01 — `demo-api-mock.ts` in-memory API stubs

| Field | Value |
|-------|-------|
| **ID** | `MOCK-A-01` |
| **Severity** | 🔴 Critical |
| **File** | [`demo-api-mock.ts`](file:///Users/oluwatobiloba/Desktop/sbtsgroup/regguard/frontend/lib/demo-api-mock.ts) |
| **Lines** | 1–707 (entire file) |
| **What** | 707-line module with `handleDemoMockRequest()` + `isDemoMockEndpoint()` providing seed data for assessments, remediation, workforce, threats, reports, report-logs. Contains hardcoded "Demo Financial Institution", composite score `2.73`, demo workforce names ("Ada Okonkwo", "Chidi Eze", "Funke Adeyemi"), chart data, and evidence records. |
| **Impact** | Any session in demo mode receives entirely fabricated data across ALL portal surfaces. The mock DB is never persisted. |
| **Replacement** | If demo mode is a **product feature** (guided tour): isolate behind a feature flag with explicit "DEMO" watermark on all UI surfaces. If demo mode should be removed: delete `demo-api-mock.ts` and all references. |

#### MOCK-A-02 — `demo-ciso-session.ts` fake auth bootstrap

| Field | Value |
|-------|-------|
| **ID** | `MOCK-A-02` |
| **Severity** | 🔴 Critical |
| **File** | [`demo-ciso-session.ts`](file:///Users/oluwatobiloba/Desktop/sbtsgroup/regguard/frontend/lib/demo-ciso-session.ts) |
| **Lines** | 1–34 (entire file) |
| **What** | Creates a fake `AuthUser` with `id: "demo-ciso-user"`, `name: "Demo CISO"`, `email: "demo.ciso@aegis360ai.demo"`, `role: "CISO"`, and a placeholder JWT `"demo-guided-ciso-session"`. Logs in without touching `/auth/login`. |
| **Impact** | Bypasses real authentication entirely. The fake JWT would be rejected by NestJS guards but the demo mock layer prevents those requests from ever reaching the backend. |
| **Replacement** | Keep only if demo mode is a product feature. Otherwise remove and ensure all auth flows go through real `/auth/login`. |

#### MOCK-A-03 — `api.ts` demo interceptor

| Field | Value |
|-------|-------|
| **ID** | `MOCK-A-03` |
| **Severity** | 🔴 Critical |
| **File** | [`api.ts`](file:///Users/oluwatobiloba/Desktop/sbtsgroup/regguard/frontend/services/api.ts) |
| **Lines** | 2–6 (imports), 33–43 (interceptor block) |
| **What** | Imports `handleDemoMockRequest`, `isDemoMockEndpoint`, `MockRequestMethod` from `demo-api-mock`. Inside `request()`, checks `isDemoModeActive()` and returns mock data before ever calling `fetch()`. |
| **Impact** | Root of all mock interception — every service module (`assessmentsService`, `riskService`, etc.) is silently mocked when demo mode is active. |
| **Replacement** | Remove the interceptor block and imports if demo mode is deleted. If kept, add UI-visible "DEMO MODE" watermark. |

---

### Category B: Hardcoded Identity & Fallback Values

#### MOCK-B-01 — `"Ismail Muhammad"` hardcoded approver name

| Field | Value |
|-------|-------|
| **ID** | `MOCK-B-01` |
| **Severity** | 🟡 Medium |
| **File** | [`improvement-tab.tsx`](file:///Users/oluwatobiloba/Desktop/sbtsgroup/regguard/frontend/components/institution/compliance-portal/tabs/improvement-tab.tsx) |
| **Line** | 341 |
| **Code** | `const approverName = user?.name || user?.email || "Ismail Muhammad";` |
| **What** | If the authenticated user has no `name` or `email`, the system records "Ismail Muhammad" as the change request approver. |
| **Impact** | Audit trail records a specific person's name for approvals made by other users. Compliance violation. |
| **Replacement** | Use a generic fallback: `user?.name \|\| user?.email \|\| t("portal.improvement.unknownApprover")`. The backend should reject approvals without valid user context. |

#### MOCK-B-02 — `"Demo CISO"` name fallback in Clause 7

| Field | Value |
|-------|-------|
| **ID** | `MOCK-B-02` |
| **Severity** | 🟡 Medium |
| **File** | [`clause-7.tsx`](file:///Users/oluwatobiloba/Desktop/sbtsgroup/regguard/frontend/components/institution/compliance-portal/tabs/clause-tab/clause-7.tsx) |
| **Lines** | 1737, 2018, 2558, 2838 |
| **Code** | `const cisoName = user?.name \|\| "Demo CISO";` (4 occurrences) |
| **What** | When recording competence records, awareness programmes, and training frequency changes, the CISO name defaults to "Demo CISO" if `user.name` is falsy. |
| **Impact** | Production records may contain "Demo CISO" in compliance documents if user profile is incomplete. |
| **Replacement** | `const cisoName = user?.name \|\| user?.email \|\| t("portal.clause7.unknownUser");` — and add a profile completeness check that warns users to set their display name. |

#### MOCK-B-03 — `"Demo CISO"` in `demo-api-mock.ts` comment author

| Field | Value |
|-------|-------|
| **ID** | `MOCK-B-03` |
| **Severity** | 🟡 Medium (if demo mode kept) / 🔴 Critical (if removed) |
| **File** | [`demo-api-mock.ts`](file:///Users/oluwatobiloba/Desktop/sbtsgroup/regguard/frontend/lib/demo-api-mock.ts) |
| **Line** | 124 |
| **What** | Seed comment author `name: "Demo CISO"` in mock remediation comments. |
| **Impact** | Only visible in demo mode. Resolved by MOCK-A-01 removal. |
| **Replacement** | Covered by MOCK-A-01. |

---

### Category C: Placeholder Empty-State Tabs (No Real Backend)

#### MOCK-C-01 — `ActivitiesTab` — pure empty state

| Field | Value |
|-------|-------|
| **ID** | `MOCK-C-01` |
| **Severity** | 🟢 Low |
| **File** | [`activities-tab.tsx`](file:///Users/oluwatobiloba/Desktop/sbtsgroup/regguard/frontend/components/institution/compliance-portal/tabs/activities-tab.tsx) |
| **Lines** | 1–24 (entire file) |
| **What** | Renders `<PortalEmptyState>` with i18n strings. No API call, no data source. |
| **Impact** | Honest — shows empty state, never fabricates data. But there is no pathway for users to populate activities. |
| **Replacement** | Wire to `monitoringService.getActivities()` (already used by `calendar-tab.tsx`) or a dedicated activities API. Show the same data as the ISMS Calendar tab's monitoring events. |

#### MOCK-C-02 — `csat-placeholder-tab.tsx` — CBN CSAT placeholder

| Field | Value |
|-------|-------|
| **ID** | `MOCK-C-02` |
| **Severity** | 🟢 Low |
| **File** | [`csat-placeholder-tab.tsx`](file:///Users/oluwatobiloba/Desktop/sbtsgroup/regguard/frontend/components/institution/compliance-portal/tabs/csat-placeholder-tab.tsx) |
| **What** | Generic "Content Coming" placeholder for unimplemented CSAT workbook tabs. |
| **Impact** | Honest empty state. Not ISO 27001 specific — only used in CBN CSAT portal context. |
| **Replacement** | Out of scope for ISO 27001 mock removal. Leave for CSAT workbook implementation. |

---

### Category D: CISO Role Gate Hardcoded to Email

> **Pattern:** `const isCISO = user?.role === "CISO" || user?.email === "ciso@gtbank.com";`
> This appears in **8 files** and hardcodes a specific GTBank email as a CISO escape hatch. Production deployments for other institutions will deny CISO privileges to their actual CISOs unless they happen to share that email.

#### MOCK-D-01 — Hardcoded `ciso@gtbank.com` email gate

| Field | Value |
|-------|-------|
| **ID** | `MOCK-D-01` |
| **Severity** | 🟡 Medium |
| **Affected Files** | 8 files |

| # | File | Line |
|---|------|------|
| 1 | [`controls-tab.tsx`](file:///Users/oluwatobiloba/Desktop/sbtsgroup/regguard/frontend/components/institution/compliance-portal/tabs/controls-tab.tsx) | 82 |
| 2 | [`incidents-tab.tsx`](file:///Users/oluwatobiloba/Desktop/sbtsgroup/regguard/frontend/components/institution/compliance-portal/tabs/incidents-tab.tsx) | 203 |
| 3 | [`vendors-tab.tsx`](file:///Users/oluwatobiloba/Desktop/sbtsgroup/regguard/frontend/components/institution/compliance-portal/tabs/vendors-tab.tsx) | 85 |
| 4 | [`improvement-tab.tsx`](file:///Users/oluwatobiloba/Desktop/sbtsgroup/regguard/frontend/components/institution/compliance-portal/tabs/improvement-tab.tsx) | 53 |
| 5 | [`clause-4.tsx`](file:///Users/oluwatobiloba/Desktop/sbtsgroup/regguard/frontend/components/institution/compliance-portal/tabs/clause-tab/clause-4.tsx) | 162 |
| 6 | [`clause-5.tsx`](file:///Users/oluwatobiloba/Desktop/sbtsgroup/regguard/frontend/components/institution/compliance-portal/tabs/clause-tab/clause-5.tsx) | 95 |
| 7 | [`clause-shared.tsx`](file:///Users/oluwatobiloba/Desktop/sbtsgroup/regguard/frontend/components/institution/compliance-portal/tabs/clause-tab/clause-shared.tsx) | 61 |
| 8 | [`likelihood-impact.tsx`](file:///Users/oluwatobiloba/Desktop/sbtsgroup/regguard/frontend/components/institution/compliance-portal/tabs/clause6-components/likelihood-impact.tsx) | 44 |

| **What** | All files use the same pattern: `user?.role === "CISO" \|\| user?.email === "ciso@gtbank.com"` |
| **Impact** | RBAC bypass for one specific email; RBAC denial for all other institutions' CISOs if their email differs. |
| **Replacement** | Create a shared utility `isCISOUser(user)` in `lib/auth-utils.ts` that checks `user.role === "CISO"` only (or includes institution-level role mapping from the backend). Remove all `ciso@gtbank.com` references. |

**Validation:** `rg "ciso@gtbank" frontend/` returns 0 results after fix.

---

### Category E: Miscellaneous Mock Patterns

#### MOCK-E-01 — `DemoModal.tsx` imports `demo-api-mock`

| Field | Value |
|-------|-------|
| **ID** | `MOCK-E-01` |
| **Severity** | 🟡 Medium |
| **File** | [`DemoModal.tsx`](file:///Users/oluwatobiloba/Desktop/sbtsgroup/regguard/frontend/components/public/DemoModal.tsx) |
| **What** | Public-facing demo modal that triggers demo mode. References demo-api-mock module. |
| **Impact** | Entry point for the demo flow. |
| **Replacement** | If demo mode removed, delete or gut this component. If kept, add "DEMO" watermarks. |

#### MOCK-E-02 — `useDemoStore` state management

| Field | Value |
|-------|-------|
| **ID** | `MOCK-E-02` |
| **Severity** | 🟡 Medium |
| **File** | Referenced in `api.ts` line 7: `import { useDemoStore } from "@/stores/use-demo-store"` |
| **What** | Zustand store holding `isDemoMode` flag that gates the entire mock layer. |
| **Impact** | If `isDemoMode` is accidentally set to true in production, all API calls return fake data. |
| **Replacement** | If demo mode removed, delete the store. If kept, ensure `isDemoMode` cannot be set in production builds (build-time flag or environment check). |

---

## 4. Phased Task Plan

Prefix: `MOCK-FIX-{phase}-{nn}`

### Phase 1 — Critical: Demo Mock Layer Decision (Sprint 1)

| Task ID | Task | Affected IDs | Validation Criteria | Status |
|---------|------|-------------|---------------------|--------|
| **MOCK-FIX-1-01** | **Decision:** Keep or remove demo mode. If keeping: scope watermark + environment guard. If removing: proceed to MOCK-FIX-1-02. | MOCK-A-01, A-02, A-03, E-01, E-02 | Decision documented and approved by product owner. | Pending |
| **MOCK-FIX-1-02** | **If removing:** Delete `demo-api-mock.ts`, `demo-ciso-session.ts`, `DemoModal.tsx`, `use-demo-store.ts`. Remove imports and interceptor block from `api.ts` (lines 2–6, 7, 31–43). | MOCK-A-01, A-02, A-03, E-01, E-02 | `rg "demo-api-mock\|demo-ciso-session\|useDemoStore\|isDemoMode" frontend/` returns 0 results. App builds cleanly. No runtime `isDemoModeActive` check in network path. | Pending |
| **MOCK-FIX-1-03** | **If keeping:** Add `process.env.NEXT_PUBLIC_DEMO_ENABLED !== "true"` guard to `isDemoModeActive()`. Add persistent "DEMO MODE" banner to portal shell. Reset mock state on every demo session init. | MOCK-A-01, A-02, A-03 | Demo mode only activatable when env var is set. Banner visible on every page. Production build with no env var → demo mode impossible. | Pending |

**Exit Gate:** Demo mock layer either removed or production-guarded. No silent API interception possible in production builds.

---

### Phase 2 — Medium: Hardcoded Identity Removal (Sprint 2)

| Task ID | Task | Affected IDs | Validation Criteria | Status |
|---------|------|-------------|---------------------|--------|
| **MOCK-FIX-2-01** | Create `frontend/lib/auth-utils.ts` with `isCISOUser(user): boolean` checking `user.role === "CISO"` only. Export and use across all 8 files. | MOCK-D-01 | `rg "ciso@gtbank" frontend/` returns 0 results. `isCISOUser` used in all 8 files. Non-CISO-role users cannot access CISO actions regardless of email. | Pending |
| **MOCK-FIX-2-02** | Replace `"Ismail Muhammad"` fallback in `improvement-tab.tsx:341` with `t("portal.improvement.unknownApprover")` or `"System"`. | MOCK-B-01 | `rg "Ismail Muhammad" frontend/` returns 0 results. Change requests approved by nameless users show localized fallback, not a specific person's name. | Pending |
| **MOCK-FIX-2-03** | Replace all 4 `"Demo CISO"` fallbacks in `clause-7.tsx` with `user?.name \|\| user?.email \|\| t("portal.clause7.unknownUser")`. | MOCK-B-02 | `rg '"Demo CISO"' frontend/components/` returns 0 results (excluding `demo-ciso-session.ts` if demo mode kept). Competence records never contain "Demo CISO" when user has a name. | Pending |
| **MOCK-FIX-2-04** | Add profile completeness warning: if `user.name` is falsy, show a toast/banner on portal load prompting user to update their profile. | MOCK-B-02 (prevention) | Users with empty `name` field see a non-blocking prompt on portal entry. | Pending |

**Exit Gate:** No hardcoded names or emails in production compliance records. RBAC works for all institutions.

---

### Phase 3 — Low: Empty-State Tab Enhancement (Backlog)

| Task ID | Task | Affected IDs | Validation Criteria | Status |
|---------|------|-------------|---------------------|--------|
| **MOCK-FIX-3-01** | Wire `ActivitiesTab` to `monitoringService.getActivities()`. Show real monitoring/evaluation activities in a table. Keep empty state when no data. | MOCK-C-01 | Activities tab shows real data from monitoring service. Empty state when no activities exist. No mock generators. | Pending |
| **MOCK-FIX-3-02** | *(Optional)* Consider merging Activities tab into ISMS Calendar tab to avoid duplicate data display. | MOCK-C-01 | If merged: Activities sidebar entry redirects to Calendar tab. If kept separate: both show consistent data from same source. | Pending |

**Exit Gate:** All tabs either show real data or an honest empty state with a clear pathway to populate.

---

## 5. Task ID Quick-Reference Index

| ID | Phase | Category | One-line | Severity |
|----|-------|----------|----------|----------|
| MOCK-FIX-1-01 | 1 | A | Demo mode keep/remove decision | 🔴 Critical |
| MOCK-FIX-1-02 | 1 | A | Delete demo mock layer (if removing) | 🔴 Critical |
| MOCK-FIX-1-03 | 1 | A | Guard demo mode (if keeping) | 🔴 Critical |
| MOCK-FIX-2-01 | 2 | D | Extract `isCISOUser()`, remove `ciso@gtbank.com` | 🟡 Medium |
| MOCK-FIX-2-02 | 2 | B | Remove `"Ismail Muhammad"` fallback | 🟡 Medium |
| MOCK-FIX-2-03 | 2 | E | Remove `"Demo CISO"` fallbacks in clause-7 | 🟡 Medium |
| MOCK-FIX-2-04 | 2 | B | Profile completeness warning | 🟡 Medium |
| MOCK-FIX-3-01 | 3 | C | Wire Activities tab to real API | 🟢 Low |
| MOCK-FIX-3-02 | 3 | C | Consider Activities/Calendar merge | 🟢 Low |

**Total tasks:** 9 (3 Critical, 4 Medium, 2 Low).

---

## 6. File Map

### Files to Delete (if demo mode removed — Phase 1)

| File | Size | Role |
|------|------|------|
| `frontend/lib/demo-api-mock.ts` | 707 lines | In-memory API stubs |
| `frontend/lib/demo-ciso-session.ts` | 34 lines | Fake auth bootstrap |
| `frontend/components/public/DemoModal.tsx` | ~TBD | Demo entry point |
| `frontend/stores/use-demo-store.ts` | ~TBD | Demo mode flag |

### Files to Modify (Phase 1)

| File | Change |
|------|--------|
| `frontend/services/api.ts` | Remove lines 2–6 (imports), line 7 (`useDemoStore`), lines 31–43 (interceptor block) |

### Files to Modify (Phase 2)

| File | Change |
|------|--------|
| **[NEW]** `frontend/lib/auth-utils.ts` | New `isCISOUser(user)` utility |
| `frontend/components/institution/compliance-portal/tabs/controls-tab.tsx` | Use `isCISOUser()` |
| `frontend/components/institution/compliance-portal/tabs/incidents-tab.tsx` | Use `isCISOUser()` |
| `frontend/components/institution/compliance-portal/tabs/vendors-tab.tsx` | Use `isCISOUser()` |
| `frontend/components/institution/compliance-portal/tabs/improvement-tab.tsx` | Use `isCISOUser()` + remove `"Ismail Muhammad"` |
| `frontend/components/institution/compliance-portal/tabs/clause-tab/clause-4.tsx` | Use `isCISOUser()` |
| `frontend/components/institution/compliance-portal/tabs/clause-tab/clause-5.tsx` | Use `isCISOUser()` |
| `frontend/components/institution/compliance-portal/tabs/clause-tab/clause-shared.tsx` | Use `isCISOUser()` |
| `frontend/components/institution/compliance-portal/tabs/clause6-components/likelihood-impact.tsx` | Use `isCISOUser()` |
| `frontend/components/institution/compliance-portal/tabs/clause-tab/clause-7.tsx` | Remove 4× `"Demo CISO"` fallbacks |

### Files to Modify (Phase 3)

| File | Change |
|------|--------|
| `frontend/components/institution/compliance-portal/tabs/activities-tab.tsx` | Wire to `monitoringService.getActivities()` |

---

## 7. Verification Plan

### Automated Checks

```bash
# Phase 1 — Demo mode removed
rg "demo-api-mock|demo-ciso-session|useDemoStore|isDemoMode|isDemoModeActive" frontend/ --type ts --type tsx
# Expected: 0 results (or only in demo-guard code if keeping)

# Phase 2 — Hardcoded identities
rg "ciso@gtbank" frontend/ --type ts --type tsx
rg "Ismail Muhammad" frontend/ --type ts --type tsx
rg '"Demo CISO"' frontend/components/ --type ts --type tsx
# Expected: 0 results each

# Build check
npm run build
# Expected: clean build, no import errors
```

### Manual Verification

| Check | Steps | Expected Result |
|-------|-------|-----------------|
| Demo mode removed | 1. Open app. 2. Check there is no demo entry point. 3. Verify `api.ts` sends all requests to real backend. | No demo mode accessible. All API calls hit NestJS. |
| CISO RBAC | 1. Log in as a CISO user with email ≠ `ciso@gtbank.com`. 2. Navigate to Controls, Incidents, Vendors, Improvement tabs. 3. Verify CISO actions (approve/reject) are available. | CISO actions available based on `role`, not email. |
| Approver name | 1. Approve a change request as a user with no `name` field. 2. Check the approval record. | Shows localized "Unknown Approver" or user email, never "Ismail Muhammad". |
| Clause 7 records | 1. Create competence/awareness records as a user with empty `name`. 2. Inspect saved records. | Records show user email or localized fallback, never "Demo CISO". |
| Activities tab | 1. Create monitoring activities via Clause 9.1 tab. 2. Navigate to Activities tab. | Real activities displayed. Empty state when none exist. |

---

## 8. Relationship to Existing Redesign Tasks

This document complements [ISO27001_Portal_Redesign_Tasks.md](./ISO27001_Portal_Redesign_Tasks.md) which completed Phases 0–5 (Annex A scoring, SoA ownership, Overview consistency, backend hardening, secondary tab mock cleanup, QA).

**What was already fixed by the redesign:**
- ✅ `generateRisks()`, `generateActivities()`, `generateAudits()`, `generateTrendData()` — removed
- ✅ Overview hardcoded `71%` and fake trend series — replaced with derived scores
- ✅ Risk tab wired to `/risk` API
- ✅ SoA Export wired (CSV)
- ✅ Sidebar typos fixed (`Awarness` → `Awareness`, `Intend Audit` → `Internal Audit`)
- ✅ Annex A scoring formula (30/70 model) implemented end-to-end
- ✅ Backend `compositeScore` fix

**What this document addresses (not covered by redesign):**
- ❌ Demo API mock layer (systemic, cross-cutting)
- ❌ Hardcoded `ciso@gtbank.com` RBAC gate
- ❌ `"Ismail Muhammad"` / `"Demo CISO"` identity fallbacks
- ❌ Activities tab still empty-state-only

---

## Status Legend

| Status | Meaning |
|--------|---------|
| **Pending** | Not started |
| **In Progress** | Actively being implemented |
| **Done** | Validation criteria met |
| **Blocked** | Waiting on decision (e.g., MOCK-FIX-1-01 demo mode decision) |

Update Status columns in §4 as work completes.
