# Phase 5 — Theme Removal & Sidebar Cleanup

> **Goal**: Remove the 100-theme variation system entirely. Remove KYC & Limits, Notifications, and Settings **from the sidebar only** to avoid redundancy — these pages remain accessible via the navbar/topbar.

---

## Task 5.1 — Frontend: Remove 100-Theme System

### Description
Delete the entire theme variation system including the themes data file, theme switcher component, and all references.

### Files to Delete
- `src/lib/themes.ts` — 1,652 lines of theme definitions
- `src/components/shared/theme-switcher.tsx` — Theme switcher component

### Files to Modify (Remove Imports/References)
Search and remove all references to:
- `HUNDRED_THEMES`
- `THEME_CATEGORIES`
- `ThemePreset`
- `theme-switcher`
- `ThemeSwitcher`
- `applyThemePreset`

### Files Known to Reference Themes
- `src/components/shared/theme-switcher.tsx` — **DELETE**
- Any layout or topbar component that renders `<ThemeSwitcher />`

### Search Command
```bash
grep -rn "theme-switcher\|HUNDRED_THEMES\|THEME_CATEGORIES\|ThemePreset\|ThemeSwitcher\|applyThemePreset" src/ --include="*.tsx" --include="*.ts"
```

### Implementation Details
1. Delete `src/lib/themes.ts`
2. Delete `src/components/shared/theme-switcher.tsx`
3. Search for all imports of these deleted modules and remove them
4. Remove any `<ThemeSwitcher />` JSX usage from layouts/topbars
5. Verify no runtime errors

### Validation Criteria
- [ ] `src/lib/themes.ts` deleted
- [ ] `src/components/shared/theme-switcher.tsx` deleted
- [ ] Zero references to deleted files remain in codebase
- [ ] No TypeScript compilation errors
- [ ] App renders correctly with hardcoded dark theme
- [ ] No console errors related to missing theme modules
- [ ] Bundle size reduced (53KB themes file removed)

---

## Task 5.2 — Frontend: Remove KYC & Limits from Sidebar Only

### Description
Remove "KYC & Limits" from the sidebar navigation and mobile "More" menu **only**. The KYC pages, store, API module, and all associated code remain untouched — they are still accessible via the navbar/topbar to avoid redundancy in the sidebar.

### Files to Modify
- [`src/components/layout/app-sidebar.tsx`](file:///Users/oluwatobiloba/Desktop/qsd/vescrow/vescrow_frontend/src/components/layout/app-sidebar.tsx)
  - Remove from `navItems` array: `{ href: ROUTES.KYC_STATUS, label: 'KYC & Limits', icon: Award }`
  - Remove `Award` from lucide-react imports if no longer used elsewhere in this file
- [`src/components/layout/mobile-bottom-nav.tsx`](file:///Users/oluwatobiloba/Desktop/qsd/vescrow/vescrow_frontend/src/components/layout/mobile-bottom-nav.tsx)
  - Remove from `moreLinks` array: `{ href: ROUTES.KYC_STATUS, label: 'KYC & Limits', icon: Award }`
  - Remove `Award` from lucide-react imports if no longer used elsewhere in this file

### What Stays Untouched
- `src/app/app/kyc/` — All KYC pages remain (verify, documents, status)
- `src/stores/kyc-store.ts` — KYC store remains
- `src/lib/api/kyc.ts` — KYC API module remains
- `src/app/app/layout.tsx` — `useKYCStore` and `fetchKYCStatus()` remain
- `src/lib/constants.ts` — KYC routes and `KYC_STATUS_CONFIG` remain
- `src/lib/types.ts` — All KYC types remain

### Validation Criteria
- [ ] "KYC & Limits" no longer appears in sidebar
- [ ] "KYC & Limits" no longer appears in mobile "More" menu
- [ ] `/app/kyc/*` routes still work and are accessible via navbar
- [ ] KYC store, API, pages, types all remain functional
- [ ] No TypeScript compilation errors

---

## Task 5.3 — Frontend: Remove Notifications from Sidebar Only

### Description
Remove "Notifications" from the sidebar navigation and mobile "More" menu **only**. The notifications page, store, and API remain fully functional — accessible via the navbar/topbar.

### Files to Modify
- [`src/components/layout/app-sidebar.tsx`](file:///Users/oluwatobiloba/Desktop/qsd/vescrow/vescrow_frontend/src/components/layout/app-sidebar.tsx)
  - Remove from `navItems` array: `{ href: ROUTES.NOTIFICATIONS, label: 'Notifications', icon: Bell, badge: unreadCount }`
  - Remove `Bell` from lucide-react imports if no longer used elsewhere in this file
  - Remove `useNotificationStore` import if no longer used elsewhere in this file
- [`src/components/layout/mobile-bottom-nav.tsx`](file:///Users/oluwatobiloba/Desktop/qsd/vescrow/vescrow_frontend/src/components/layout/mobile-bottom-nav.tsx)
  - Remove from `moreLinks` array: `{ href: ROUTES.NOTIFICATIONS, label: 'Notifications', icon: Bell }`

### What Stays Untouched
- `src/app/app/notifications/page.tsx` — Notifications page remains
- `src/stores/notification-store.ts` — Store remains (used by topbar/mobile-header for badge)
- `src/lib/api/notifications.ts` — API module remains
- `src/lib/constants.ts` — `NOTIFICATIONS` route remains

### Validation Criteria
- [ ] "Notifications" no longer appears in sidebar
- [ ] "Notifications" no longer appears in mobile "More" menu
- [ ] `/app/notifications` still works and is accessible via navbar
- [ ] Notification bell icon + unread badge still visible in topbar/mobile-header
- [ ] No TypeScript compilation errors

---

## Task 5.4 — Frontend: Remove Settings from Sidebar Only

### Description
Remove "Settings" from the sidebar navigation and mobile "More" menu **only**. The settings page remains fully functional — accessible via the navbar/topbar.

### Files to Modify
- [`src/components/layout/app-sidebar.tsx`](file:///Users/oluwatobiloba/Desktop/qsd/vescrow/vescrow_frontend/src/components/layout/app-sidebar.tsx)
  - Remove from `navItems` array: `{ href: ROUTES.SETTINGS, label: 'Settings', icon: Settings }`
  - Remove `Settings` from lucide-react imports if no longer used elsewhere in this file
- [`src/components/layout/mobile-bottom-nav.tsx`](file:///Users/oluwatobiloba/Desktop/qsd/vescrow/vescrow_frontend/src/components/layout/mobile-bottom-nav.tsx)
  - Remove from `moreLinks` array: `{ href: ROUTES.SETTINGS, label: 'Settings', icon: Settings }`

### What Stays Untouched
- `src/app/app/settings/page.tsx` — Settings page remains
- `src/lib/constants.ts` — `SETTINGS` route remains

### Validation Criteria
- [ ] "Settings" no longer appears in sidebar
- [ ] "Settings" no longer appears in mobile "More" menu
- [ ] `/app/settings` still works and is accessible via navbar
- [ ] No TypeScript compilation errors

---

## Task 5.5 — Frontend: Final Dead Code Sweep (Theme-only)

### Description
Run a comprehensive search for any remaining imports or references to the **deleted theme system only**. No other modules are being deleted.

### Implementation Details
Run these searches and verify zero results:

```bash
# Deleted theme modules only
grep -rn "theme-switcher\|HUNDRED_THEMES\|THEME_CATEGORIES\|ThemePreset\|applyThemePreset" src/ --include="*.tsx" --include="*.ts"
```

Also verify:
- `src/components/shared/` no longer has `theme-switcher.tsx`
- No `<ThemeSwitcher />` usage anywhere

### Validation Criteria
- [ ] Zero dangling imports to deleted theme modules
- [ ] No runtime errors when navigating through the app
- [ ] All remaining tests pass
- [ ] Build succeeds without errors: `npm run build`

---

## Phase 5 Validation Criteria

- [ ] 100-theme system completely removed (~53KB bundle reduction)
- [ ] Sidebar shows only: Overview, Escrow Deals, Share Listings, Ledger, Escrow Guidelines
- [ ] Mobile bottom nav shows only: Home, Escrows, Ledger, More
- [ ] Mobile "More" menu shows only: Listings, Escrow Guidelines, Log out
- [ ] KYC pages **still work** — just not in sidebar (accessible via navbar)
- [ ] Notifications page **still works** — just not in sidebar (accessible via navbar)
- [ ] Settings page **still works** — just not in sidebar (accessible via navbar)
- [ ] Zero dead theme imports or references
- [ ] `npm run build` succeeds
- [ ] No console errors in browser
