# Vescrow — Database Schema Documentation

**Version**: 2.0  
**Last Updated**: August 2026  
**Database**: PostgreSQL 15  
**ORM**: GORM v1.25+  
**Database Name**: `vescrow_db`

---

## Table of Contents

1. [Overview](#1-overview)
2. [Entity Relationship Diagram](#2-entity-relationship-diagram)
3. [Table Definitions](#3-table-definitions)
4. [Indexes](#4-indexes)
5. [Constraints & Rules](#5-constraints--rules)
6. [Data Types & Conventions](#6-data-types--conventions)

---

## 1. Overview

All tables reside in a single PostgreSQL database (`vescrow_db`). The schema is auto-managed by GORM's AutoMigrate feature.

### Table Summary

| # | Table | Domain | Description |
|---|-------|--------|-------------|
| 1 | `roles` | Auth | RBAC role definitions |
| 2 | `users` | Auth | User accounts |
| 3 | `kyc_documents` | Auth | KYC verification submissions |
| 4 | `escrow_listings` | Listing | Shareable product listings |
| 5 | `payment_reservations` | Listing | Temporary slots during checkout |
| 6 | `escrows` | Escrow | Escrow transactions |
| 7 | `escrow_milestones` | Escrow | Transaction milestones |
| 8 | `escrow_payments` | Payment | Payment records (Paystack) |
| 9 | `webhook_failures` | Payment | Failed webhook payloads for replay |
| 10 | `payout_outbox` | Payout | Async withdrawal transfer jobs |
| 11 | `messages` | Escrow | In-escrow buyer/seller messages |
| 12 | `wallets` | Wallet | User and system wallets |
| 13 | `ledger_entries` | Wallet | Double-entry bookkeeping records |
| 14 | `withdrawals` | Wallet | Bank withdrawal requests |
| 15 | `disputes` | Dispute | Dispute cases |
| 16 | `dispute_evidence` | Dispute | Evidence file uploads |
| 17 | `dispute_comments` | Dispute | Discussion thread comments |
| 18 | `trust_profiles` | Trust | User trust score and status |
| 19 | `trust_events` | Trust | Score adjustment history |
| 20 | `block_entries` | Trust | User/NIN/IP/device blocks |
| 21 | `appeals` | Trust | Block appeal submissions |
| 22 | `user_devices` | Trust | Known device fingerprints |
| 23 | `organizations` | B2B | Tenant organizations |
| 24 | `members` | B2B | Org membership |
| 25 | `api_keys` | B2B | Hashed organization API keys |
| 26 | `inbox_notifications` | Notification | In-app notification inbox |
| 27 | `audit_logs` | Audit | Immutable activity trail |
| 28 | `platform_settings` | Admin | Platform configuration key-value |

> **Migration note**: The legacy `kyc_tier` column on `users` was removed. KYC gating uses `kyc_status` only.

---

## 2. Entity Relationship Diagram

```
                                ┌──────────────────┐
                                │      roles       │
                                │──────────────────│
                                │ id (PK)          │
                                │ name             │
                                │ permissions      │
                                └────────┬─────────┘
                                         │ 1:N
                                         ▼
┌───────────────────┐           ┌──────────────────┐           ┌──────────────────────┐
│  kyc_documents    │◄──────────│      users       │──────────►│       wallets        │
│───────────────────│   1:N     │──────────────────│    1:N    │──────────────────────│
│ id (PK)           │           │ id (PK)          │           │ id (PK)              │
│ user_id (FK)      │           │ name             │           │ user_id (FK)         │
│ document_type     │           │ email (UK)       │           │ wallet_type          │
│ document_number   │           │ phone (UK)       │           │ balance_kobo         │
│ document_url      │           │ password_hash    │           │ currency             │
│ selfie_url        │           │ role_id (FK)     │           │ is_frozen            │
│ status            │           │ status           │           └───────────┬──────────┘
│ verified_by       │           │ kyc_status       │                       │ 1:N
│ rejection_reason  │           │ organization_id  │            ┌──────────┴──────────┐
└───────────────────┘           │ bvn_hash         │            │                     │
                                │ nin_hash         │    ┌───────┴──────────┐  ┌───────┴─────────┐
                                └──────┬───────────┘    │ ledger_entries   │  │  withdrawals    │
                                       │                │──────────────────│  │─────────────────│
                         ┌─────────────┼─────────┐      │ id (PK)          │  │ id (PK)         │
                         │             │         │      │ wallet_id (FK)   │  │ wallet_id (FK)  │
                    Buyer│        Seller│    Opens│      │ escrow_id (FK)   │  │ user_id (FK)    │
                         ▼             ▼         │      │ entry_type       │  │ amount_kobo     │
                  ┌──────────────────┐           │      │ amount_kobo      │  │ bank_code       │
                  │     escrows      │           │      │ balance_after    │  │ account_number  │
                  │──────────────────│           │      │ reference (UK)   │  │ status          │
                  │ id (PK)          │           │      │ description      │  │ recipient_code  │
                  │ title            │           │      │ category         │  │ transfer_code   │
                  │ description      │           │      └──────────────────┘  └─────────────────┘
                  │ amount_kobo      │           │
                  │ currency         │           │
                  │ buyer_id (FK)    │           │
                  │ seller_id (FK)   │           │
                  │ status           │           │
                  │ delivery_date    │           │
                  │ inspection_days  │           │
                  │ escrow_fee_kobo  │           │
                  │ platform_fee     │           │
                  └──┬───┬───┬───┬──┘           │
                     │   │   │   │              │
        ┌────────────┘   │   │   └────────┐     │
        ▼                │   │            ▼     │
┌───────────────┐        │   │     ┌────────────┴───┐
│  milestones   │        │   │     │    disputes    │
│───────────────│        │   │     │────────────────│
│ id (PK)       │        │   │     │ id (PK)        │
│ escrow_id(FK) │        │   │     │ escrow_id (FK) │
│ title         │        │   │     │ opened_by (FK) │
│ amount_kobo   │        │   │     │ reason         │
│ sequence      │        │   │     │ status         │
│ status        │        │   │     │ resolution     │
│ due_date      │        │   │     │ resolved_by    │
└───────────────┘        │   │     └──┬──────┬──────┘
                         │   │        │      │
                         ▼   │        ▼      ▼
                  ┌──────────┴──┐  ┌──────┐ ┌──────────┐
                  │  messages   │  │evid. │ │comments  │
                  │─────────────│  │──────│ │──────────│
                  │ id (PK)     │  │id(PK)│ │ id (PK)  │
                  │ escrow_id   │  │disp. │ │ disp_id  │
                  │ sender_id   │  │upldr │ │ author   │
                  │ content     │  │file  │ │ role     │
                  │ attachment  │  │desc. │ │ content  │
                  └─────────────┘  └──────┘ └──────────┘

┌──────────────────┐        ┌────────────────────┐
│   audit_logs     │        │ platform_settings  │
│──────────────────│        │────────────────────│
│ id (PK)          │        │ id (PK)            │
│ actor_id (FK)    │        │ key (UK)           │
│ actor_type       │        │ value              │
│ action           │        │ description        │
│ resource_type    │        │ updated_at         │
│ resource_id      │        └────────────────────┘
│ old_state (JSON) │
│ new_state (JSON) │
│ ip_address       │
│ user_agent       │
│ created_at       │
└──────────────────┘
```

---

## 3. Table Definitions

### 3.1 `roles`

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `SERIAL` | PK | Auto-increment ID |
| `name` | `VARCHAR(50)` | UNIQUE, NOT NULL | Role name: `user`, `admin`, `super_admin` |
| `permissions` | `TEXT` | | JSON string of permissions |
| `created_at` | `TIMESTAMP` | NOT NULL, DEFAULT NOW() | Creation time |
| `updated_at` | `TIMESTAMP` | NOT NULL | Last update |
| `deleted_at` | `TIMESTAMP` | INDEX | Soft delete |

**Seed Data**:
```
{ id: 1, name: "user",        permissions: "" }
{ id: 2, name: "admin",       permissions: "manage_users,manage_kyc,manage_disputes" }
{ id: 3, name: "super_admin", permissions: "manage_users,manage_kyc,manage_disputes,manage_settings,view_audit" }
```

---

### 3.2 `users`

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `UUID` | PK, DEFAULT uuid_generate_v4() | User ID |
| `name` | `VARCHAR(255)` | NOT NULL | Full name |
| `email` | `VARCHAR(255)` | UNIQUE, NOT NULL | Email address |
| `phone` | `VARCHAR(20)` | UNIQUE, NOT NULL | Phone number (E.164 format) |
| `password_hash` | `BYTEA` | NOT NULL | Argon2id hash |
| `role_id` | `INTEGER` | FK → roles(id), DEFAULT 1 | User role |
| `status` | `VARCHAR(20)` | NOT NULL, DEFAULT 'active' | `active`, `suspended`, `banned` |
| `email_verified` | `BOOLEAN` | DEFAULT false | Email verification status |
| `phone_verified` | `BOOLEAN` | DEFAULT false | Phone verification status |
| `is_2fa_enabled` | `BOOLEAN` | DEFAULT false | 2FA enabled flag |
| `kyc_status` | `VARCHAR(20)` | DEFAULT 'none' | `none`, `pending`, `verified`, `approved`, `rejected` |
| `organization_id` | `UUID` | FK → organizations(id), NULLABLE | B2B tenant association |
| `totp_secret` | `TEXT` | | TOTP secret for admin MFA (never exposed in API) |
| `bvn_hash` | `TEXT` | | AES-256-GCM encrypted BVN |
| `nin_hash` | `TEXT` | | AES-256-GCM encrypted NIN |
| `avatar_url` | `VARCHAR(500)` | | Profile picture URL |
| `created_at` | `TIMESTAMP` | NOT NULL | Registration time |
| `updated_at` | `TIMESTAMP` | NOT NULL | Last update |
| `deleted_at` | `TIMESTAMP` | INDEX | Soft delete |

---

### 3.3 `kyc_documents`

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `UUID` | PK | Document ID |
| `user_id` | `UUID` | FK → users(id), NOT NULL | Owner |
| `document_type` | `VARCHAR(30)` | NOT NULL | `bvn`, `nin`, `national_id`, `passport`, `drivers_license`, `cac_cert`, `utility_bill` |
| `document_number_hash` | `TEXT` | | Encrypted document number |
| `document_url` | `VARCHAR(500)` | | Storage path/URL |
| `selfie_url` | `VARCHAR(500)` | | Selfie for face match |
| `status` | `VARCHAR(20)` | DEFAULT 'pending' | `pending`, `approved`, `rejected` |
| `rejection_reason` | `TEXT` | | Reason if rejected |
| `verified_by` | `UUID` | FK → users(id) | Admin who verified |
| `verified_at` | `TIMESTAMP` | | Verification time |
| `created_at` | `TIMESTAMP` | NOT NULL | Submission time |

---

### 3.4 `escrow_listings`

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `UUID` | PK | Listing ID |
| `creator_id` | `UUID` | FK → users(id), NOT NULL | User who created the listing |
| `creator_role` | `VARCHAR(10)` | NOT NULL | `seller` or `buyer` |
| `seller_id` | `UUID` | FK → users(id), NOT NULL | Seller for spawned escrows |
| `title` | `VARCHAR(255)` | NOT NULL | Listing title |
| `description` | `TEXT` | NOT NULL | Product description (min 50 chars at API layer) |
| `images` | `JSONB` | NOT NULL, DEFAULT `'[]'` | Array of image URLs (1–5 required on create) |
| `amount_kobo` | `BIGINT` | NOT NULL | Price per unit |
| `currency` | `VARCHAR(3)` | DEFAULT 'NGN' | Currency code |
| `quantity_total` | `INTEGER` | DEFAULT 1 | Total available units |
| `quantity_sold` | `INTEGER` | DEFAULT 0 | Completed purchases |
| `quantity_reserved` | `INTEGER` | DEFAULT 0 | Slots held during checkout |
| `share_token` | `VARCHAR(32)` | UNIQUE, NOT NULL | Public share token |
| `status` | `VARCHAR(20)` | DEFAULT 'active' | `active`, `paused`, `sold_out`, `expired` |
| `inspection_period_days` | `INTEGER` | DEFAULT 3 | Passed to spawned escrows |
| `expires_at` | `TIMESTAMP` | | Optional listing expiry |
| `created_at` | `TIMESTAMP` | NOT NULL | |
| `updated_at` | `TIMESTAMP` | NOT NULL | |
| `deleted_at` | `TIMESTAMP` | INDEX | Soft delete |

---

### 3.5 `payment_reservations`

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `UUID` | PK | Reservation ID |
| `listing_id` | `UUID` | FK → escrow_listings(id), NOT NULL | Parent listing |
| `escrow_id` | `UUID` | FK → escrows(id), UNIQUE, NOT NULL | Draft escrow created for checkout |
| `buyer_id` | `UUID` | FK → users(id), NOT NULL | Buyer holding the slot |
| `status` | `VARCHAR(20)` | DEFAULT 'reserved' | `reserved`, `completed`, `expired` |
| `slot_number` | `INTEGER` | NOT NULL | Unit number within listing |
| `expires_at` | `TIMESTAMP` | NOT NULL, INDEX | Reservation TTL |
| `created_at` | `TIMESTAMP` | NOT NULL | |
| `updated_at` | `TIMESTAMP` | NOT NULL | |

---

### 3.6 `escrows`

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `UUID` | PK | Escrow ID |
| `title` | `VARCHAR(255)` | NOT NULL | Transaction title |
| `description` | `TEXT` | | Detailed description (seller product copy after accept) |
| `images` | `JSONB` | | Product image URLs (`string[]`, 1–5 items) |
| `amount_kobo` | `BIGINT` | NOT NULL | Amount in kobo (₦1 = 100 kobo) |
| `currency` | `VARCHAR(3)` | DEFAULT 'NGN' | Currency code |
| `listing_id` | `UUID` | FK → escrow_listings(id), NULLABLE | Source listing (if any) |
| `organization_id` | `UUID` | FK → organizations(id), NULLABLE | B2B tenant |
| `slot_number` | `INTEGER` | DEFAULT 0 | Unit slot from listing |
| `buyer_id` | `UUID` | FK → users(id), NOT NULL | Buyer user |
| `seller_id` | `UUID` | FK → users(id), NULLABLE | Seller user (null until invite accepted / linked) |
| `pending_seller_email` | `VARCHAR(255)` | NULLABLE, indexed | Invited seller email before registration |
| `invite_token` | `VARCHAR(64)` | UNIQUE, indexed | Public invite link token (`/invite/{token}`) |
| `status` | `VARCHAR(30)` | NOT NULL, DEFAULT 'draft' | See state machine |
| `delivery_date` | `DATE` | | Expected delivery date |
| `inspection_period_days` | `INTEGER` | DEFAULT 3 | 1–30 days |
| `escrow_fee_kobo` | `BIGINT` | | Calculated platform fee |
| `platform_fee_kobo` | `BIGINT` | | Same as escrow_fee (configurable) |
| `incoming_fee_kobo` | `BIGINT` | DEFAULT 0 | Fee on buyer side |
| `outgoing_fee_kobo` | `BIGINT` | DEFAULT 0 | Fee on seller side |
| `buyer_total_kobo` | `BIGINT` | DEFAULT 0 | Total buyer pays (amount + fees) |
| `seller_bank_code` | `VARCHAR(10)` | | Seller payout bank |
| `seller_account_number` | `VARCHAR(20)` | | Seller payout account |
| `seller_account_name` | `VARCHAR(255)` | | Verified account name |
| `disbursement_status` | `VARCHAR(20)` | | Payout status after release |
| `disbursement_reference` | `VARCHAR(100)` | | Paystack transfer reference |
| `disbursed_at` | `TIMESTAMP` | | When seller was paid out |
| `payment_method` | `VARCHAR(20)` | DEFAULT 'paystack' | Payment provider |
| `delivery_proof_url` | `VARCHAR(500)` | | Proof of delivery |
| `metadata` | `JSONB` | | Flexible metadata |
| `funded_at` | `TIMESTAMP` | | When payment confirmed |
| `delivered_at` | `TIMESTAMP` | | When marked delivered |
| `completed_at` | `TIMESTAMP` | | When approved/auto-released |
| `expires_at` | `TIMESTAMP` | | Expiry time (72h window for seller response + product details) |
| `created_at` | `TIMESTAMP` | NOT NULL | |
| `updated_at` | `TIMESTAMP` | NOT NULL | |
| `deleted_at` | `TIMESTAMP` | INDEX | Soft delete |

**Status Values**: `draft`, `pending_acceptance`, `awaiting_product_details`, `accepted`, `funded`, `in_progress`, `delivered`, `completed`, `disputed`, `refunded`, `partial_release`, `cancelled`, `expired`, `rejected`

---

### 3.7 `escrow_milestones`

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `UUID` | PK | Milestone ID |
| `escrow_id` | `UUID` | FK → escrows(id), NOT NULL | Parent escrow |
| `title` | `VARCHAR(255)` | NOT NULL | Milestone title |
| `description` | `TEXT` | | Details |
| `amount_kobo` | `BIGINT` | NOT NULL | Milestone amount |
| `sequence_order` | `INTEGER` | NOT NULL | Order (1, 2, 3...) |
| `status` | `VARCHAR(20)` | DEFAULT 'pending' | `pending`, `in_progress`, `completed`, `disputed` |
| `due_date` | `TIMESTAMP` | | Expected completion date |
| `completed_at` | `TIMESTAMP` | | Actual completion time |
| `created_at` | `TIMESTAMP` | NOT NULL | |

---

### 3.8 `escrow_payments`

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `UUID` | PK | Payment ID |
| `escrow_id` | `UUID` | FK → escrows(id), NOT NULL | Related escrow |
| `user_id` | `UUID` | FK → users(id), NOT NULL | Payer |
| `provider` | `VARCHAR(20)` | DEFAULT 'paystack' | Payment provider |
| `provider_reference` | `VARCHAR(100)` | UNIQUE | Provider's transaction reference |
| `paystack_reference` | `VARCHAR(100)` | UNIQUE | Our generated reference |
| `amount_kobo` | `BIGINT` | NOT NULL | Payment amount |
| `status` | `VARCHAR(20)` | DEFAULT 'pending' | `pending`, `confirmed`, `failed`, `refunded` |
| `provider_response` | `JSONB` | | Full provider response |
| `idempotency_key` | `VARCHAR(100)` | UNIQUE | Deduplication key |
| `paid_at` | `TIMESTAMP` | | Confirmation time |
| `created_at` | `TIMESTAMP` | NOT NULL | |

---

### 3.9 `webhook_failures`

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `UUID` | PK | Failure record ID |
| `provider` | `VARCHAR(30)` | NOT NULL, INDEX | e.g. `paystack` |
| `event_type` | `VARCHAR(100)` | | Webhook event name |
| `reference` | `VARCHAR(255)` | INDEX | Payment/transfer reference |
| `payload` | `BYTEA` | NOT NULL | Raw webhook body |
| `error_msg` | `TEXT` | NOT NULL | Processing error |
| `resolved` | `BOOLEAN` | DEFAULT false, INDEX | Manual replay completed |
| `created_at` | `TIMESTAMP` | NOT NULL | |

---

### 3.10 `payout_outbox`

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `UUID` | PK | Outbox entry ID |
| `withdrawal_id` | `UUID` | NOT NULL, INDEX | Related withdrawal |
| `user_id` | `UUID` | NOT NULL, INDEX | Payee |
| `amount_kobo` | `BIGINT` | NOT NULL | Transfer amount |
| `bank_code` | `VARCHAR(10)` | NOT NULL | Destination bank |
| `account_number` | `VARCHAR(20)` | NOT NULL | Destination account |
| `status` | `VARCHAR(20)` | DEFAULT 'pending' | `pending`, `processing`, `completed`, `failed` |
| `attempts` | `INTEGER` | DEFAULT 0 | Retry count |
| `last_error` | `TEXT` | | Last failure message |
| `processed_at` | `TIMESTAMP` | | Completion time |
| `created_at` | `TIMESTAMP` | NOT NULL | |
| `updated_at` | `TIMESTAMP` | NOT NULL | |

---

### 3.11 `messages`

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `UUID` | PK | Message ID |
| `escrow_id` | `UUID` | FK → escrows(id), NOT NULL | Related escrow |
| `sender_id` | `UUID` | FK → users(id), NOT NULL | Message author |
| `content` | `TEXT` | NOT NULL | Message text |
| `attachment_url` | `VARCHAR(500)` | | Attached file URL |
| `created_at` | `TIMESTAMP` | NOT NULL | |

---

### 3.12 `wallets`

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `UUID` | PK | Wallet ID |
| `user_id` | `UUID` | FK → users(id), NULLABLE | NULL for system wallets |
| `wallet_type` | `VARCHAR(20)` | NOT NULL | `buyer`, `seller`, `escrow`, `platform_fee` |
| `balance_kobo` | `BIGINT` | NOT NULL, DEFAULT 0 | Current balance |
| `currency` | `VARCHAR(3)` | DEFAULT 'NGN' | Currency |
| `is_frozen` | `BOOLEAN` | DEFAULT false | Frozen flag (disputes) |
| `created_at` | `TIMESTAMP` | NOT NULL | |
| `updated_at` | `TIMESTAMP` | NOT NULL | |

**Unique Constraint**: `(user_id, wallet_type)` — one wallet per type per user.

**System Wallets** (seeded, `user_id = NULL`):
- `escrow` — temporary holding during active escrows
- `platform_fee` — accumulated 3% fees

---

### 3.13 `ledger_entries`

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `UUID` | PK | Entry ID |
| `wallet_id` | `UUID` | FK → wallets(id), NOT NULL | Target wallet |
| `escrow_id` | `UUID` | FK → escrows(id), NULLABLE | Related escrow |
| `entry_type` | `VARCHAR(10)` | NOT NULL | `debit` or `credit` |
| `amount_kobo` | `BIGINT` | NOT NULL | Entry amount (always positive) |
| `balance_after_kobo` | `BIGINT` | NOT NULL | Running balance snapshot |
| `reference` | `VARCHAR(100)` | UNIQUE | Unique entry reference |
| `description` | `VARCHAR(500)` | | Human-readable description |
| `category` | `VARCHAR(30)` | NOT NULL | `escrow_fund`, `escrow_release`, `escrow_refund`, `withdrawal`, `platform_fee`, `deposit` |
| `created_at` | `TIMESTAMP` | NOT NULL | |

> ⚠️ **IMMUTABLE**: This table supports INSERT only. No UPDATE or DELETE operations.

---

### 3.14 `withdrawals`

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `UUID` | PK | Withdrawal ID |
| `wallet_id` | `UUID` | FK → wallets(id), NOT NULL | Source wallet |
| `user_id` | `UUID` | FK → users(id), NOT NULL | Requesting user |
| `amount_kobo` | `BIGINT` | NOT NULL | Withdrawal amount |
| `bank_code` | `VARCHAR(10)` | NOT NULL | Nigerian bank code |
| `account_number` | `VARCHAR(20)` | NOT NULL | NUBAN account number |
| `account_name` | `VARCHAR(255)` | NOT NULL | Verified account name |
| `recipient_code` | `VARCHAR(100)` | | Paystack recipient_code |
| `transfer_code` | `VARCHAR(100)` | | Paystack transfer_code |
| `status` | `VARCHAR(20)` | DEFAULT 'pending' | `pending`, `processing`, `completed`, `failed`, `reversed` |
| `failure_reason` | `TEXT` | | Reason if failed |
| `created_at` | `TIMESTAMP` | NOT NULL | |
| `completed_at` | `TIMESTAMP` | | Completion time |

---

### 3.15 `disputes`

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `UUID` | PK | Dispute ID |
| `escrow_id` | `UUID` | FK → escrows(id), NOT NULL | Related escrow |
| `opened_by` | `UUID` | FK → users(id), NOT NULL | User who opened |
| `reason` | `VARCHAR(100)` | NOT NULL | Dispute reason category |
| `description` | `TEXT` | NOT NULL | Detailed description |
| `status` | `VARCHAR(30)` | DEFAULT 'open' | `open`, `seller_responded`, `under_review`, `resolved` |
| `resolution` | `VARCHAR(20)` | NULLABLE | `full_refund`, `full_release`, `partial_split` |
| `buyer_amount_kobo` | `BIGINT` | | Amount to buyer (partial split) |
| `seller_amount_kobo` | `BIGINT` | | Amount to seller (partial split) |
| `resolved_by` | `UUID` | FK → users(id) | Admin who resolved |
| `resolution_notes` | `TEXT` | | Admin's resolution explanation |
| `opened_at` | `TIMESTAMP` | NOT NULL | |
| `resolved_at` | `TIMESTAMP` | | |
| `created_at` | `TIMESTAMP` | NOT NULL | |

---

### 3.16 `dispute_evidence`

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `UUID` | PK | Evidence ID |
| `dispute_id` | `UUID` | FK → disputes(id), NOT NULL | Parent dispute |
| `uploaded_by` | `UUID` | FK → users(id), NOT NULL | Uploader |
| `file_url` | `VARCHAR(500)` | NOT NULL | File storage URL |
| `file_type` | `VARCHAR(20)` | | `image`, `pdf`, `document` |
| `description` | `VARCHAR(500)` | | Evidence description |
| `created_at` | `TIMESTAMP` | NOT NULL | |

---

### 3.17 `dispute_comments`

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `UUID` | PK | Comment ID |
| `dispute_id` | `UUID` | FK → disputes(id), NOT NULL | Parent dispute |
| `author_id` | `UUID` | FK → users(id), NOT NULL | Comment author |
| `role` | `VARCHAR(10)` | NOT NULL | `buyer`, `seller`, `admin` |
| `content` | `TEXT` | NOT NULL | Comment text |
| `created_at` | `TIMESTAMP` | NOT NULL | |

---

### 3.18 `trust_profiles`

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `user_id` | `UUID` | PK, FK → users(id) | User |
| `score` | `INTEGER` | DEFAULT 100 | Trust score (0–100) |
| `status` | `VARCHAR(20)` | DEFAULT 'good' | `good`, `warning`, `restricted`, `blocked` |
| `last_event_at` | `TIMESTAMP` | | Last score change |
| `updated_at` | `TIMESTAMP` | NOT NULL | |

---

### 3.19 `trust_events`

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `UUID` | PK | Event ID |
| `user_id` | `UUID` | FK → users(id), NOT NULL, INDEX | Affected user |
| `event_type` | `VARCHAR(40)` | NOT NULL | Event category |
| `delta` | `INTEGER` | NOT NULL | Score change (+/-) |
| `escrow_id` | `UUID` | FK → escrows(id), NULLABLE | Related escrow |
| `reason` | `TEXT` | | Human-readable reason |
| `created_at` | `TIMESTAMP` | NOT NULL | |

---

### 3.20 `block_entries`

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `UUID` | PK | Block ID |
| `user_id` | `UUID` | FK → users(id), NULLABLE, INDEX | Blocked user |
| `nin_hash` | `VARCHAR(255)` | INDEX | Blocked NIN hash |
| `ip_address` | `VARCHAR(45)` | INDEX | Blocked IP |
| `device_fp` | `VARCHAR(128)` | INDEX | Blocked device fingerprint |
| `reason` | `TEXT` | NOT NULL | Block reason |
| `blocked_at` | `TIMESTAMP` | NOT NULL | |
| `expires_at` | `TIMESTAMP` | | NULL = permanent |
| `active` | `BOOLEAN` | DEFAULT true | |

---

### 3.21 `appeals`

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `UUID` | PK | Appeal ID |
| `user_id` | `UUID` | FK → users(id), NOT NULL, INDEX | Appellant |
| `block_id` | `UUID` | FK → block_entries(id), NOT NULL | Related block |
| `reason` | `TEXT` | NOT NULL | Appeal justification |
| `evidence` | `JSONB` | | Supporting evidence |
| `status` | `VARCHAR(20)` | DEFAULT 'pending' | `pending`, `approved`, `rejected` |
| `reviewed_by` | `UUID` | FK → users(id) | Admin reviewer |
| `review_note` | `TEXT` | | Admin decision note |
| `created_at` | `TIMESTAMP` | NOT NULL | |
| `updated_at` | `TIMESTAMP` | NOT NULL | |

---

### 3.22 `user_devices`

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `UUID` | PK | Device record ID |
| `user_id` | `UUID` | FK → users(id), NOT NULL, INDEX | Owner |
| `device_fp` | `VARCHAR(128)` | NOT NULL, INDEX | Client fingerprint (`X-Device-FP`) |
| `last_seen` | `TIMESTAMP` | NOT NULL | Last activity |
| `created_at` | `TIMESTAMP` | NOT NULL | |

---

### 3.23 `organizations`

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `UUID` | PK | Organization ID |
| `name` | `VARCHAR(255)` | NOT NULL | Display name |
| `slug` | `VARCHAR(100)` | UNIQUE, NOT NULL | URL-safe identifier |
| `status` | `VARCHAR(20)` | DEFAULT 'active' | `active`, `suspended` |
| `created_at` | `TIMESTAMP` | NOT NULL | |
| `updated_at` | `TIMESTAMP` | NOT NULL | |
| `deleted_at` | `TIMESTAMP` | INDEX | Soft delete |

---

### 3.24 `members`

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `UUID` | PK | Membership ID |
| `organization_id` | `UUID` | FK → organizations(id), NOT NULL, INDEX | Tenant |
| `user_id` | `UUID` | FK → users(id), NOT NULL, INDEX | Member user |
| `role` | `VARCHAR(30)` | DEFAULT 'member' | `owner`, `admin`, `operator`, `viewer` |
| `created_at` | `TIMESTAMP` | NOT NULL | |
| `updated_at` | `TIMESTAMP` | NOT NULL | |
| `deleted_at` | `TIMESTAMP` | INDEX | Soft delete |

---

### 3.25 `api_keys`

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `UUID` | PK | Key ID |
| `organization_id` | `UUID` | FK → organizations(id), NOT NULL, INDEX | Owning org |
| `name` | `VARCHAR(100)` | NOT NULL | Key label |
| `key_prefix` | `VARCHAR(16)` | NOT NULL, INDEX | Lookup prefix |
| `key_hash` | `BYTEA` | NOT NULL | Argon2id hash of full key |
| `scopes` | `TEXT` | NOT NULL | Comma-separated permissions |
| `last_used_at` | `TIMESTAMP` | | Last API call |
| `expires_at` | `TIMESTAMP` | | Optional expiry |
| `revoked_at` | `TIMESTAMP` | | Revocation time |
| `created_at` | `TIMESTAMP` | NOT NULL | |
| `updated_at` | `TIMESTAMP` | NOT NULL | |
| `deleted_at` | `TIMESTAMP` | INDEX | Soft delete |

---

### 3.26 `inbox_notifications`

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `UUID` | PK | Notification ID |
| `user_id` | `UUID` | FK → users(id), NOT NULL, INDEX | Recipient |
| `type` | `VARCHAR(30)` | NOT NULL | Notification category |
| `title` | `VARCHAR(255)` | NOT NULL | Title |
| `body` | `TEXT` | NOT NULL | Message body |
| `resource_type` | `VARCHAR(30)` | | Linked resource type |
| `resource_id` | `UUID` | | Linked resource ID |
| `dedupe_key` | `VARCHAR(100)` | UNIQUE with `user_id` when not NULL | Idempotency key to prevent duplicate inbox rows |
| `read_at` | `TIMESTAMP` | | When marked read |
| `created_at` | `TIMESTAMP` | NOT NULL | |

---

### 3.27 `audit_logs`

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `UUID` | PK | Log entry ID |
| `actor_id` | `UUID` | NULLABLE | User/admin who performed action (NULL for system) |
| `actor_type` | `VARCHAR(10)` | NOT NULL | `user`, `admin`, `system` |
| `action` | `VARCHAR(100)` | NOT NULL | Action name (e.g., `escrow.created`) |
| `resource_type` | `VARCHAR(30)` | NOT NULL | `user`, `escrow`, `wallet`, `dispute`, `payment`, `withdrawal`, `kyc` |
| `resource_id` | `UUID` | NOT NULL | ID of the affected resource |
| `old_state` | `JSONB` | | Previous state (for updates) |
| `new_state` | `JSONB` | | New state (after action) |
| `ip_address` | `VARCHAR(45)` | | Client IP address |
| `user_agent` | `VARCHAR(500)` | | Client user agent |
| `created_at` | `TIMESTAMP` | NOT NULL | |

> ⚠️ **IMMUTABLE**: This table supports INSERT only. No UPDATE or DELETE operations. 5-year retention.

---

### 3.28 `platform_settings`

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `SERIAL` | PK | Setting ID |
| `key` | `VARCHAR(100)` | UNIQUE, NOT NULL | Setting key |
| `value` | `TEXT` | NOT NULL | Setting value |
| `description` | `VARCHAR(500)` | | Human-readable description |
| `updated_at` | `TIMESTAMP` | NOT NULL | Last update |

**Seed Data**:
```
{ key: "platform_fee_percent",        value: "3",       description: "Platform fee percentage" }
{ key: "default_inspection_days",     value: "3",       description: "Default inspection period in days" }
{ key: "max_inspection_days",         value: "30",      description: "Maximum inspection period" }
{ key: "min_inspection_days",         value: "1",       description: "Minimum inspection period" }
{ key: "daily_withdrawal_limit_kobo", value: "500000000", description: "Daily withdrawal limit (₦5M)" }
{ key: "per_tx_withdrawal_limit_kobo", value: "200000000", description: "Per-transaction withdrawal limit (₦2M)" }
{ key: "min_withdrawal_kobo",         value: "100000",  description: "Minimum withdrawal (₦1,000)" }
{ key: "withdrawal_cooldown_hours",   value: "24",      description: "Cool-down after first withdrawal" }
{ key: "high_value_threshold_kobo",   value: "50000000", description: "High-value transaction threshold (₦500K)" }
{ key: "pending_acceptance_expiry_hours", value: "72",   description: "Hours before pending escrow expires" }
```

---

## 4. Indexes

### 4.1 Primary & Unique Indexes

| Table | Column(s) | Type |
|-------|-----------|------|
| All tables | `id` | PRIMARY KEY |
| `users` | `email` | UNIQUE |
| `users` | `phone` | UNIQUE |
| `escrow_payments` | `provider_reference` | UNIQUE |
| `escrow_payments` | `paystack_reference` | UNIQUE |
| `escrow_payments` | `idempotency_key` | UNIQUE |
| `ledger_entries` | `reference` | UNIQUE |
| `wallets` | `(user_id, wallet_type)` | UNIQUE COMPOSITE |
| `escrow_listings` | `share_token` | UNIQUE |
| `platform_settings` | `key` | UNIQUE |

### 4.2 Performance Indexes

| Table | Column(s) | Type | Purpose |
|-------|-----------|------|---------|
| `users` | `role_id` | B-TREE | Role-based queries |
| `users` | `kyc_status` | B-TREE | Admin KYC queue |
| `users` | `status` | B-TREE | Active user filtering |
| `escrows` | `buyer_id` | B-TREE | User's escrows |
| `escrows` | `seller_id` | B-TREE | User's escrows |
| `escrows` | `status` | B-TREE | Status-based filtering |
| `escrows` | `(status, delivered_at)` | COMPOSITE | Auto-release cron |
| `escrow_listings` | `share_token` | UNIQUE | Public lookup |
| `escrow_listings` | `status` | B-TREE | Active listing filter |
| `payment_reservations` | `listing_id` | B-TREE | Listing reservations |
| `payment_reservations` | `expires_at` | B-TREE | Expiry worker |
| `webhook_failures` | `provider` | B-TREE | Provider filter |
| `webhook_failures` | `resolved` | B-TREE | Unresolved queue |
| `payout_outbox` | `status` | B-TREE | Worker polling |
| `trust_events` | `user_id` | B-TREE | User history |
| `block_entries` | `user_id` | B-TREE | Active blocks |
| `appeals` | `user_id` | B-TREE | User appeals |
| `api_keys` | `key_prefix` | B-TREE | Key lookup |
| `inbox_notifications` | `user_id` | B-TREE | User inbox |
| `inbox_notifications` | `user_id`, `dedupe_key` | UNIQUE | Prevent duplicate inbox rows per event |
| `users` | `organization_id` | B-TREE | Tenant users |
| `escrows` | `listing_id` | B-TREE | Listing orders |
| `escrows` | `organization_id` | B-TREE | Tenant escrows |
| `escrow_milestones` | `escrow_id` | B-TREE | Milestone listing |
| `escrow_payments` | `escrow_id` | B-TREE | Payment lookup |
| `messages` | `escrow_id` | B-TREE | Message thread |
| `wallets` | `user_id` | B-TREE | User wallet lookup |
| `ledger_entries` | `wallet_id` | B-TREE | Transaction history |
| `ledger_entries` | `escrow_id` | B-TREE | Escrow-related entries |
| `withdrawals` | `user_id` | B-TREE | User withdrawal history |
| `withdrawals` | `status` | B-TREE | Pending withdrawal queue |
| `disputes` | `escrow_id` | B-TREE | Escrow's disputes |
| `disputes` | `status` | B-TREE | Admin dispute queue |
| `audit_logs` | `(resource_type, resource_id)` | COMPOSITE | Resource history |
| `audit_logs` | `created_at` | B-TREE | Time-range queries |
| `audit_logs` | `actor_id` | B-TREE | User action history |

### 4.3 Soft Delete Indexes

All tables with `deleted_at`: B-TREE index on `deleted_at` (GORM convention).

---

## 5. Constraints & Rules

### 5.1 Foreign Key Constraints

| Child Table | Column | References | On Delete |
|------------|--------|------------|-----------|
| `users` | `role_id` | `roles(id)` | SET DEFAULT |
| `kyc_documents` | `user_id` | `users(id)` | CASCADE |
| `escrows` | `buyer_id` | `users(id)` | RESTRICT |
| `escrows` | `seller_id` | `users(id)` | RESTRICT |
| `escrow_milestones` | `escrow_id` | `escrows(id)` | CASCADE |
| `escrow_payments` | `escrow_id` | `escrows(id)` | RESTRICT |
| `messages` | `escrow_id` | `escrows(id)` | CASCADE |
| `wallets` | `user_id` | `users(id)` | RESTRICT |
| `ledger_entries` | `wallet_id` | `wallets(id)` | RESTRICT |
| `withdrawals` | `wallet_id` | `wallets(id)` | RESTRICT |
| `disputes` | `escrow_id` | `escrows(id)` | RESTRICT |
| `dispute_evidence` | `dispute_id` | `disputes(id)` | CASCADE |
| `dispute_comments` | `dispute_id` | `disputes(id)` | CASCADE |

### 5.2 Business Rules

| Rule | Enforcement |
|------|------------|
| Amount must be > 0 | Application-level CHECK |
| Buyer ≠ Seller | Application-level validation |
| Milestone amounts must sum to escrow amount | Application-level validation |
| Ledger entries are immutable | No UPDATE/DELETE permissions on table |
| Audit logs are immutable | No UPDATE/DELETE permissions on table |
| Balance cannot go negative | Application-level check before debit |

---

## 6. Data Types & Conventions

### 6.1 Monetary Values

| Convention | Details |
|-----------|---------|
| **Unit** | All monetary values stored in **kobo** (₦1 = 100 kobo) |
| **Type** | `BIGINT` (int64 in Go) |
| **Reason** | Avoids floating-point precision errors |
| **Display** | Convert to naira in API responses: `amount_kobo / 100` |
| **Example** | ₦10,000.50 = `1000050` kobo |

### 6.2 Timestamps

| Convention | Details |
|-----------|---------|
| **Type** | `TIMESTAMP WITH TIME ZONE` |
| **Timezone** | UTC in database, convert to WAT (UTC+1) for Nigerian display |
| **Format** | ISO 8601 in API responses: `2026-06-14T15:00:00Z` |

### 6.3 UUIDs

| Convention | Details |
|-----------|---------|
| **Type** | `UUID` (PostgreSQL native) |
| **Generation** | `uuid_generate_v4()` via pgcrypto/uuid-ossp extension |
| **Go Library** | `github.com/google/uuid` |

### 6.4 Sensitive Data

| Data | Storage | API Response |
|------|---------|-------------|
| Password | Argon2id hash | Never returned |
| BVN | AES-256-GCM encrypted | `****5678` |
| NIN | AES-256-GCM encrypted | `****1234` |
| Bank Account | AES-256-GCM encrypted | `****5678` |
| Phone | Plain text (indexed for lookup) | `0801****5678` |
| Email | Plain text (indexed for lookup) | `o***@gmail.com` |

---

**Document Version**: 2.0  
**Last Updated**: August 2026
