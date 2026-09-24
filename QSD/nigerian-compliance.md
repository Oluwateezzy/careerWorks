# Nigerian Regulatory Compliance Guide — Vescrow

**Version**: 1.0  
**Last Updated**: June 2026  
**Disclaimer**: This document is for technical planning purposes only. It does not constitute legal advice. Consult a qualified Nigerian legal practitioner and/or the Central Bank of Nigeria for formal regulatory guidance before launching.

---

## Table of Contents

1. [Regulatory Landscape Overview](#1-regulatory-landscape-overview)
2. [Escrow Legal Framework](#2-escrow-legal-framework)
3. [CBN Licensing Requirements](#3-cbn-licensing-requirements)
4. [KYC/AML Compliance](#4-kycaml-compliance)
5. [Data Protection (NDPA 2023)](#5-data-protection-ndpa-2023)
6. [Consumer Protection](#6-consumer-protection)
7. [Payment Settlement & Withdrawal](#7-payment-settlement--withdrawal)
8. [Inspection Period & Buyer Rights](#8-inspection-period--buyer-rights)
9. [Record Retention Requirements](#9-record-retention-requirements)
10. [Technical Implementation Checklist](#10-technical-implementation-checklist)
11. [Regulatory Bodies & Contacts](#11-regulatory-bodies--contacts)

---

## 1. Regulatory Landscape Overview

Operating an escrow platform in Nigeria requires compliance with multiple regulatory frameworks. There is no single "Escrow Act" — instead, escrow services operate under a combination of:

| Framework | Governing Body | Relevance |
|-----------|---------------|-----------|
| Contract Law | Nigerian Courts | Escrow = valid "stakeholder agreement" |
| Trust Law | Nigerian Courts | Fiduciary obligations for fund holding |
| CBN Payment System Guidelines | Central Bank of Nigeria | Payment processing, fund holding |
| BOFIA 2020 | CBN | Banks and Other Financial Institutions Act |
| FCCPA 2018 | FCCPC | Federal Consumer Protection |
| NDPA 2023 | NDPC | Data protection and privacy |
| CBN AML/CFT Guidelines | CBN / NFIU | Anti-money laundering |
| Cybersecurity Guidelines | CBN | Financial sector cybersecurity |

---

## 2. Escrow Legal Framework

### 2.1 Legal Basis

Escrow arrangements are legally recognized in Nigeria through:

1. **Freedom of Contract**: Parties can freely enter into stakeholder/escrow agreements defining terms, conditions, and release triggers.

2. **Trust and Fiduciary Law**: The escrow agent (Vescrow) has fiduciary obligations to:
   - Hold funds in trust for both parties
   - Release funds only according to agreed terms
   - Act impartially in disputes
   - Maintain proper records

3. **Case Law Precedent**: Nigerian courts have upheld escrow arrangements as valid mechanisms for securing commercial transactions.

### 2.2 Key Legal Requirements

| Requirement | Implementation |
|------------|---------------|
| Written Agreement | Clear Terms of Service accepted by both parties at transaction creation |
| Defined Release Conditions | Explicit conditions (delivery confirmation, inspection period) in escrow contract |
| Dispute Resolution Mechanism | Built-in dispute system with evidence submission and admin arbitration |
| Fund Segregation | Escrowed funds held in dedicated bank accounts (via banking partner) |
| Transparency | Both parties can view transaction status, timeline, and terms at all times |

---

## 3. CBN Licensing Requirements

### 3.1 License Category

The most applicable CBN license for Vescrow is the **Payment Solution Service Provider (PSSP)** license.

| Aspect | Requirement |
|--------|-------------|
| **License Type** | PSSP (Payment Solution Service Provider) |
| **Minimum Capital** | ₦100,000,000 (shareholders' funds, unimpaired by losses) |
| **Escrow Deposit** | ₦100,000,000 refundable deposit into CBN PSP Share Capital Deposit Account |
| **Application Fee** | ₦100,000 (non-refundable) |
| **Licensing Fee** | ₦1,000,000 |
| **Process Duration** | 6–12 months typically |

### 3.2 Application Process

```
Step 1: PREPARATION
├── Register company with CAC
├── Draft Memorandum & Articles of Association
│   (restrict activities to permissible PSSP operations)
├── Appoint governance structure
│   ├── Chairman
│   ├── CEO/MD
│   └── ≥1 Independent Non-Executive Director
├── Appoint MLRO (Money Laundering Reporting Officer) + alternate
├── Prepare business plan with 5-year financial projections
└── Prepare IT, risk management, data protection, AML/CFT policies

Step 2: APPROVAL-IN-PRINCIPLE (AIP)
├── Submit application to Director, Payments System Management Dept, CBN
├── Include all governance, technical, and financial documentation
├── CBN reviews application
├── If approved: AIP valid for 6 months
│   (does NOT authorize commencement of operations)
└── Deposit ₦100M into CBN PSP Share Capital Deposit Account

Step 3: OPERATIONAL READINESS
├── Set up physical office
├── Fulfill technical requirements (PCI-DSS certification)
├── Establish banking partnership for fund holding
├── Implement AML/CFT systems
└── Notify CBN of readiness

Step 4: FINAL LICENSE
├── CBN conducts operational assessment
├── Final license issued
└── Escrow deposit refunded (with accrued interest)
```

### 3.3 Critical: Fund Holding Restriction

> ⚠️ **A PSSP license does NOT authorize holding customer funds directly.**

Vescrow must partner with a CBN-licensed commercial bank to hold escrowed funds:

| Arrangement | Details |
|------------|---------|
| **Banking Partner** | A licensed Nigerian commercial bank |
| **Fund Account** | Dedicated escrow/trust account at the partner bank |
| **Vescrow's Role** | Technology interface — manages the platform and transaction logic |
| **Bank's Role** | Holds and releases funds per Vescrow's instructions |
| **Regulatory Oversight** | Funds are subject to standard banking supervision |
| **Client Segregation** | Escrowed funds must be segregated from Vescrow's operational funds |

---

## 4. KYC/AML Compliance

### 4.1 CBN KYC Requirements (2025/2026)

#### Tiered KYC System

| Tier | Requirements | Max Transaction | Max Daily Balance |
|------|-------------|-----------------|-------------------|
| **Tier 1 (Basic)** | BVN or NIN | ₦50,000 | ₦300,000 |
| **Tier 2 (Standard)** | BVN + NIN + Government-issued ID | ₦500,000 | ₦5,000,000 |
| **Tier 3 (Enhanced)** | Tier 2 + Address verification + Selfie match | Unlimited | Unlimited |

#### BVN/NIN Verification Requirements

| Requirement | Details |
|------------|---------|
| **Electronic Retrieval** | Must verify BVN/NIN directly from NIBSS/NIMC databases |
| **No Manual Entry** | Manual profile creation followed by later BVN attachment is prohibited |
| **Real-Time Verification** | Verification must happen during onboarding, not after |
| **BVN Phone Updates** | Limited to once in a lifetime (May 2026 CBN directive) |

### 4.2 AML/CFT Requirements

#### CBN Circular BSD/DIR/PUB/LAB/019/002 (March 2026)

| Requirement | Implementation |
|------------|---------------|
| **Automated Transaction Monitoring** | Must implement automated (not manual) suspicious activity detection |
| **Behavioral Analysis** | Static threshold-only rules are insufficient; must support behavioral transaction analysis |
| **Real-Time Flagging** | AI-powered tools to flag or block suspicious transactions as they occur |
| **360° Customer View** | KYC/KYB data continuously integrated with customer risk profiles |
| **Watchlist Integration** | Monitor CBN BVN watchlist (transactions may be paused up to 24 hours) |

#### Reporting Obligations

| Report | Deadline | Recipient |
|--------|----------|-----------|
| Suspicious Transaction Report (STR) | Within 24 hours | NFIU |
| Currency Transaction Report (CTR) | For cash ≥ ₦5M (individual) / ₦10M (corporate) | NFIU |
| International Fund Transfer Report | Within 24 hours | NFIU |

#### Vescrow Implementation

```
Transaction Monitoring Pipeline:
├── Per-transaction checks
│   ├── Amount threshold: Flag if > ₦1,000,000
│   ├── Frequency: Flag if > 10 transactions/day
│   ├── BVN watchlist: Check against CBN watchlist
│   └── Velocity: Flag if cumulative daily amount > ₦5,000,000
│
├── Periodic analysis (daily batch)
│   ├── Unusual pattern detection
│   ├── Dormant account sudden activity
│   └── Round-amount transactions pattern
│
└── Actions
    ├── Flag for manual review
    ├── Temporary hold (max 24 hours)
    ├── Generate STR
    └── Freeze account (escalation)
```

---

## 5. Data Protection (NDPA 2023)

### 5.1 Overview

The Nigeria Data Protection Act (NDPA) 2023, operationalized through the General Application and Implementation Directive (GAID) 2025 (effective September 19, 2025), governs all processing of personal data.

### 5.2 Vescrow Classification

As a fintech processing large volumes of financial and personal data, Vescrow is classified as a **Data Controller/Processor of Major Importance (DCPMI)**.

### 5.3 Compliance Requirements

| Requirement | Details | Vescrow Implementation |
|------------|---------|----------------------|
| **NDPC Registration** | Register with Nigeria Data Protection Commission | Corporate compliance action |
| **DPO Appointment** | Appoint qualified Data Protection Officer | Corporate compliance action |
| **Annual Audit** | Compliance Audit Returns (CAR) by March 31 yearly | Engage licensed DPCO |
| **Privacy Notice** | Clear, accessible privacy notices | `/privacy-policy` page + in-app notice |
| **SNAG** | Standard Notice to Address Grievance | Data subject rights request form |
| **DPIA** | Data Protection Impact Assessment for high-risk processing | Before launch |
| **Breach Notification** | Notify NDPC within prescribed timeline | Incident response playbook |
| **Consent Management** | Explicit, informed consent before data collection | Registration flow consent |
| **Data Minimization** | Collect only necessary data | Audit data fields against necessity |
| **Cross-Border Transfer** | Adequacy decision, SCCs, or explicit consent | Ensure all data stored in Nigeria |
| **Right to Access** | Users can request their data | `GET /auth/profile` + data export |
| **Right to Deletion** | Users can request account deletion | Account deletion flow |
| **Right to Rectification** | Users can correct their data | `PUT /auth/profile` |

### 5.4 Technical Implementation

```
Data Protection Measures:
├── Encryption at Rest
│   ├── BVN: AES-256-GCM encrypted, stored as hash + encrypted value
│   ├── NIN: AES-256-GCM encrypted
│   ├── Bank Account Numbers: AES-256-GCM encrypted
│   └── Encryption keys: Stored in environment variables, rotated quarterly
│
├── Encryption in Transit
│   └── TLS 1.3 for all HTTPS connections
│
├── PII Masking in API Responses
│   ├── Phone: 0801****5678
│   ├── Email: o***@gmail.com
│   ├── BVN: ****5678
│   └── Account Number: ****1234
│
├── Access Controls
│   ├── Role-based access (users can only access their own data)
│   ├── Admin access logged in audit trail
│   └── No bulk data export without Super Admin approval
│
├── Retention & Deletion
│   ├── Financial records: 5 years (CBN requirement)
│   ├── Audit logs: 5 years (CBN requirement)
│   ├── User data: Until deletion requested + 90 day grace period
│   └── KYC documents: Retained for duration of account + 5 years
│
└── Breach Response
    ├── Detection: Automated monitoring of access patterns
    ├── Assessment: Within 4 hours of detection
    ├── Notification: NDPC notified per prescribed timeline
    └── User notification: If high risk to data subjects
```

### 5.5 Penalties for Non-Compliance

| Violation | Penalty |
|-----------|---------|
| General non-compliance | Up to 2% of annual gross revenue or ₦10 million (whichever is higher) |
| Failure to register | Inclusion on NDPC non-compliance register |
| Repeated violations | Potential suspension/revocation of license |

---

## 6. Consumer Protection

### 6.1 FCCPA 2018

The Federal Competition and Consumer Protection Act provides the overarching consumer rights framework:

| Right | Vescrow Application |
|-------|---------------------|
| **Right to Information** | Clear transaction terms, fees, and timelines displayed before commitment |
| **Right to Fair Treatment** | Impartial dispute resolution; platform cannot favor either party |
| **Right to Redress** | Built-in dispute mechanism + escalation to FCCPC if needed |
| **Protection from Unfair Practices** | Transparent 3% fee structure; no hidden charges |
| **Right to Safety** | Fund segregation; escrow funds cannot be used for platform operations |

### 6.2 CBN Consumer Protection Regulations (2019)

| Requirement | Implementation |
|------------|---------------|
| Transparency | All fees, terms, and conditions clearly disclosed |
| Fair practices | Equal treatment of buyers and sellers |
| Complaint handling | Built-in dispute resolution + 48-hour acknowledgment SLA |
| Dispute resolution | Three-tier: Platform resolution → Admin mediation → External arbitration |
| Data confidentiality | NDPA-compliant data handling |

---

## 7. Payment Settlement & Withdrawal

### 7.1 Paystack Settlement Model

| Feature | Details |
|---------|---------|
| Standard settlement | T+1 (next working day, excludes weekends/holidays) |
| Payout on Demand | Up to 70% of processed balance immediately (eligible businesses) |
| Transfer API | Programmatic instant transfers to any Nigerian NUBAN account |

### 7.2 CBN Instant Payment Regulations (2026)

| Regulation | Impact on Vescrow |
|-----------|-------------------|
| Voluntary opt-in/opt-out | Users can opt out of receiving instant transfers |
| Adjustable transaction limits | Users can set personal limits |
| Enhanced MFA | Multi-factor auth required for instant payments |
| BVN Watchlist | Transfers to flagged BVNs may be paused up to 24 hours |

### 7.3 Vescrow Withdrawal Strategy

**Decision**: Instant withdrawal via Paystack Transfer API

```
Withdrawal Flow:
1. Seller requests withdrawal (amount, bank_code, account_number)
2. System validates:
   ├── KYC status ≥ Tier 1
   ├── Sufficient wallet balance
   ├── No pending disputes on related escrows
   ├── Within daily limit (₦5,000,000)
   ├── Within per-transaction limit (₦2,000,000)
   └── Past cool-down period (24h after first-ever withdrawal)
3. Create/retrieve Paystack Transfer Recipient
4. Debit seller wallet (double-entry ledger)
5. Initiate Paystack Transfer (amount in kobo)
6. Save withdrawal record (status: processing)
7. Await Paystack webhook (transfer.success / transfer.failed)
8. Update withdrawal status
9. Notify seller via email + SMS
```

### 7.4 Transfer Limits

| Limit | Value | Rationale |
|-------|-------|-----------|
| Daily withdrawal limit | ₦5,000,000 | Fraud prevention |
| Per-transaction limit | ₦2,000,000 | Fraud prevention |
| First withdrawal cool-down | 24 hours | New user fraud prevention |
| Minimum withdrawal | ₦1,000 | Operational minimum |

---

## 8. Inspection Period & Buyer Rights

### 8.1 Legal Position

There is **no legally mandated inspection period** for escrow transactions in Nigeria. The duration is determined by agreement between the parties.

### 8.2 Industry Benchmarks

| Platform | Default Period | Range |
|----------|---------------|-------|
| Vesicash | Configurable | 1–30 days |
| Escrow.com (Global) | Configurable | 1–30 days |
| PayScrow | 7 days | 3–14 days |
| EscrowLock | 7 days | Variable |

### 8.3 Vescrow Configuration

| Parameter | Value |
|-----------|-------|
| Default inspection period | 3 days (72 hours) |
| Minimum allowed | 1 day |
| Maximum allowed | 30 days |
| High-value threshold (>₦500,000) | Minimum 3 days enforced |
| Extension allowed | 1 extension per escrow, up to 7 additional days (requires seller approval) |
| Auto-release | If buyer does not approve/dispute within period, funds auto-release to seller |

### 8.4 Auto-Release Protection

To protect buyers from accidentally losing funds:
- Email notification sent when inspection period begins
- Reminder email sent 24 hours before auto-release
- SMS reminder sent 12 hours before auto-release
- Clear countdown displayed in the application

---

## 9. Record Retention Requirements

| Record Type | Minimum Retention | Legal Basis |
|------------|-------------------|-------------|
| Customer identity records (KYC) | 5 years after account closure | CBN AML/CFT Guidelines |
| Transaction records | 5 years | CBN AML/CFT Guidelines |
| Audit logs | 5 years | CBN AML/CFT Guidelines |
| Suspicious Transaction Reports | 5 years | NFIU regulations |
| Communication records | 5 years | General compliance |
| Consent records | Duration of processing + 3 years | NDPA 2023 |
| Financial statements | 6 years | Companies and Allied Matters Act |

### Technical Implementation

```
Retention Policy:
├── Database: Soft deletes only (never hard delete financial records)
├── Audit Logs: Append-only table, no UPDATE/DELETE permissions
├── Automated archival: Records > 5 years moved to cold storage
├── Backup: Daily database backups, 90-day retention
└── Encrypted backups: AES-256 encrypted, stored separately
```

---

## 10. Technical Implementation Checklist

### Pre-Launch Compliance

- [ ] **Legal Entity**: Register company with CAC
- [ ] **Legal Counsel**: Engage fintech regulatory lawyer
- [ ] **Banking Partner**: Establish relationship with CBN-licensed bank for fund holding
- [ ] **PSSP License Application**: Begin AIP process with CBN
- [ ] **Terms of Service**: Draft comprehensive ToS reviewed by legal
- [ ] **Privacy Policy**: NDPA-compliant privacy policy
- [ ] **NDPC Registration**: Register as Data Controller of Major Importance
- [ ] **DPO Appointment**: Appoint qualified Data Protection Officer
- [ ] **DPIA**: Complete Data Protection Impact Assessment
- [ ] **MLRO Appointment**: Appoint Money Laundering Reporting Officer
- [ ] **AML Policy**: Draft AML/CFT policy approved by board
- [ ] **Insurance**: Professional indemnity insurance
- [ ] **Dispute Resolution Policy**: Clear, published dispute resolution procedure

### Technical Compliance

- [ ] **KYC Integration**: BVN/NIN verification via NIBSS/NIMC APIs
- [ ] **Tiered KYC**: Implement transaction limits per KYC tier
- [ ] **AML Monitoring**: Automated transaction monitoring system
- [ ] **PII Encryption**: AES-256-GCM for all sensitive data
- [ ] **Audit Trail**: Immutable, append-only audit logging
- [ ] **Data Retention**: 5-year retention for all financial records
- [ ] **Webhook Security**: HMAC signature verification on all webhooks
- [ ] **Rate Limiting**: Protection against brute force and abuse
- [ ] **API Masking**: PII masked in all API responses
- [ ] **Breach Playbook**: Documented incident response procedure
- [ ] **Consent Flow**: Explicit consent collection during registration
- [ ] **Data Export**: User data export endpoint (NDPA right of access)
- [ ] **Account Deletion**: User account deletion with grace period

---

## 11. Regulatory Bodies & Contacts

| Body | Acronym | Jurisdiction | Website |
|------|---------|-------------|---------|
| Central Bank of Nigeria | CBN | Financial regulation, payment systems | [cbn.gov.ng](https://www.cbn.gov.ng) |
| Nigeria Data Protection Commission | NDPC | Data protection and privacy | [ndpc.gov.ng](https://ndpc.gov.ng) |
| Nigeria Financial Intelligence Unit | NFIU | AML/CFT reporting | [nfiu.gov.ng](https://www.nfiu.gov.ng) |
| Federal Competition & Consumer Protection Commission | FCCPC | Consumer protection | [fccpc.gov.ng](https://fccpc.gov.ng) |
| Nigeria Inter-Bank Settlement System | NIBSS | BVN verification, payment clearing | [nibss-plc.com.ng](https://www.nibss-plc.com.ng) |
| National Identity Management Commission | NIMC | NIN verification | [nimc.gov.ng](https://nimc.gov.ng) |
| Corporate Affairs Commission | CAC | Company registration | [cac.gov.ng](https://www.cac.gov.ng) |

---

**Document Version**: 1.0  
**Classification**: Internal — Confidential  
**Review Schedule**: Quarterly (regulatory landscape changes frequently)
