# Manual UI Test Guide: Multi-Font Support & Font Integration

This guide provides step-by-step manual validation procedures for testing the **Phase 3 & 3B Multi-Font Support & Font Integration** features in the Strata Design System. It covers Google Fonts API integration, custom font file uploads, wizard typeface selection, token modal font pickers, Brand Context Engine overrides, and the Font Manager library.

---

## 📋 Table of Contents

1. [Prerequisites & Environment Setup](#1-prerequisites--environment-setup)
2. [Feature Touchpoints Overview](#2-feature-touchpoints-overview)
3. [Step-by-Step Test Scenarios](#3-step-by-step-test-scenarios)
   - [Scenario 1: Google Fonts Catalog Search & Dynamic Preview](#scenario-1-google-fonts-catalog-search--dynamic-preview)
   - [Scenario 2: Custom Font Upload & S3 Sync](#scenario-2-custom-font-upload--s3-sync)
   - [Scenario 3: Build from Scratch Wizard Typeface Selection](#scenario-3-build-from-scratch-wizard-typeface-selection)
   - [Scenario 4: Add & Edit Design Token Font Picker](#scenario-4-add--edit-design-token-font-picker)
   - [Scenario 5: Component-Scoped Token Font Picker](#scenario-5-component-scoped-token-font-picker)
   - [Scenario 6: Brand Context Review Font Override](#scenario-6-brand-context-review-font-override)
   - [Scenario 7: Font Manager Library & Font Deletion](#scenario-7-font-manager-library--font-deletion)
   - [Scenario 8: Offline / Missing API Key Fallback](#scenario-8-offline--missing-api-key-fallback)
4. [Visual Checklist & Success Criteria](#4-visual-checklist--success-criteria)

---

## 1. Prerequisites & Environment Setup

### Step A: Configure Backend
1. Open [charisol-design-system-be](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-be) and verify `.env`:
   ```ini
   PORT=5000
   AWS_REGION=us-east-1
   S3_BUCKET_NAME=strata-font-storage
   DYNAMODB_USER_TABLE_NAME=Users
   ```
2. Start the backend:
   ```bash
   npm run dev
   ```

### Step B: Configure Frontend
1. Open [charisol-design-system-fe](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-fe) and verify `.env.local`:
   ```ini
   NEXT_PUBLIC_GOOGLE_FONTS_API_KEY=AIzaSyDLpWAHLDyjbvSMPs5Jqlmj9aevVtDVU-M
   NEXT_PUBLIC_API_URL=http://localhost:5000/v1
   ```
2. Start the frontend:
   ```bash
   npm run dev
   ```
3. Open `http://localhost:3000` in Google Chrome or Safari.

---

## 2. Feature Touchpoints Overview

| Touchpoint Component | File Path | Key Functionality Tested |
|----------------------|-----------|--------------------------|
| **FontPicker** | `components/ui/FontPicker.tsx` | Searchable dropdown, Google Fonts catalog, Recently Used, Brand Fonts, Custom Uploads |
| **FontUploadModal** | `components/modals/project/FontUploadModal.tsx` | Drag-and-drop file upload (`.woff2`, `.ttf`, `.otf`, `.woff`), pre-upload preview |
| **BuildFromScratchWizard** | `components/modals/project/BuildFromScratchWizard.tsx` | Preset font pairings + custom heading/body font pickers |
| **AddVariableModal** | `components/modals/project/AddVariableModal.tsx` | Dynamic `<FontPicker>` for `fontfamily` / `font family` tokens |
| **EditTokenModal** | `components/modals/project/EditTokenModal.tsx` | Dynamic `<FontPicker>` for editing existing font tokens |
| **ComponentsSection** | `components/sections/project/ComponentsSection.tsx` | Component-scoped token creation modal font picker |
| **BrandContextModal** | `components/modals/brand-context/BrandContextModal.tsx` | Review step typography scale inline font picker overrides |
| **FontManager** | `components/sections/settings/FontManager.tsx` | Font library overview, custom font deletion, sample preview |

---

## 3. Step-by-Step Test Scenarios

### Scenario 1: Google Fonts Catalog Search & Dynamic Preview
**Goal**: Verify searching for Google Fonts renders live catalog results and dynamically loads font styles into the browser `<head>`.

1. Open any project and navigate to **Tokens** tab.
2. Click **+ Add Token**.
3. Select **CSS Property Name**: `font-family`.
4. In the **Token Value** field, click the **FontPicker** dropdown.
5. In the search input, type `Outfit`.
6. **Verification**:
   - The dropdown filters immediately to show `Outfit` with a `Google` badge.
   - The dropdown row text renders in the actual `Outfit` font style.
7. Click `Outfit` to select it.
8. **Verification**:
   - Inspect `<head>` in Chrome DevTools: a `<link rel="stylesheet">` for `https://fonts.googleapis.com/css2?family=Outfit...` should be dynamically injected.
   - The Visual Preview card at the bottom of the modal updates instantly to display preview text rendered in `Outfit`.

---

### Scenario 2: Custom Font Upload & S3 Sync
**Goal**: Verify uploading a custom `.woff2` or `.ttf` font persists to S3 and DynamoDB and becomes available across projects.

1. In the **FontPicker** dropdown, click **+ Upload Custom Font** (or navigate to **Settings > Fonts** tab).
2. The **Font Upload Modal** appears.
3. Drag and drop a valid custom font file (e.g., `CustomBrandFont.woff2`, size < 2MB).
4. **Verification**:
   - Live pre-upload preview box immediately renders sample text using a temporary Blob URL.
   - File details display correct file size and format tag (`WOFF2`).
5. Optionally edit the display name to `My Brand Bold`.
6. Click **Upload 1 Font**.
7. **Verification**:
   - Upload progress bar animates to 100%.
   - A success toast appears: *"Font uploaded successfully!"*.
   - The new font `My Brand Bold` appears in **FontPicker** under the **Custom Uploaded Fonts** section with an `Uploaded` badge.

---

### Scenario 3: Build from Scratch Wizard Typeface Selection
**Goal**: Verify project creation wizard supports both preset type pairings and custom font selection via FontPicker.

1. Click **+ New Project** from the main dashboard.
2. Choose **Start from Scratch**.
3. Progress to the **Type** step ("Choose your type").
4. Select one of the preset pairings (e.g., *Playfair Display + Source Sans 3*).
5. **Verification**: Specimen cards render live in Playfair Display and Source Sans 3.
6. Scroll down to **Your own typeface**.
7. Use the **Heading font** `FontPicker` to select `Syne`.
8. Use the **Body font** `FontPicker` to select `Plus Jakarta Sans`.
9. **Verification**:
   - The selection automatically switches to "Your own typeface".
   - Progress to **Review** step; verify typography summary lists `Syne + Plus Jakarta Sans`.
   - Click **Apply to project** and verify project tokens default to the chosen fonts.

---

### Scenario 4: Add & Edit Design Token Font Picker
**Goal**: Verify `AddVariableModal` and `EditTokenModal` seamlessly use FontPicker for typography tokens.

1. In **Tokens Section**, click **+ Add Token**.
2. Set Token Name: `font.heading.primary`, Property: `font-family`.
3. Select `Cabinet Grotesk` from FontPicker and click **Create Token**.
4. **Verification**: Token appears in the Token Tree with value `Cabinet Grotesk`.
5. Click the edit icon next to `font.heading.primary` to open `EditTokenModal`.
6. **Verification**:
   - The Token Value input displays `<FontPicker>` pre-filled with `Cabinet Grotesk`.
   - The Visual Preview card renders `Cabinet Grotesk`.
7. Change the font to `Clash Display` and click **Save Changes**.
8. **Verification**: Token value updates cleanly to `Clash Display`.

---

### Scenario 5: Component-Scoped Token Font Picker
**Goal**: Verify adding scoped tokens inside a component definition supports FontPicker.

1. Navigate to **Components** tab.
2. Select any component (e.g., `Button` or `Header`).
3. In the Component Details panel, click **+ Add Scoped Token**.
4. Set Property: `font-family`.
5. **Verification**: Token Value renders `<FontPicker>`.
6. Select `DM Sans` and confirm token creation.
7. **Verification**: Scoped component property updates to use `DM Sans`.

---

### Scenario 6: Brand Context Review Font Override
**Goal**: Verify overriding AI-extracted brand fonts during the Brand Context Engine review phase.

1. Open **Brand Context Modal** (or click **Extract Brand Context**).
2. Input a website URL or brand description and click **Analyze & Generate**.
3. Once analysis completes, progress to the **Review** step.
4. Click on the **Tokens** sub-tab and locate the **Typography scale** table.
5. **Verification**:
   - Font family tokens (e.g., `--font-family-base`) render an inline `<FontPicker>` in the **Value** column instead of static text.
6. Change `--font-family-base` from its extracted value to `Outfit`.
7. **Verification**:
   - The live preview component on the right side of the modal updates instantly to render using `Outfit`.
8. Click **Commit Brand Context**.
9. **Verification**: Project design tokens adopt `Outfit` as the primary base font.

---

### Scenario 7: Font Manager Library & Font Deletion
**Goal**: Manage uploaded fonts and project typography in the Font Manager settings view.

1. Navigate to **Settings > Fonts** tab (`FontManager.tsx`).
2. **Verification**:
   - Displays **Custom Uploaded Fonts** card grid and **Project Google Fonts** list.
   - Interactive preview text box allows typing custom sample text to test font rendering.
3. Find the uploaded font `My Brand Bold`.
4. Click the **Delete (Trash)** button.
5. A confirmation dialog appears: *"Delete Custom Font?"*.
6. Click **Confirm Delete**.
7. **Verification**:
   - Toast notification: *"Font deleted successfully"*.
   - Font is removed from S3/DynamoDB and disappears from all `FontPicker` dropdowns.

---

### Scenario 8: Offline / Missing API Key Fallback
**Goal**: Verify graceful fallback when Google Fonts API is unreachable or key is missing.

1. Temporarily clear `NEXT_PUBLIC_GOOGLE_FONTS_API_KEY` in `.env.local` and restart frontend.
2. Open `FontPicker`.
3. Type a query in search.
4. **Verification**:
   - FontPicker displays the 50-font popular fallback list (Inter, Roboto, Open Sans, Montserrat, Lora, etc.).
   - No runtime error or white screen occurs.

---

## 4. Visual Checklist & Success Criteria

- [ ] **FontPicker Search Responsiveness**: Search filters 1500+ fonts in < 100ms with zero UI lag.
- [ ] **Font Badges**: Rows clearly display badges: `Google` (blue), `Uploaded` (purple), `Brand` (amber), `Recent` (gray).
- [ ] **Real-Time Previews**: Specimen text inside FontPicker options and preview cards render using the exact font family via dynamic stylesheet injection.
- [ ] **Dynamic Injection**: Inspecting `<head>` shows `<link id="google-font-...">` tags loaded without duplicate tags for the same font family.
- [ ] **Cross-Browser Compatibility**: Custom uploaded `.woff2` and `.ttf` fonts render identically across Chrome, Firefox, and Safari.
