# Enterprise Security, Regulatory Governance & Intelligence (`/SBTS`)

[![Portfolio Category](https://img.shields.io/badge/Domain-Cybersecurity%20%26%20Governance-blue.svg)](#overview)
[![Frameworks](https://img.shields.io/badge/Frameworks-CBN%20CSAT%20%7C%20ISO%2027001%20%7C%20FFIEC-green.svg)](#subsystem-directory-structure)
[![Tech Stack](https://img.shields.io/badge/Stack-NestJS%20%7C%20Prisma%20%7C%20PostgreSQL%20%7C%20Next.js-navy.svg)](#architecture--security-matrices)

This domain houses enterprise compliance automation platforms, statutory cybersecurity rules engines, and security intelligence systems engineered for banking regulators, enterprise organizations, and democratic governance bodies.

---

## Overview

The `/SBTS` domain encompasses three flagship initiatives:
1. **CBN CSAT**: Central Bank of Nigeria Cybersecurity Self-Assessment Tool rules engine (FFIEC CAT foundation).
2. **ISO 27001**: Enterprise ISMS compliance platform with dynamic Annex A 30/70 scoring and automated SoA governance.
3. **ElectionsSentinel360 (ES360)**: Real-time election security, statutory over-voting fraud detection, and OSINT stream ingestion.

---

## Subsystem Directory Structure

```
SBTS/
├── README.md                                 # Master domain overview & index (this file)
│
├── cbn-csat/                                 # Central Bank of Nigeria CSAT Rules Engine
│   ├── README.md                             # Engine architecture, threshold bands & math
│   ├── cbn-csat-implementation-plan.md       # Rules engine specification & scoring formulas
│   └── cbn-csat-backend-migration-tasks.md   # NestJS + Prisma backend migration roadmap
│
├── iso27001/                                 # ISO/IEC 27001:2022 ISMS Compliance Portal
│   ├── README.md                             # ISMS lifecycle, 30/70 formula & SoA governance
│   ├── iso27001-portal-redesign-tasks.md     # Portal redesign, Annex A scoring & SoA architecture
│   ├── iso27001-mock-removal-implementation.md # Hardening: replacing mock state with live APIs
│   └── iso27001-phase5-qa-acceptance.md      # Control A.5.1 acceptance test script & evidence
│
├── elections-sentinel/                       # ElectionsSentinel360 (ES360) Security Platform
│   ├── README.md                             # Platform architecture, statutory invariants & OSINT
│   ├── 01-requirements-and-core-entities.md  # Statutory foundation (Electoral Act 2026), ERD & RBAC
│   ├── 02-architecture-and-data-flow.md      # Streaming ingestion, geospatial index & state machine
│   ├── 03-implementation-plan.md             # Phased milestone delivery & situation room UI
│   ├── 04-security-architecture-review.md    # STRIDE threat modeling & anti-tamper controls
│   ├── 05-security-signoff-and-pentest-readiness.md # Vulnerability remediation & pentest sign-off
│   └── elections-sentinel360-live-data-sources-and-osint-strategy.pdf # OSINT collection strategy
│
└── audit-reports/                            # Penetration Testing & Endpoint Security Audits
    ├── README.md                             # Governance & audit summary
    └── backend-endpoints-audit-report.pdf    # REST API endpoints security audit report
```

---

## Master Document Matrix

| Initiative | Document | Category | Scope / Focus |
| :--- | :--- | :--- | :--- |
| **CBN CSAT** | [cbn-csat/README.md](./cbn-csat/README.md) | **Overview** | Inherent risk bands, 100% maturity gates, multi-tenant isolation |
| **CBN CSAT** | [cbn-csat/cbn-csat-implementation-plan.md](./cbn-csat/cbn-csat-implementation-plan.md) | **Rules Engine** | 102 Inherent Risk attributes, 504 Declarative statements, formulas |
| **CBN CSAT** | [cbn-csat/cbn-csat-backend-migration-tasks.md](./cbn-csat/cbn-csat-backend-migration-tasks.md) | **Backend Plan** | NestJS & Prisma migration roadmap, schema seeding, multi-tenancy |
| **ISO 27001** | [iso27001/README.md](./iso27001/README.md) | **Overview** | Annex A controls, 30/70 scoring model, Statement of Applicability |
| **ISO 27001** | [iso27001/iso27001-portal-redesign-tasks.md](./iso27001/iso27001-portal-redesign-tasks.md) | **Architecture** | Annex A scoring engine, SoA ownership workflows, Clause 4–10 linking |
| **ISO 27001** | [iso27001/iso27001-mock-removal-implementation.md](./iso27001/iso27001-mock-removal-implementation.md) | **Hardening** | Replacing mock generators and stubs with authenticated API calls |
| **ISO 27001** | [iso27001/iso27001-phase5-qa-acceptance.md](./iso27001/iso27001-phase5-qa-acceptance.md) | **QA Validation** | Control A.5.1 test script and mathematical scoring validation |
| **ES360** | [elections-sentinel/README.md](./elections-sentinel/README.md) | **Overview** | Situation room design, statutory over-voting fraud detection |
| **ES360** | [elections-sentinel/01-requirements-and-core-entities.md](./elections-sentinel/01-requirements-and-core-entities.md) | **Requirements** | Electoral Act 2026 §51(2) legal mapping, core entity model, RBAC |
| **ES360** | [elections-sentinel/02-architecture-and-data-flow.md](./elections-sentinel/02-architecture-and-data-flow.md) | **Architecture** | High-throughput observer ingestion, geospatial mapping, stream pipeline |
| **ES360** | [elections-sentinel/03-implementation-plan.md](./elections-sentinel/03-implementation-plan.md) | **Roadmap** | Multi-phase development roadmap and acceptance milestones |
| **ES360** | [elections-sentinel/04-security-architecture-review.md](./elections-sentinel/04-security-architecture-review.md) | **Threat Model** | STRIDE threat analysis, anti-tamper evidence verification |
| **ES360** | [elections-sentinel/05-security-signoff-and-pentest-readiness.md](./elections-sentinel/05-security-signoff-and-pentest-readiness.md) | **Pentest Sign-off**| Pentest readiness checklist and vulnerability remediation matrix |
| **ES360** | [elections-sentinel/elections-sentinel360-live-data-sources-and-osint-strategy.pdf](./elections-sentinel/elections-sentinel360-live-data-sources-and-osint-strategy.pdf) | **OSINT** | Real-time social media & news intelligence collection pipeline |
| **Audits** | [audit-reports/README.md](./audit-reports/README.md) | **Audits** | Security vulnerability governance & audit remediation |
| **Audits** | [audit-reports/backend-endpoints-audit-report.pdf](./audit-reports/backend-endpoints-audit-report.pdf) | **Report** | API endpoints security audit and penetration test report |

---

## Architectural Principles Across `/SBTS`

1. **Statutory Integrity**: Core business rules directly reflect published regulatory statutes (e.g., CBN circulars, ISO/IEC 27001:2022 standards, and Nigeria Electoral Act 2026).
2. **Deterministic Calculations**: No black-box scoring. Inherent risk formulas, maturity gates, and Annex A percentages are 100% deterministic and auditable.
3. **Defense-in-Depth & Data Isolation**: Multi-tenant database design prevents data cross-contamination between banking institutions and observer groups.
