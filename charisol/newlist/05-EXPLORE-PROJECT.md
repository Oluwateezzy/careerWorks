# Phase 5 — Explore Project Page Analysis & Mock Data Removal

> **Priority**: 🔴 High (User-facing explore page accuracy)
> **Estimated Effort**: 1–1.5 days
> **Dependencies**: None (Uses existing `getPublicProject` endpoint data)

---

## Background

The Explore Project page (`/explore/project/[id]`, e.g. `https://strata.charisol.io/explore/project/019ed1ba-74da-7529-aca0-54433a737ad6`) is the public showpiece of Strata design systems. Currently:

1. **`DETAIL_SAFE_FIELDS` in backend `publicController.js` excludes AI brand identity and logo data**: `brandIdentity`, `brandContext`, `logo`, `color`, and `connectedSurfaces` are stripped out before sending to the client.
2. **Hardcoded `DICT_TOKENS` mock array** is used in the Tokens tab instead of reading `project.designSystemData.variables`.
3. **Hardcoded `HANDOFF_CODE` static snippets** are used in the Components & Spec tab instead of dynamically formatting the project's actual design tokens.
4. **Hardcoded typography names (`Outfit`, `Inter`) and mood keywords (`Clean`, `Minimalist`, `Explore`)** are shown in the Overview and Brand System tabs instead of pulling from `project.brandIdentity`.
5. **Hardcoded package install command (`npm i @strata-ds/my-design`)** is displayed instead of deriving from project slug/title and version.

---

## Tasks

### RV-500 — Expose `brandIdentity`, `brandContext`, `logo`, `color`, and `connectedSurfaces` in backend `getPublicProject`

**Files**: [`publicController.js`](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-be/controllers/publicController.js#L9)

**Changes**:
- Update `DETAIL_SAFE_FIELDS` array in `publicController.js`:
  ```js
  const DETAIL_SAFE_FIELDS = [
    "projectId", "title", "description", "logo", "color",
    "brandIdentity", "brandContext", "connectedSurfaces",
    "snapshotCdnBase", "versions", "defaultVersion",
    "updatedAt", "createdAt"
  ];
  ```
- Ensure `getPublicProject` returns `brandIdentity`, `brandContext`, `logo`, `color`, and `connectedSurfaces` if present on the project record.

**Validation Criteria**:
- [ ] GET `/v1/public/projects/:projectId` includes `brandIdentity` object in response
- [ ] GET `/v1/public/projects/:projectId` includes `logo` and `color` in response
- [ ] Private/sensitive fields (e.g. `userId`, `webhookToken`) remain omitted

---

### RV-501 — Dynamic Token Dictionary from `rawVariables` in `app/explore/project/[id]/page.tsx`

**Files**: [`app/explore/project/[id]/page.tsx`](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-fe/app/explore/project/%5Bid%5D/page.tsx#L46-L88)

**Changes**:
- Remove hardcoded `DICT_TOKENS` constant.
- Create memoized `dictTokens` selector derived from `rawVariables`:
  ```ts
  const dictTokens = useMemo(() => {
    return rawVariables.map((v: any) => {
      const name = String(v.key || v.name || "");
      const value = String(v.value || "");
      const type = String(v.type || inferTypeFromName(name));
      const tier = (v.tier || inferTierFromName(name)) as "BRAND" | "SEMANTIC" | "COMPONENT";
      return { name, value, type, tier, swatch: type === "color" ? value : undefined };
    });
  }, [rawVariables]);
  ```
- Update `filteredDictTokens` to filter over `dictTokens` instead of `DICT_TOKENS`.
- Handle empty variables array with a clean empty state message ("No design tokens found for this project").

**Validation Criteria**:
- [ ] Tokens tab renders the real variables from `project.designSystemData.variables`
- [ ] Tier filters ("Brand Core", "Semantic", "Component Tier") filter real tokens accurately
- [ ] Category filters ("Color", "Typography", "Spacing", etc.) match token types accurately
- [ ] Zero mock data used in token dictionary table

---

### RV-502 — Dynamic Handoff Code generator from `rawVariables`

**Files**: [`app/explore/project/[id]/page.tsx`](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-fe/app/explore/project/%5Bid%5D/page.tsx#L90-L164)

**Changes**:
- Remove hardcoded `HANDOFF_CODE` constant.
- Create dynamic generators `generateCssCode(vars)`, `generateTailwindCode(vars)`, and `generateJsonCode(vars)`.
- Generate actual `:root { --variable-name: value; }` for CSS tab.
- Generate actual Tailwind theme extension JSON for Tailwind tab.
- Generate valid W3C Design Tokens format JSON for JSON tab.
- Wrap in `useMemo` based on `rawVariables`.

**Validation Criteria**:
- [ ] CSS code block outputs exact CSS variables derived from project's `rawVariables`
- [ ] Tailwind code block outputs valid theme config JSON using project's color and typography tokens
- [ ] JSON code block outputs valid W3C design tokens JSON format
- [ ] Copy button copies the dynamically generated code to clipboard

---

### RV-503 — Dynamic Brand Typography, Mood Keywords, & Personality in Overview and Brand System tabs

**Files**: [`app/explore/project/[id]/page.tsx`](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-fe/app/explore/project/%5Bid%5D/page.tsx#L616-L645)

**Changes**:
- Heading Font: extract from `project.brandIdentity?.typography?.heading?.family` or font variable in `rawVariables` (fallback to "Outfit" if unassigned).
- Body Font: extract from `project.brandIdentity?.typography?.body?.family` or font variable in `rawVariables` (fallback to "Inter" if unassigned).
- Mood Keywords: extract from `project.brandIdentity?.moodKeywords` array (fallback to keywords derived from project tags).
- Personality/Voice: extract from `project.brandIdentity?.personality` or `project.brandIdentity?.voice` string.

**Validation Criteria**:
- [ ] Typography previews render using the project's real font family names
- [ ] Mood keyword pills render real keywords from `project.brandIdentity`
- [ ] Brand voice quote displays real personality text from `project.brandIdentity`

---

### RV-504 — Dynamic Package Name & Info in Explore Sidebar

**Files**: [`app/explore/project/[id]/page.tsx`](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-fe/app/explore/project/%5Bid%5D/page.tsx#L1128-L1165)

**Changes**:
- Package Install Code: format as `npm i @strata-ds/${slugifiedTitle}` (e.g. `npm i @strata-ds/my-design-system`).
- Current Version: display `v${(project.designSystemData as any)?.metadata?.version || project.snapshotLatestVersion || "1.0.0"}`.
- Live Sites Using It: display `project.connectedSurfaces?.length || 1` Connected.

**Validation Criteria**:
- [ ] Package install command reflects the project's actual title/slug
- [ ] Version tag matches project version
- [ ] Copy button copies the dynamic package command to clipboard

---

### RV-505 — End-to-End Verification & Build Test

**Scope**: Verification & Code Quality

**Steps**:
1. Open `https://strata.charisol.io/explore/project/019ed1ba-74da-7529-aca0-54433a737ad6` (or local equivalent `http://localhost:3000/explore/project/019ed1ba-74da-7529-aca0-54433a737ad6`).
2. Verify all tabs ("Overview", "Brand System", "Tokens", "Components & Spec") load without TypeScript or runtime console errors.
3. Verify zero hardcoded `DICT_TOKENS` or hardcoded `HANDOFF_CODE` constants remain in `app/explore/project/[id]/page.tsx`.
4. Run `yarn build` and confirm production build succeeds with exit code 0.

**Validation Criteria**:
- [ ] `yarn build` passes cleanly with zero errors
- [ ] Explore page renders project data dynamically without fallback mock data
- [ ] All interactive elements (copy code, tabs, persona filters, sandbox controls) work cleanly
