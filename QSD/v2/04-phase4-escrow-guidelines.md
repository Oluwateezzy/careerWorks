# Phase 4 — Escrow Guidelines System

> **Goal**: Create a detailed escrow guideline document that users must agree to before creating or accepting escrows. The guidelines should be accessible at every point in the app.

---

## Task 4.1 — Frontend: Create Guidelines Content & Component

### Description
Create a comprehensive escrow guidelines component that renders the full guideline document. This is a shared component used across the app.

### Files to Create
- `src/components/shared/escrow-guidelines.tsx` — The guideline content component
- `src/components/shared/escrow-guidelines-modal.tsx` — Modal wrapper with agreement checkbox

### Implementation Details

**Guidelines Content** (`escrow-guidelines.tsx`):
The component renders a scrollable, well-formatted guideline document covering:

1. **What is Escrow?** — How Vescrow protects both buyer and seller
2. **Buyer Responsibilities**
   - Provide accurate transaction details
   - Fund escrow promptly at creation
   - Verify delivery within the inspection period
   - Communicate through the platform
   - Report issues through disputes (not chargebacks)
3. **Seller Responsibilities**
   - Accept or reject terms within 72 hours
   - Submit pre-shipment evidence (photos, serial numbers)
   - Ship using trackable courier services
   - Provide accurate tracking information
   - Respond to disputes with evidence
4. **Inspection Period**
   - Purpose and duration
   - What to inspect
   - Consequences of not verifying (auto-release + trust penalty)
5. **Dispute Process**
   - When to file a dispute
   - Required evidence (photos, videos)
   - Arbitration timeline
   - Possible resolutions
6. **Trust & Rating System**
   - How trust scores work
   - Actions that affect your score
   - Consequences of low scores
7. **Fees & Charges**
   - Incoming fee (3%) — paid by buyer
   - Outgoing fee (3%) — deducted on release
   - No hidden charges
8. **Prohibited Activities**
   - Fraudulent transactions
   - Off-platform payments
   - Chargeback abuse
9. **Account Penalties**
   - Score thresholds (good ≥70, warning ≥40, red ≥20, blocked <20)
   - Appeal process

```tsx
export const EscrowGuidelines: React.FC = () => (
  <div className="prose prose-invert prose-sm max-w-none">
    <h2>Vescrow Escrow Guidelines</h2>
    <p className="text-slate-400 text-xs">Last updated: October 2026</p>
    {/* ... structured sections ... */}
  </div>
);
```

**Guidelines Modal** (`escrow-guidelines-modal.tsx`):
```tsx
interface EscrowGuidelinesModalProps {
  isOpen: boolean;
  onClose: () => void;
  onAgree?: () => void;
  requireAgreement?: boolean;
}
```

When `requireAgreement` is true, shows a checkbox "I have read and agree to the escrow guidelines" and an "Agree & Continue" button. Without agreement, the modal can be dismissed normally.

### Validation Criteria
- [ ] Guidelines component renders all 9 sections
- [ ] Modal displays scrollable guidelines with proper formatting
- [ ] When `requireAgreement=true`, "Agree" button is disabled until checkbox is checked
- [ ] Modal animates in/out properly
- [ ] Content is readable on both desktop and mobile

---

## Task 4.2 — Frontend: Integrate Guidelines into Create Escrow Flow

### Description
Require the buyer to agree to guidelines before creating an escrow.

### Files to Modify
- [`src/app/app/escrows/create/page.tsx`](file:///Users/oluwatobiloba/Desktop/qsd/vescrow/vescrow_frontend/src/app/app/escrows/create/page.tsx)

### Implementation Details
Add a guidelines agreement step at the beginning of the create flow:

```tsx
const [guidelinesAccepted, setGuidelinesAccepted] = useState(false);
const [showGuidelinesModal, setShowGuidelinesModal] = useState(false);

// Before the form, show a banner if not yet accepted
{!guidelinesAccepted && (
  <Card className="...">
    <p>Before creating an escrow, please review and agree to our escrow guidelines.</p>
    <Button onClick={() => setShowGuidelinesModal(true)}>
      Read Escrow Guidelines
    </Button>
  </Card>
)}

// The form is disabled until guidelines are accepted
<form onSubmit={handleSubmit} className={!guidelinesAccepted ? 'opacity-50 pointer-events-none' : ''}>
  ...
</form>

<EscrowGuidelinesModal
  isOpen={showGuidelinesModal}
  onClose={() => setShowGuidelinesModal(false)}
  requireAgreement
  onAgree={() => {
    setGuidelinesAccepted(true);
    setShowGuidelinesModal(false);
  }}
/>
```

Persist the agreement in `localStorage` so the user doesn't have to re-agree every time (but they can always re-read via the sidebar link).

### Validation Criteria
- [ ] Create escrow form is disabled until guidelines are accepted
- [ ] Guidelines modal opens with "Read Escrow Guidelines" button
- [ ] Checkbox must be checked to proceed
- [ ] Agreement persisted in localStorage
- [ ] Returning users bypass the agreement step (but can re-read anytime)

---

## Task 4.3 — Frontend: Integrate Guidelines into Accept Escrow Flow

### Description
When a seller clicks "Accept Terms" on a pending escrow, show the guidelines modal with agreement requirement.

### Files to Modify
- [`src/app/app/escrows/[id]/page.tsx`](file:///Users/oluwatobiloba/Desktop/qsd/vescrow/vescrow_frontend/src/app/app/escrows/%5Bid%5D/page.tsx)
- [`src/components/escrow/escrow-invite-view.tsx`](file:///Users/oluwatobiloba/Desktop/qsd/vescrow/vescrow_frontend/src/components/escrow/escrow-invite-view.tsx)

### Implementation Details
Intercept the "Accept Terms" action:

```tsx
const handleAcceptTerms = () => {
  const hasAgreed = localStorage.getItem('vescrow_guidelines_agreed');
  if (!hasAgreed) {
    setShowGuidelinesModal(true);
    return;
  }
  acceptTerms(currentEscrow.id);
};

<EscrowGuidelinesModal
  isOpen={showGuidelinesModal}
  onClose={() => setShowGuidelinesModal(false)}
  requireAgreement
  onAgree={() => {
    localStorage.setItem('vescrow_guidelines_agreed', 'true');
    setShowGuidelinesModal(false);
    acceptTerms(currentEscrow.id);
  }}
/>
```

### Validation Criteria
- [ ] First-time sellers must agree to guidelines before accepting
- [ ] Agreement persisted — subsequent accepts proceed immediately
- [ ] Guidelines modal appears on the invite accept page as well

---

## Task 4.4 — Frontend: Add Permanent Guidelines Access Points

### Description
Add a permanent "Escrow Guidelines" link in the sidebar and make it accessible from various points in the app.

### Files to Modify
- [`src/components/layout/app-sidebar.tsx`](file:///Users/oluwatobiloba/Desktop/qsd/vescrow/vescrow_frontend/src/components/layout/app-sidebar.tsx)
- [`src/components/layout/mobile-bottom-nav.tsx`](file:///Users/oluwatobiloba/Desktop/qsd/vescrow/vescrow_frontend/src/components/layout/mobile-bottom-nav.tsx)
- [`src/app/app/escrows/[id]/page.tsx`](file:///Users/oluwatobiloba/Desktop/qsd/vescrow/vescrow_frontend/src/app/app/escrows/%5Bid%5D/page.tsx)
- [`src/components/layout/footer.tsx`](file:///Users/oluwatobiloba/Desktop/qsd/vescrow/vescrow_frontend/src/components/layout/footer.tsx)

### Implementation Details

**Sidebar** — Add a guidelines button after the nav items:
```tsx
// Add to navItems or as a separate element below nav
<button
  onClick={() => setShowGuidelinesModal(true)}
  className="flex items-center gap-3.5 px-3.5 py-3 rounded-[12px] text-sm font-medium text-[var(--text-sidebar)] hover:bg-white/5 hover:text-white transition-all"
>
  <FileText className="w-5 h-5 text-[#94A3B8]" />
  {!isCollapsed && <span>Escrow Guidelines</span>}
</button>
```

Since the sidebar is a client component and the modal needs state, either:
- Use a global state/context for the guidelines modal
- Or make it a separate page at `/app/guidelines`

**Recommended**: Create a dedicated page at `/app/guidelines` and link to it from everywhere. This is simpler and more accessible.

**Create** `src/app/app/guidelines/page.tsx`:
```tsx
export default function GuidelinesPage() {
  return (
    <div className="max-w-3xl mx-auto">
      <PageHeader title="Escrow Guidelines" description="..." />
      <Card><EscrowGuidelines /></Card>
    </div>
  );
}
```

**Escrow Detail Page** — Add an info icon/link:
```tsx
<Link href="/app/guidelines" className="text-xs text-emerald-400 hover:underline">
  📋 View Escrow Guidelines
</Link>
```

**Mobile "More" Menu** — Add guidelines link.

**Footer** — Add "Escrow Guidelines" to footer links.

### Validation Criteria
- [ ] Sidebar has "Escrow Guidelines" link
- [ ] Mobile "More" menu has "Escrow Guidelines" link
- [ ] Escrow detail page has guidelines link
- [ ] Footer has guidelines link
- [ ] `/app/guidelines` page renders full guidelines content
- [ ] Guidelines are accessible from every major page in max 1 click

---

## Phase 4 Validation Criteria

- [ ] Guidelines component renders all required sections
- [ ] Guidelines must be agreed to before creating an escrow
- [ ] Guidelines must be agreed to before accepting an escrow (first time)
- [ ] Guidelines accessible from sidebar, mobile nav, escrow detail, footer
- [ ] Agreement state persisted in localStorage
- [ ] Guidelines page accessible at `/app/guidelines`
- [ ] Content is properly formatted and readable on all screen sizes
