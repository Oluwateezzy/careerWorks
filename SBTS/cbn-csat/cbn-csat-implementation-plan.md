# CBN Cybersecurity Self-Assessment Tool (CSAT) — Implementation Plan
**Target stack:** existing NestJS + Prisma (API) and Next.js (frontend) codebase
**Source spec:** `2_Updated_CBN_Cybersecurity_Self-Assessment_Tool.xlsx` (the CBN workbook) + `CSAT_Parameters.xlsx` (scoring weights)

---

## 0. What the spreadsheets actually specify (read this before starting any task)

The workbook is not a form — it's a rules engine. Rebuilding it means replicating five linked computations:

1. **Institution & Approval** — `Details of Institution`, `Approval Page`, `Items to Submit`: static intake fields + a sign-off workflow (Preparer → Approver) + a fixed checklist of artifacts to bundle for submission to CBN.
2. **Inherent Risk Profile** — `Inherent Risk Profile Input` (102 rows) + `Inherent Risk - Explanation`: 5 categories (*Technologies & Connection Types, Delivery Channels, Online/Mobile Products & Technology Services, Organizational Characteristics, External Attacks*), each with weighted risk-attribute statements scored 1–5 (Low→Extremely High). Category score = `AVERAGE` of answered attributes; category rating = threshold bands (`<1.5 Low, <2.5 Moderate, <3.5 Above Average, <4.5 High, else Extremely High`); an aggregate inherent risk level rolls up from all categories.
3. **Maturity Assessment** — `Maturity Tool Input` (504 rows): 5 Domains → Assessment Factors → Components → 5 Maturity Levels (Baseline/Evolving/Intermediate/Advanced/Innovative), each level containing declarative statements answered `Yes / Yes [CC] / No / N/A`. `CSAT_Parameters.xlsx` supplies the **point weight of every Component × Maturity Level cell** — this is the scoring key, not just reference data.
4. **Maturity Rollup** — a domain's current maturity level is the **highest level at which all declarative statements are answered "Yes" or "Yes [CC]"** (standard FFIEC CAT logic), weighted per `CSAT_Parameters`. `Risk - Maturity Summary` cross-tabs the 5 domain maturity levels against the aggregate inherent risk level.
5. **Reference data** — `Threats` and `Vulnerabilities` (26 rows each) are static catalogs referenced/selected from, not computed.

⚠️ **Known defect to design around:** many formulas reference an external, missing workbook (`'[1]Table of Contents'!...`, `'[1]Maturity Results'!...`, `[1]ref!...` — 24 broken defined names). The live rollup/threshold logic for the summary dashboard was never fully self-contained in this file. Phase 0 includes explicitly reverse-engineering/re-deriving that logic (using the FFIEC CAT methodology the CBN framework is based on) rather than assuming the extracted formulas are complete.

---

## Phase 0 — Discovery & Domain Modeling
*No code. Output: a signed-off data dictionary everyone builds against.*

| Task ID | Task | Validation Criteria |
|---|---|---|
| **CSAT-P0-01** | Audit existing Prisma schema + NestJS module structure + Next.js routing to find any overlapping models (User, Institution, Org, Role) to avoid duplication. | Written audit doc lists every existing model/table that a CSAT entity could reuse or must not collide with. |
| **CSAT-P0-02** | Produce the canonical **Domain → Assessment Factor → Component → Maturity Level → Declarative Statement** hierarchy as structured JSON/CSV, extracted from `Maturity Tool Input`. | Row count in JSON = 504 minus header/blank rows; spot-check 10 random statements against source cells match exactly (text, FFIEC citation, maturity level). |
| **CSAT-P0-03** | Produce canonical **Inherent Risk Category → Risk Attribute Statement → 5 risk-level descriptions** structured JSON, extracted from `Inherent Risk Profile Input`. | 5 categories present; every attribute row has exactly 5 non-empty level descriptions (E–I columns). |
| **CSAT-P0-04** | Produce canonical **scoring weight matrix** from `CSAT_Parameters.xlsx` (Domain/Component × Baseline/Evolving/Intermediate/Advanced/Innovative points), and confirm every Component in P0-02 has a matching weight row. | Zero unmatched components between P0-02 output and CSAT_Parameters; `SUM` per row in JSON equals the sheet's `=SUM(E:I)` cached value. |
| **CSAT-P0-05** | Re-derive and document, in plain pseudocode, the **maturity-level rollup rule** and **inherent-risk aggregation rule** (since source formulas point to a missing external workbook). | Pseudocode reviewed/approved by whoever owns the CBN compliance relationship; includes worked example with sample answers producing an expected domain maturity level. |
| **CSAT-P0-06** | Extract `Threats` and `Vulnerabilities` reference lists to seed data JSON. | 26 rows each, matches source exactly. |

---

## Phase 1 — Data Model (Prisma Schema)
*Depends on Phase 0. Output: migration applied to a dev DB.*

| Task ID | Task | Validation Criteria |
|---|---|---|
| **CSAT-P1-01** | Model `Institution` (name, address, year of assessment, CISO details, reporting line) extending/linking to existing org/tenant model if one exists. | `npx prisma migrate dev` succeeds; one institution record can be created via Prisma Studio with all `Details of Institution` fields. |
| **CSAT-P1-02** | Model `Domain`, `AssessmentFactor`, `Component`, `MaturityLevel` (enum: Baseline/Evolving/Intermediate/Advanced/Innovative), `DeclarativeStatement` (with FFIEC citation text field) as normalized, seedable tables. | Seed script populated from P0-02 JSON inserts exactly matching row counts with no FK errors. |
| **CSAT-P1-03** | Model `ComponentMaturityWeight` (component + level → point value) sourced from `CSAT_Parameters`. | Seed matches P0-04 JSON 1:1; a query summing weights per component-row equals the sheet's row `TOTAL`. |
| **CSAT-P1-04** | Model `InherentRiskCategory` and `InherentRiskAttribute` (with 5 level-description text fields) from P0-03. | Seed matches source; 5 categories, correct attribute counts per category (14 / 4 / … as extracted). |
| **CSAT-P1-05** | Model assessment instance tables: `Assessment` (one per institution per year, status: Draft/Submitted/Approved), `MaturityResponse` (assessment + statement → Yes/Yes[CC]/No/N/A + comment + compensating-control flag), `InherentRiskResponse` (assessment + attribute → selected risk level 1–5 + comment). | Can create a full draft Assessment with responses via Prisma Studio without constraint violations. |
| **CSAT-P1-06** | Model `Threat` and `Vulnerability` reference tables + `AssessmentThreatSelection` / `AssessmentVulnerabilitySelection` join tables. | Seed = 26 rows each; join table enforces FK to a valid Assessment. |
| **CSAT-P1-07** | Model `ApprovalRecord` (preparer name/date, approver name/date, status) matching `Approval Page`. | An Assessment cannot transition to `Submitted` status without a completed ApprovalRecord (enforced at service layer, tested in P3). |

---

## Phase 2 — Reference Data Seeding
*Depends on Phase 1. Output: idempotent seed scripts in the repo.*

| Task ID | Task | Validation Criteria |
|---|---|---|
| **CSAT-P2-01** | Write Prisma seed script for Domains/Factors/Components/Statements (P0-02). | Re-running seed is idempotent (upsert, no duplicate rows on second run). |
| **CSAT-P2-02** | Write seed script for `ComponentMaturityWeight` (P0-04). | Idempotent; totals reconciled against CSAT_Parameters sheet as a unit test. |
| **CSAT-P2-03** | Write seed script for Inherent Risk categories/attributes (P0-03). | Idempotent; row counts match. |
| **CSAT-P2-04** | Write seed script for Threats/Vulnerabilities (P0-06). | 26 + 26 rows present after seed. |

---

## Phase 3 — Backend API (NestJS)
*Depends on Phase 2.*

| Task ID | Task | Validation Criteria |
|---|---|---|
| **CSAT-P3-01** | `InstitutionModule`: CRUD for institution profile + CISO/stakeholder contacts. | E2E test: create/update/get institution returns exact payload shape; auth-guarded (only institution's own users can edit). |
| **CSAT-P3-02** | `AssessmentModule`: create new yearly Assessment (Draft), list assessments per institution, fetch full assessment tree (domains→factors→components→statements pre-joined with any existing responses). | E2E test: new assessment auto-includes every seeded statement with `null` response; GET returns nested structure the frontend can render 1:1 against `Maturity Tool Input` layout. |
| **CSAT-P3-03** | `MaturityResponseController`: PATCH endpoint to set Yes/Yes[CC]/No/N/A + comment per statement, with optimistic bulk-save support (institutions fill many rows at once). | Unit test rejects invalid enum values; bulk PATCH of 504 rows completes and is reflected on re-fetch. |
| **CSAT-P3-04** | `InherentRiskController`: endpoint to set selected risk level (1–5) + comment per attribute. | Unit test rejects level outside 1–5; partial-completion state (`Incomplete`) surfaces via the scoring endpoint below. |
| **CSAT-P3-05** | `ScoringService`: implement category-level inherent risk averaging + threshold banding (reproducing `IF(COUNTA(...)<n,"Incomplete", IF(avg<1.5,"Low",...))` logic) and aggregate inherent risk rollup. | Unit tests with hand-computed fixtures (e.g., all attributes = "Low" → category="Low"; mixed values → matches manually-calculated average and band) plus an "incomplete" case when fewer than N attributes are answered. |
| **CSAT-P3-06** | `ScoringService`: implement per-domain maturity level rollup (highest level where all statements = Yes/Yes[CC]) weighted against `ComponentMaturityWeight`, per P0-05 pseudocode. | Unit tests replicate the worked example from CSAT-P0-05 exactly; a domain with one "No" at Intermediate level caps rollup at Evolving. |
| **CSAT-P3-07** | `SummaryController`: endpoint returning the Risk-Maturity Summary equivalent (5 domain current/target maturity levels × aggregate inherent risk level cross-tab). | Response shape matches what `Risk - Maturity Summary` sheet displays; verified against 2–3 manually-scored sample assessments. |
| **CSAT-P3-08** | `ApprovalModule`: submit-for-approval, approve, reject flow with role guards (Preparer vs Approver vs CBN reviewer if applicable). | E2E test: cannot mark `Submitted` without preparer+approver signatures recorded; state machine rejects invalid transitions (e.g., re-approving an already-approved assessment). |
| **CSAT-P3-09** | `ThreatVulnerabilityController`: CRUD for selecting/annotating Threats & Vulnerabilities per assessment. | E2E test round-trips selections correctly against seeded reference lists. |
| **CSAT-P3-10** | Authorization/tenancy pass: ensure every endpoint above scopes data by institution and enforces existing auth guards from the codebase. | Automated test: user from Institution A cannot read/write Institution B's assessment (403). |

---

## Phase 4 — Frontend (Next.js)
*Depends on Phase 3 endpoints being available (can start in parallel against a mocked API contract).*

| Task ID | Task | Validation Criteria |
|---|---|---|
| **CSAT-P4-01** | `Institution Details` page mirroring `Details of Institution` sheet fields. | Form validation matches required fields; saved data round-trips correctly on reload. |
| **CSAT-P4-02** | `Inherent Risk Profile` wizard: 5 collapsible category sections, each attribute row rendered as a radio/select of the 5 level descriptions (not just "1–5" — must show the actual descriptive text from P0-03, exactly like the sheet's dropdown-with-description UX). | Each of the 5 categories renders correct attribute count; selecting a level updates live category average/band shown inline (calls P3-05). |
| **CSAT-P4-03** | `Maturity Assessment` wizard: grouped by Domain → Factor → Component → Maturity Level, each declarative statement as Yes/Yes[CC]/No/N/A with comment box; compensating-control flag auto-derives when "Yes [CC]" chosen. | Renders all 504 statements grouped correctly; progress indicator matches `COUNTA` completion logic from source. |
| **CSAT-P4-04** | `Risk-Maturity Summary` dashboard: cross-tab table + current-vs-target maturity chart per domain (reuse Recharts or existing chart lib in codebase). | Visual output matches values from `CSAT-P3-07` for 2 test assessments at different completion states, including the "--" placeholder for incomplete inherent-risk total. |
| **CSAT-P4-05** | `Threats` & `Vulnerabilities` selection screens. | Selections persist and display correctly against the 26-item reference lists. |
| **CSAT-P4-06** | `Approval Page` UI: preparer/approver sign-off, submission checklist matching `Items to Submit` (8-item list) with per-item completion status. | Submit button disabled until checklist + both signatures present; matches P3-08 state machine. |
| **CSAT-P4-07** | Navigation shell binding all pages into a single assessment flow (Title → Institution → Inherent Risk → Maturity → Summary → Threats/Vulns → Approval), consistent with existing app layout/design system. | Manual UAT walkthrough of full assessment start-to-submit without dead ends or console errors. |

---

## Phase 5 — Export & Submission Package
*Depends on Phase 4.*

| Task ID | Task | Validation Criteria |
|---|---|---|
| **CSAT-P5-01** | Generate downloadable Excel export of a completed assessment reproducing the original workbook's sheet layout (for institutions/examiners expecting the familiar format). | Exported file opens cleanly, contains no `#REF!`/`#NAME?` errors, values match the DB-stored responses. |
| **CSAT-P5-02** | Generate a PDF/Word "submission bundle" per the `Items to Submit` checklist (Details of Institution, Approval Page, Risk-Maturity Summary, Inherent Risk Result, Charts, Threats, Vulnerabilities). | Bundle contains exactly the 7 listed sections in the specified order; spot-checked against a completed sample assessment. |
| **CSAT-P5-03** | (If required) email/submission integration to the CBN mailbox address referenced in `Items to Submit`. | Confirmed test send in a staging environment; failure states surfaced to the user. |

---

## Phase 6 — QA, Validation & Deployment
*Depends on all prior phases.*

| Task ID | Task | Validation Criteria |
|---|---|---|
| **CSAT-P6-01** | Regression-test the full scoring engine against 3–5 fully-answered sample assessments with hand-calculated expected outputs (inherent risk levels + domain maturity levels). | 100% match between engine output and hand calculations; any mismatch blocks release. |
| **CSAT-P6-02** | Load/perf test the Maturity wizard bulk-save (504 rows) and Summary endpoint. | P95 response time within existing app's SLA/benchmarks. |
| **CSAT-P6-03** | Security review: tenancy isolation, role-based access on approval flow, input validation on all response enums. | No cross-tenant data leakage in pen-test pass; all endpoints reject malformed enum/level values. |
| **CSAT-P6-04** | UAT sign-off with actual compliance/CISO stakeholder using the re-derived rollup logic (P0-05) as source of truth. | Written sign-off that outputs match expected CBN CSAT scoring behavior. |
| **CSAT-P6-05** | Deploy to staging → production per existing CI/CD pipeline; migrate + seed reference data in target environment. | Production smoke test: create institution, complete one assessment end-to-end, confirm summary and export both work. |

---

## Dependency Overview

```
P0 (Discovery) → P1 (Schema) → P2 (Seed data) → P3 (API) → P4 (Frontend) → P5 (Export) → P6 (QA/Deploy)
                                                     ↘_________________________________↗
                                          P4 can start in parallel against a mocked P3 contract
```

## Open items to confirm before Phase 1 starts
- Whether "Institution" in this system maps to an existing tenant/org model already in the codebase, or needs to be new.
- Who counts as "Approver" in the existing role system (mapping to Preparer/Approver in `Approval Page`).
- Whether the Excel/PDF export in Phase 5 is a hard requirement or a nice-to-have — it changes P5 scope significantly.
- Confirmation of the re-derived rollup logic in **CSAT-P0-05**, since the original file's own formulas can't be fully trusted (broken external links).
