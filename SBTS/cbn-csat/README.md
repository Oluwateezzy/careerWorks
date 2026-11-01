# Central Bank of Nigeria — Cybersecurity Self-Assessment Tool (CBN CSAT)

[![Regulatory Body](https://img.shields.io/badge/Regulator-Central%20Bank%20of%20Nigeria%20(CBN)-green.svg)](#statutory-foundation)
[![Standard](https://img.shields.io/badge/Standard-FFIEC%20CAT%20Adapted-blue.svg)](#scoring-engine-architecture)
[![Tech Stack](https://img.shields.io/badge/Stack-NestJS%20%7C%20Prisma%20%7C%20PostgreSQL%20%7C%20Next.js-navy.svg)](#implementation-roadmap)

This directory houses the complete engineering architecture, statutory calculation models, and NestJS/Prisma migration roadmap for the **Central Bank of Nigeria (CBN) Cybersecurity Self-Assessment Tool (CSAT)** rules engine.

---

## Executive Overview

The CBN CSAT is a statutory cybersecurity assessment workbook mandated for all Deposit Money Banks (DMBs), Payment Service Providers (PSPs), and Other Financial Institutions (OFIs) in Nigeria. 

The system transitions traditional, error-prone macro spreadsheets into a deterministic, high-throughput cloud compliance platform that enforces multi-tenant institutional isolation, cryptographic audit logging, and automated score computation.

```
cbn-csat/
├── README.md                               # This documentation index & architecture overview
├── cbn-csat-implementation-plan.md         # Rules engine specification & scoring formulas
└── cbn-csat-backend-migration-tasks.md     # Production NestJS + Prisma backend migration plan
```

---

## Document Index

| Document | Category | Scope / Key Focus | Status |
| :--- | :--- | :--- | :--- |
| [cbn-csat-implementation-plan.md](./cbn-csat-implementation-plan.md) | **Rules Engine Spec** | Full mathematical formulas, 102 Inherent Risk attributes, 504 Maturity statements, risk-to-maturity alignment matrix | Production Spec |
| [cbn-csat-backend-migration-tasks.md](./cbn-csat-backend-migration-tasks.md) | **Backend Migration** | Phased NestJS & Prisma ORM migration roadmap (P0–P6), multi-tenancy scoping (`institutionId`), catalog seeding | Completed / Reference |

---

## Scoring Engine Architecture

The CBN CSAT engine evaluates risk posture using two primary mathematical models:

```mermaid
graph TD
    subgraph "Inherent Risk Profile (IRP)"
        IR1[Technologies & Connections<br/>Category 1]
        IR2[Delivery Channels<br/>Category 2]
        IR3[Online/Mobile Services<br/>Category 3]
        IR4[Organizational Characteristics<br/>Category 4]
        IR5[External Threats<br/>Category 5]
        IR1 & IR2 & IR3 & IR4 & IR5 -->|Weighted Average| IRAvg[102 Attributes Evaluated]
        IRAvg -->|Threshold Bands| Band[Inherent Risk Band: Low to Extremely High]
    end

    subgraph "Cybersecurity Maturity (CSM)"
        D1[Domain 1: Cyber Risk Management]
        D2[Domain 2: Threat Intelligence]
        D3[Domain 3: Cybersecurity Controls]
        D4[Domain 4: External Dependencies]
        D5[Domain 5: Incident Response]
        D1 & D2 & D3 & D4 & D5 -->|504 Declarative Statements| MLevel[Levels 1–5]
        MLevel -->|100% Gate Invariant| Gate[Achieved Maturity Level]
    end

    Band --> Align[Risk-to-Maturity Alignment Matrix]
    Gate --> Align
    Align --> Gap[Compliance Gap & Board Audit Report]
```

### 1. Inherent Risk Level Thresholds
- **Low**: Score $< 1.5$
- **Moderate**: $1.5 \le \text{Score} < 2.5$
- **Above Average**: $2.5 \le \text{Score} < 3.5$
- **High**: $3.5 \le \text{Score} < 4.5$
- **Extremely High**: $\text{Score} \ge 4.5$

### 2. Maturity Level Gate Invariant
Maturity levels are sequential: **Baseline (Level 1) $\rightarrow$ Evolving (Level 2) $\rightarrow$ Intermediate (Level 3) $\rightarrow$ Advanced (Level 4) $\rightarrow$ Innovative (Level 5)**. A financial institution can only be certified at level $N$ if and only if **100% of declarative statements for levels $1$ through $N$ are satisfied (Yes)**.

---

## Core Security & Tenancy Rules

- **Strict Multi-Tenancy**: All CSAT queries and mutations are isolated by `req.user.institutionId`. Cross-institution assessment access returns `403 Forbidden`.
- **Immutable Assessment History**: Completed assessments are versioned and cryptographically signed for regulatory inspection.
