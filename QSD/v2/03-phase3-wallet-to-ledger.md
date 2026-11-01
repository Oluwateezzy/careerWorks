# Phase 3 — Wallet → Ledger Rename

> **Goal**: Rename all user-facing "Wallet" labels to "Ledger" across the entire frontend. Backend API routes and internal names remain unchanged to avoid breaking changes.

---

## Task 3.1 — Frontend: Rename Sidebar & Navigation Labels

### Description
Update all navigation components to display "Ledger" instead of "Wallet".

### Files to Modify
- [`src/components/layout/app-sidebar.tsx`](file:///Users/oluwatobiloba/Desktop/qsd/vescrow/vescrow_frontend/src/components/layout/app-sidebar.tsx) — Line 41
- [`src/components/layout/mobile-bottom-nav.tsx`](file:///Users/oluwatobiloba/Desktop/qsd/vescrow/vescrow_frontend/src/components/layout/mobile-bottom-nav.tsx) — Line 26
- [`src/components/layout/mobile-header.tsx`](file:///Users/oluwatobiloba/Desktop/qsd/vescrow/vescrow_frontend/src/components/layout/mobile-header.tsx)

### Implementation Details

**`app-sidebar.tsx`:**
```diff
-    { href: ROUTES.WALLET, label: 'Wallet & Ledger', icon: Wallet },
+    { href: ROUTES.WALLET, label: 'Ledger', icon: BookOpen },
```
Change icon from `Wallet` to `BookOpen` (from lucide-react) to reflect ledger concept.

**`mobile-bottom-nav.tsx`:**
```diff
-  { href: ROUTES.WALLET, label: 'Wallet', icon: Wallet },
+  { href: ROUTES.WALLET, label: 'Ledger', icon: BookOpen },
```

### Validation Criteria
- [ ] Sidebar shows "Ledger" with BookOpen icon
- [ ] Mobile bottom nav shows "Ledger" with BookOpen icon
- [ ] Routes still point to `/app/wallet` (no URL changes)
- [ ] Active state highlighting still works

---

## Task 3.2 — Frontend: Rename Wallet Page Headers & Content

### Description
Update all user-facing text on the wallet page, wallet transactions, deposit, and withdraw pages.

### Files to Modify
- [`src/app/app/wallet/page.tsx`](file:///Users/oluwatobiloba/Desktop/qsd/vescrow/vescrow_frontend/src/app/app/wallet/page.tsx)
- [`src/app/app/wallet/transactions/page.tsx`](file:///Users/oluwatobiloba/Desktop/qsd/vescrow/vescrow_frontend/src/app/app/wallet/transactions)
- [`src/app/app/wallet/deposit/page.tsx`](file:///Users/oluwatobiloba/Desktop/qsd/vescrow/vescrow_frontend/src/app/app/wallet/deposit)
- [`src/app/app/wallet/withdraw/page.tsx`](file:///Users/oluwatobiloba/Desktop/qsd/vescrow/vescrow_frontend/src/app/app/wallet/withdraw)

### Implementation Details

**`wallet/page.tsx`** — Key text replacements:
```diff
-  title="Wallet & Balance"
-  description="Cleared funds ready for instant NIBSS bank withdrawal"
+  title="Ledger"
+  description="Your double-entry ledger — cleared funds ready for instant NIBSS withdrawal"

-  <span className="...">Providus Bank Vault Balance</span>
+  <span className="...">Ledger Balance</span>

-  <h3 className="...">Recent Ledger Activity</h3>
+  <h3 className="...">Recent Activity</h3>
   {/* Already says "Ledger", just clean up redundancy */}
```

**Transaction/Deposit/Withdraw pages** — Scan for any "Wallet" text and replace with "Ledger":
- Page titles
- Breadcrumbs
- Description text
- Button labels

### Validation Criteria
- [ ] `/app/wallet` page header says "Ledger"
- [ ] Balance card says "Ledger Balance" not "Wallet Balance"
- [ ] All sub-pages (transactions, deposit, withdraw) say "Ledger"
- [ ] No user-visible "Wallet" text remains (except in code identifiers)

---

## Task 3.3 — Frontend: Rename Wallet References in Escrow Pages

### Description
Update any references to "Wallet" in escrow-related pages and components.

### Files to Modify (search & replace)
- [`src/app/app/escrows/[id]/page.tsx`](file:///Users/oluwatobiloba/Desktop/qsd/vescrow/vescrow_frontend/src/app/app/escrows/%5Bid%5D/page.tsx)
- [`src/app/app/overview/page.tsx`](file:///Users/oluwatobiloba/Desktop/qsd/vescrow/vescrow_frontend/src/app/app/overview)
- [`src/components/fintech/stat-card.tsx`](file:///Users/oluwatobiloba/Desktop/qsd/vescrow/vescrow_frontend/src/components/fintech/stat-card.tsx)

### Implementation Details
Run a project-wide search for user-facing "Wallet" strings (not code identifiers like `useWalletStore` or `wallet-store.ts`) and replace with "Ledger":

```bash
# Search for user-facing wallet text (not imports/code)
grep -rn "Wallet" src/ --include="*.tsx" | grep -v "import\|from\|useWallet\|wallet-store\|wallet_\|walletApi\|WalletStore\|WalletState\|WalletType\|WalletIcon"
```

### Validation Criteria
- [ ] Global search for user-facing "Wallet" text returns zero results
- [ ] Code identifiers (store names, API paths, variable names) remain unchanged
- [ ] No TypeScript errors

---

## Task 3.4 — Frontend: Update ROUTES Constants Display Names (Optional)

### Description
While the route paths stay the same, add display name constants for breadcrumb/title use.

### Files to Modify
- [`src/lib/constants.ts`](file:///Users/oluwatobiloba/Desktop/qsd/vescrow/vescrow_frontend/src/lib/constants.ts)

### Implementation Details
Add display name mapping:
```typescript
export const ROUTE_DISPLAY_NAMES: Record<string, string> = {
  [ROUTES.WALLET]: 'Ledger',
  [ROUTES.WALLET_TRANSACTIONS]: 'Ledger Transactions',
  [ROUTES.WALLET_WITHDRAW]: 'Withdraw from Ledger',
  [ROUTES.WALLET_DEPOSIT]: 'Deposit to Ledger',
};
```

### Validation Criteria
- [ ] Display names are available for breadcrumb components
- [ ] Route paths unchanged

---

## Phase 3 Validation Criteria

- [ ] Zero user-visible instances of "Wallet" remain in the entire frontend
- [ ] Sidebar navigation shows "Ledger" with BookOpen icon
- [ ] Mobile bottom nav shows "Ledger"
- [ ] All wallet pages have "Ledger" in their headers
- [ ] Code identifiers (stores, API, types) remain as `wallet` — no unnecessary refactoring
- [ ] No broken links — all routes still work as `/app/wallet/*`
- [ ] No TypeScript compilation errors
