# ISO/IEC 27001:2022 ISMS Compliance Platform

[![Standard](https://img.shields.io/badge/Standard-ISO%2FIEC%2027001%3A2022-blue.svg)](#overview)
[![Framework](https://img.shields.io/badge/Scope-Annex%20A%20%2B%20Clauses%204--10-green.svg)](#architecture--document-index)
[![Status](https://img.shields.io/badge/Status-Production%20Hardened-emerald.svg)](#annex-a-scoring-model)

This directory contains the engineering blueprints, Statement of Applicability (SoA) lifecycle architecture, mock removal roadmap, and QA acceptance matrices for the **ISO/IEC 27001:2022 Information Security Management System (ISMS)** enterprise portal.

---

## Overview

The ISO 27001 Compliance Portal provides continuous, verifiable audit readiness for enterprise organizations. It automates control gap identification, evidence collection workflows, Statement of Applicability (SoA) governance, and dynamic Annex A compliance scoring.

```
iso27001/
├── README.md                                 # This index and architectural overview
├── iso27001-portal-redesign-tasks.md         # Portal redesign, Annex A scoring & SoA architecture
├── iso27001-mock-removal-implementation.md   # Production hardening: replacing stubs with live DB/API
└── iso27001-phase5-qa-acceptance.md          # End-to-end QA validation script & test run evidence
```

---

## Document Index

| Document | Category | Scope / Key Focus | Status |
| :--- | :--- | :--- | :--- |
| [iso27001-portal-redesign-tasks.md](./iso27001-portal-redesign-tasks.md) | **System Architecture** | Annex A 30/70 scoring engine, SoA ownership workflows, Clause 4–10 navigation, live audit trails | Production Completed |
| [iso27001-mock-removal-implementation.md](./iso27001-mock-removal-implementation.md) | **Production Hardening** | Cataloging & replacing UI mock state, fake trend lines, and stubs with authenticated API integrations | Production Completed |
| [iso27001-phase5-qa-acceptance.md](./iso27001-phase5-qa-acceptance.md) | **QA & Acceptance** | Control A.5.1 acceptance tests, 30/70 math validation (Effective + 5/5 = 100%, 4/5 = 86%), sign-off | Certified |

---

## Annex A Scoring Model (30/70 Hybrid Formula)

The portal utilizes a calibrated 30/70 formula to determine control effectiveness and compliance posture without bias:

```mermaid
graph LR
    subgraph "Control Evaluation"
        Status[Implementation Status<br/>Implemented / In Progress / Not Implemented]
        Checklist[Audit Checklist Items<br/>$C_{\text{completed}} / C_{\text{total}}$]
    end

    Status -->|Weight: 30%| CompStatus[Base Status Score]
    Checklist -->|Weight: 70%| CompChecklist[Evidence Checklist Score]

    CompStatus & CompChecklist --> ScoreEngine[Annex A 30/70 Scoring Engine]
    ScoreEngine --> Overall[Composite Domain & Annex A Score: 0% - 100%]
```

- **Base Status Weight (30%)**: Implemented ($100\%$), Partially Implemented ($50\%$), Not Implemented ($0\%$).
- **Checklist Verification Weight (70%)**: Fraction of completed audit verification items ($\frac{N_{\text{passed}}}{N_{\text{total}}} \times 100\%$).
- **Overall Control Score**: 
  $$\text{Score} = (0.30 \times \text{StatusScore}) + (0.70 \times \text{ChecklistScore})$$

---

## Key Platform Capabilities

1. **Statement of Applicability (SoA) Governance**: Built-in justification logging for excluded controls (with mandatory regulatory and operational justification before approval).
2. **Clause 4–10 Core ISMS Integration**: Real-time linking between management clauses (Context, Leadership, Planning, Support, Operation, Evaluation, Improvement) and Annex A technical controls.
3. **Automated Evidence Linking**: Secure attachment of audit artifacts, pentest reports, and policy signatures directly to relevant Annex A controls.
