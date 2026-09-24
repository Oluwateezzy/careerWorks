# ISO 27001 Annex A — Phase 5 QA / Acceptance Script

**Portal:** `/institution/dashboard?portal=ISO27001&tab=controls`  
**Primary control under test:** **A.5.1 Policies for information security**  
**Spec:** [ISO27001_Portal_Redesign_Tasks.md](./ISO27001_Portal_Redesign_Tasks.md) §1.1–§1.2  
**Date prepared:** 2026-07-24

---

## 0. Prerequisites

- [ ] Backend running (`npm run start:dev` in `backend/`)
- [ ] Frontend running (`npm run dev` in `frontend/`)
- [ ] Logged in as institution user with an ISO27001 assessment
- [ ] Open portal: `?portal=ISO27001&tab=controls`

---

## 1. Automated verification (pre-UAT) — executed 2026-07-24

| Check | Command / method | Result |
|-------|------------------|--------|
| FE Annex A scorer §1.2 examples | `cd frontend && npm test -- --testPathPatterns=annex-a-scoring` | **PASS** (13 tests) |
| BE Annex A scorer + normalize + composite | `cd backend && npm run test:iso27001` | **PASS** (15 tests) |
| CSAT scoring regression suite | `cd backend && npm run test:csat` | **PASS** (see terminal run) |
| No leftover mock generators in portal | `rg "generate(Risks\|Activities\|Audits\|TrendData)" frontend/.../compliance-portal` | **clean** |
| Sidebar typos fixed | `rg "Awarness\|Intend Audit"` | **clean** |
| Controls SoA justification read-only | Code: `readOnly` + placeholder “Not editable — sourced from the SoA” | **Present** |
| Portal still mounts CSAT + Clause tabs | `compliance-portal.tsx` branches `CBN_CSAT` / `clause` | **Present** |

Score examples locked by unit tests:

| Scenario | Expected score | Expected status |
|----------|----------------|-----------------|
| Effective + 5/5 checklist | 100 | Implemented |
| Effective + 4/5 | 86 | In Progress |
| Effective + 0/5 | 30 | Not Implemented |
| Almost Effective + 4/8 | 50 | In Progress |
| unset + 5/5 | 70 | In Progress |
| Not Applicable (SoA) | null | Not Applicable |
| 5 items weight each | 14% | — |

---

## 2. Manual UAT — A.5.1 assessment flow (ISO-P5-01)

Tester: _______________  Date: _______________

### 2.1 Open & layout

| # | Step | Expected | Pass? |
|---|------|----------|-------|
| 1 | Expand **A.5.1** | Panel shows gauge, SoA justification (disabled), notes, effectiveness (30%), checklist (70%) | ☐ |
| 2 | Confirm no manual status pills (Implemented / In Progress / … as clickable selectors) for Applicable controls | Status is badge/derived only | ☐ |
| 3 | Justification field | Disabled; placeholder mentions SoA | ☐ |

### 2.2 Score examples (live UI)

| # | Step | Expected gauge / status | Pass? |
|---|------|-------------------------|-------|
| 4 | Set effectiveness **Effective**; check all catalogue items (usually 3 for A.5.1 — note weight = 70/n) | Score = 30 + 70 = **100%**, status **Implemented** | ☐ |
| 5 | Uncheck one item (n=3 → each ≈ 23.33%; 2/3 checked → ≈ 46.67 + 30 ≈ **77%**) | Status **In Progress** (31–89) | ☐ |
| 6 | For exact §1.2 “5 items” check: Add items until 5 exist; Effective + 4 checked | **86%**, **In Progress** | ☐ |
| 7 | Effective + 0 checked (5 items) | **30%**, **Not Implemented** | ☐ |
| 8 | Hard refresh browser | Effectiveness, checklist, score, status restored | ☐ |

### 2.3 Checklist CRUD

| # | Step | Expected | Pass? |
|---|------|----------|-------|
| 9 | Add checklist item | New row appears; weight hint updates | ☐ |
| 10 | Edit item text | Text persists after refresh | ☐ |
| 11 | Try delete last remaining item | Blocked / toast; at least one remains | ☐ |
| 12 | Delete a non-last item | Removed; score recalculates | ☐ |

### 2.4 SoA ownership

| # | Step | Expected | Pass? |
|---|------|----------|-------|
| 13 | Open **SOA** tab; set A.5.1 justification text; wait ~2s autosave | Saves without error | ☐ |
| 14 | Return to **Annex-A Controls**; expand A.5.1 | Justification shows SoA text; still read-only | ☐ |
| 15 | On SOA, set Applicable → **No** for A.5.1 | Status **Not Applicable**; score **—**/null | ☐ |
| 16 | Controls tab A.5.1 | Scoring inputs hidden/disabled; N/A message | ☐ |
| 17 | SOA → Applicable **Yes** again | Scoring UI returns; can re-assess | ☐ |
| 18 | Click **Export SoA** | CSV downloads; includes A.5.1 row matching on-screen applicable/status/score/justification | ☐ |

### 2.5 AI suggestion (advisory)

| # | Step | Expected | Pass? |
|---|------|----------|-------|
| 19 | Attach evidence + AI Verify (or use existing) | Feedback shown with 0–4 score | ☐ |
| 20 | Accept Suggestion | Sets **effectiveness** only; status still derived from formula | ☐ |

### 2.6 Overview honesty

| # | Step | Expected | Pass? |
|---|------|----------|-------|
| 21 | Fresh / sparsely scored assessment → Overview | No fake **71%**; no fabricated Dec 2023–May trend series | ☐ |
| 22 | After scoring several controls | Annex theme % / status counts reflect scored data | ☐ |

---

## 3. Regression — Clauses + CSAT (ISO-P5-02)

### 3.1 Clauses tab

| # | Step | Expected | Pass? |
|---|------|----------|-------|
| 23 | `?portal=ISO27001&tab=clause` opens | Clause 4–10 overview loads | ☐ |
| 24 | Open **Clause 5.2** (or 5.1); edit a field; wait autosave; hard refresh | Value restored from `ISO27001-CLAUSE-*` / notes | ☐ |
| 25 | Open **Clause 6.1** risk UI; confirm still interactive | No crash; existing risk/treatment flows load | ☐ |

### 3.2 CBN CSAT portal

| # | Step | Expected | Pass? |
|---|------|----------|-------|
| 26 | `?portal=CBN_CSAT` (or open CSAT from dashboard) | Workbook sidebar (Institution Details, Inherent Risk, Maturity, …) loads | ☐ |
| 27 | Open **Inherent Risk** or **Maturity Tool Input**; confirm data loads from API | No blank crash; CSAT tabs render | ☐ |
| 28 | Backend `npm run test:csat` | Suite passes (automated) | ☑ automated |

---

## 4. Sign-off

| Role | Name | Date | Result |
|------|------|------|--------|
| Implementer (automated pre-checks) | Agent | 2026-07-24 | Automated §1 + §3.2.28 **PASS** |
| Manual UAT tester | | | ☐ Pass / ☐ Fail (notes below) |
| Product / acceptance | | | ☐ Accepted |

**Notes / defects found:**

```
(add findings here)
```

---

## 5. Related files

- `frontend/lib/iso27001/annex-a-scoring.ts`
- `frontend/components/institution/compliance-portal/tabs/controls-tab.tsx`
- `frontend/components/institution/compliance-portal/tabs/soa-tab.tsx`
- `frontend/components/institution/compliance-portal/tabs/overview-tab.tsx`
- `backend/src/iso27001/annex-a-scoring.ts`
- `backend/src/iso27001/domain-data.util.ts`
