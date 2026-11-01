# Vescrow QA & Verification Test Guides (`/QSD/test`)

[![Test Type](https://img.shields.io/badge/Testing-Manual%20QA%20%26%20E2E%20Verification-blue.svg)](#overview)
[![Surface](https://img.shields.io/badge/Surfaces-Backend%20API%20%7C%20Swagger%20%7C%20Web%20App-orange.svg)](#document-index)
[![Status](https://img.shields.io/badge/QA%20Status-Release%20Certified-green.svg)](#test-execution-flow)

This directory contains manual test guides, Swagger verification walkthroughs, payment simulation procedures, and end-to-end QA acceptance scripts for the Vescrow platform.

---

## Overview

Because financial escrow systems handle real funds and legal contracts, manual and automated verification must cover all edge cases—such as payment webhook dropouts, race conditions during release, and dispute settlement splits. These guides provide repeatable step-by-step test instructions.

```
test/
├── README.md                               # This document index and testing framework
├── end-to-end-application-test-guide.md    # Full buyer-seller-admin lifecycle testing
├── new-workflow-backend-manual-test.md     # Backend verification for direct disbursement model
├── payment-manual-test-guide.md            # Paystack payment and HMAC webhook simulation
├── wallet-swagger-test-guide.md            # Swagger API testing guide for wallet & ledger endpoints
├── phase5-escrow-manual-test.md            # Escrow milestones, inspection and approval QA
└── phase8-dispute-manual-test.md           # Dispute handling, arbiter intervention & payout split QA
```

---

## Document Index

| Guide | Scope | Target Subsystem | Key Verification |
| :--- | :--- | :--- | :--- |
| [end-to-end-application-test-guide.md](./end-to-end-application-test-guide.md) | **E2E Acceptance** | Web App + Go API + DB | Complete transaction lifecycle from sign-up to final bank withdrawal |
| [new-workflow-backend-manual-test.md](./new-workflow-backend-manual-test.md) | **Backend Verification** | Go Fiber API | Direct escrow disbursement model without intermediate wallet hops |
| [payment-manual-test-guide.md](./payment-manual-test-guide.md) | **Payments QA** | Paystack Webhooks | HMAC-SHA512 verification, duplicate webhook idempotency, failed charges |
| [wallet-swagger-test-guide.md](./wallet-swagger-test-guide.md) | **Swagger API QA** | OpenAPI / Swagger UI | Ledger query endpoints, deposit initialization, and balance endpoints |
| [phase5-escrow-manual-test.md](./phase5-escrow-manual-test.md) | **Escrow Lifecycle** | Escrow State Machine | Fund-on-create, inspection timer countdown, buyer release authorization |
| [phase8-dispute-manual-test.md](./phase8-dispute-manual-test.md) | **Disputes QA** | Dispute Mediation Engine | Lock-on-dispute, evidence submission, partial refund split execution |

---

## Test Execution Flow

```mermaid
sequenceDiagram
    autonumber
    actor Tester
    participant Swagger as Swagger UI / Postman
    participant API as Vescrow Go API
    participant DB as Postgres Ledger
    participant Paystack as Paystack Sandbox

    Tester->>Swagger: Initialize Escrow (POST /escrows)
    Swagger->>API: Route Request with Test Token
    API->>DB: Create Escrow in Draft State
    Tester->>Paystack: Simulate Card Payment
    Paystack->>API: POST /webhooks/paystack (HMAC Verified)
    API->>DB: Credit Escrow Account (Debit Cash)
    Tester->>Swagger: Approve Delivery (PUT /escrows/:id/approve)
    API->>DB: Execute Ledger Transfer (Escrow -> Seller)
    Tester->>DB: Query Invariants: sum(debit) == sum(credit)
```
