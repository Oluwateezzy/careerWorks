# Understanding the Double-Entry Ledger in Vescrow

## The Problem: Why Can't We Just Use a `balance` Column?

Imagine you're running an escrow platform. You store each user's balance as a single number in the database:

```
| user_id | balance_kobo |
|---------|-------------|
| Alice   | 500,000     |
| Bob     | 0           |
```

Alice wants to buy a laptop from Bob for ₦5,000 (500,000 kobo). Here's what happens with a naive implementation:

```
1. Alice pays → balance = 0
2. Bob delivers → system adds 500,000 to Bob's balance
```

Now your database says:
```
| user_id | balance_kobo |
|---------|-------------|
| Alice   | 0           |
| Bob     | 500,000     |
```

**Looks fine, right?** Now Bob says: *"I never received the money."*

**Can you prove he did?** You look at the database. All you see is `balance_kobo = 500,000`. You have no idea:

- *When* it changed
- *Why* it changed
- *Where the money came from*
- Whether there was a ₦200 platform fee deducted
- Whether Alice was refunded first and then it was re-released

**This is the fundamental problem.** A single balance number is a **snapshot without memory**. It tells you the "what" but never the "how" or "why." In a financial system, that's catastrophic.

---

## What Double-Entry Bookkeeping Actually Is

Double-entry bookkeeping is a 500-year-old accounting principle invented in Renaissance Italy. The rule is simple but powerful:

> **Every financial event must be recorded as at least two entries that sum to zero: a debit from one account and a credit to another.**

Money never appears from nowhere and never vanishes. It always **moves** from one place to another. The ledger tracks every single movement.

### In Vescrow, This Means:

When Alice funds her escrow:
```
Entry 1: CREDIT Alice's wallet   +500,000 kobo  (money arrives from payment gateway)
Entry 2: DEBIT  Alice's wallet   -500,000 kobo  (money leaves to fund escrow)
Entry 3: CREDIT Escrow wallet    +500,000 kobo  (escrow system now holds the money)
```

When Bob's delivery is approved:
```
Entry 4: DEBIT  Escrow wallet     -500,000 kobo  (money leaves escrow)
Entry 5: CREDIT Bob's wallet      +485,000 kobo  (Bob gets paid minus fee)
Entry 6: CREDIT Platform wallet   +15,000 kobo   (platform takes its fee)
```

**Notice**: At every stage, `SUM(credits) - SUM(debits) = 0`. Money is never created or destroyed. It's always accounted for.

---

## The 5 Problems Double-Entry Solves

### 1. The "Where Did the Money Go?" Problem

**Without double-entry**: Bob's balance went from 0 to 485,000. You don't know why.

**With double-entry**: You query the ledger and see entry #5: "Release payout from escrow abc-123." You trace it back to entry #4 (escrow debit) and entry #3 (escrow funding). You have a complete paper trail from Alice's payment gateway deposit to Bob's wallet.

### 2. The "Your Books Don't Add Up" Problem

**Without double-entry**: A bug in your code accidentally credits Bob twice. His balance is 970,000 but only ₦5,000 entered the system. You discover this weeks later when you try to process payouts and your Paystack balance is short.

**With double-entry**: The global invariant `SUM(credits) - SUM(debits) = 0` breaks immediately. A scheduled integrity check flags the discrepancy before any real money leaves your system.

### 3. The "Race Condition" Problem

Two requests hit your server at the same time:
```
Request A: Withdraw ₦3,000 from Bob (balance: ₦5,000)
Request B: Withdraw ₦4,000 from Bob (balance: ₦5,000)
```

**Without double-entry**: Both read `balance = 500,000`. Both pass the `balance >= amount` check. Both deduct. Bob's balance goes to `-200,000`. You've just given away money you don't have.

**With double-entry**: Vescrow uses `SELECT ... FOR UPDATE` row locks inside the `RecordEntry` function. The first transaction locks Bob's wallet row. The second transaction waits. When the first completes, Bob's balance is 200,000. The second transaction reads the updated balance (200,000), sees it's less than 400,000, and fails gracefully.

### 4. The "Audit Trail" Problem

A user disputes a charge. A regulator asks for records. Your accountant needs to reconcile.

**Without double-entry**: You grep through application logs, hoping your logging was thorough enough. It wasn't.

**With double-entry**: Every financial event is an immutable row in the `ledger_entries` table with:
- A unique reference (`LEDGER_<uuid>`)
- A timestamp
- The amount
- The entry type (credit/debit)
- The category (deposit, escrow_fund, escrow_release, platform_fee, withdrawal, refund)
- A description ("Release payout from escrow abc-123")
- The escrow ID it relates to
- The `balance_after_kobo` snapshot at that exact moment

You can reconstruct the complete history of any wallet at any point in time.

### 5. The "Platform Fee Accounting" Problem

Your platform charges a 3% fee on each escrow. Where does that money go?

**Without double-entry**: It's... somewhere. Maybe you subtract it before crediting the seller. But there's no record of the fee as a separate entity.

**With double-entry**: The platform fee is an explicit credit to the `platform_fee` system wallet:
```
CREDIT platform_fee_wallet +15,000 kobo  (category: platform_fee)
```

At any time, you can query the total platform revenue:
```sql
SELECT SUM(amount_kobo)
FROM ledger_entries
WHERE wallet_id = '<platform_fee_wallet_id>'
AND entry_type = 'credit';
```

---

## How It Works in Vescrow's Code

### The Three System Wallets

Seeded at startup in `database/seed.go`:

| Wallet | Purpose |
|---|---|
| `escrow` | Holds buyer funds between payment and delivery approval |
| `platform_fee` | Accumulates platform fees from completed escrows |
| `payout` | Staging area for bank payouts (future use) |

### The User Wallets

Created on-demand when a user first interacts with the wallet system:

| Wallet Type | When Created | Purpose |
|---|---|---|
| `buyer` | First time user funds an escrow | Temporary holding for gateway deposits before transfer to escrow |
| `seller` | First time balance is queried or payout is released | Receives payouts, source for withdrawals |

### The Flow of Money

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Paystack   │────▶│ Buyer Wallet │────▶│ Escrow Wallet│
│   Gateway    │     │  (buyer)     │     │  (system)    │
└──────────────┘     └──────────────┘     └──────┬───────┘
                                                  │
                                    ┌─────────────┼─────────────┐
                                    ▼                           ▼
                           ┌──────────────┐            ┌──────────────┐
                           │ Seller Wallet│            │ Platform Fee │
                           │  (seller)    │            │   (system)   │
                           └──────┬───────┘            └──────────────┘
                                  │
                                  ▼
                           ┌──────────────┐
                           │  Bank Payout │
                           │  (Paystack)  │
                           └──────────────┘
```

### The Golden Invariant

At **any point in time**, across the entire ledger:

```sql
SELECT
  SUM(CASE WHEN entry_type = 'credit' THEN amount_kobo ELSE 0 END) -
  SUM(CASE WHEN entry_type = 'debit' THEN amount_kobo ELSE 0 END)
FROM ledger_entries;

-- Result: ALWAYS 0
```

If this query ever returns anything other than zero, something is deeply wrong and no money should leave the system until it's investigated.

This is exactly what `TestLedgerIntegrity` validates:

1. **Global sum = 0** after a full fund → release lifecycle
2. **Per-escrow sum = 0** for entries scoped to a single escrow
3. **Every entry has a unique reference** (no duplicates)
4. **Running balance snapshots are accurate** (each `balance_after_kobo` matches the computed running total)
5. **Concurrent withdrawals never produce negative balances** (row locking works)

---

## Why This Matters for Vescrow

Vescrow is holding **other people's money**. That's fundamentally different from a social media app where a bug means someone sees the wrong post. In fintech:

- **A bug that creates money from nothing** means you owe Paystack real Naira that doesn't exist in user accounts.
- **A bug that destroys money** means a user's legitimate funds vanish and you face legal liability.
- **A missing audit trail** means you can't resolve disputes, respond to regulators, or prove you handled funds correctly.

Double-entry bookkeeping is the 500-year-old solution to all three problems. Every serious financial system uses it — from your bank to Stripe to the Central Bank of Nigeria. Now Vescrow does too.
