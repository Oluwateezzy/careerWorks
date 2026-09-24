# Phase 3 — Font Type: Multi-Font Support & Font Upload

> **Priority**: 🟡 Medium (Full stack, high user impact)
> **Estimated Effort**: 3–4 days
> **Dependencies**: Phase 2 (font extraction provides BCE-detected fonts)

---

## Background

### Current State
The DesignEditor ([`DesignEditor.tsx`](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-fe/components/sections/settings/DesignEditor.tsx)) has a **hardcoded list of 6 system fonts** at line ~669:
```js
const fontFamilyOptions = ["Arial", "Helvetica", "sans-serif", "Times New Roman", "Georgia", "serif"];
```

### Requirements
1. **Integrate fonts into Strata** — support Google Fonts catalog (1500+ fonts via API)
2. **Support multiple fonts** — users can select different fonts for heading/body/etc
3. **Ability to upload downloaded fonts** — `.woff2`, `.ttf`, `.otf`, `.woff` files
4. **Font scope**: User/org-scoped (same font reusable across projects)
5. **Font source**: Google Fonts API (live catalog, not static list)

### Architecture Decision
Fonts will be stored at the **user level** (not project level). The data model:
- **Google Fonts**: Loaded via Google Fonts CSS API at runtime. No storage needed — just the family name.
- **Custom uploaded fonts**: Uploaded to S3 under `fonts/{userId}/{fontId}.{ext}`, metadata stored in DynamoDB per user.
- **Font references in projects**: Design tokens reference font families by name (e.g., `"Inter"`, `"CustomBrand-Bold"`).

---

## Tasks

### RV-300 — Google Fonts API integration service (Frontend)

**Files**: **[NEW]** [`services/google-fonts.service.ts`](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-fe/services/google-fonts.service.ts)

**Changes**:
Create a service that:
- Fetches the Google Fonts catalog from `https://www.googleapis.com/webfonts/v1/webfonts?key={API_KEY}&sort=popularity`
- Caches the response in memory for 24 hours (avoid repeated API calls)
- Exposes `searchFonts(query: string, limit: number)` that filters by family name
- Exposes `getFontPreviewUrl(family: string, weight?: string)` → returns the Google Fonts CSS URL for `<link>` injection
- Exposes `loadFont(family: string)` → dynamically injects a `<link>` tag to load the font for preview
- Provides a fallback static list of 50 popular fonts if the API is unreachable

**Env variable**: `NEXT_PUBLIC_GOOGLE_FONTS_API_KEY`

**Validation Criteria**:
- [ ] `searchFonts("Inter", 10)` returns results including "Inter" within 500ms
- [ ] `loadFont("Outfit")` makes the font available for CSS rendering in the browser
- [ ] Works offline with fallback list when API is unreachable
- [ ] Results are cached — second call for same query doesn't hit API

---

### RV-301 — Font picker component

**Files**: **[NEW]** [`components/ui/FontPicker.tsx`](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-fe/components/ui/FontPicker.tsx)

**Changes**:
Create a searchable font picker component:
- Dropdown with search input at the top
- Lists fonts from 3 sources:
  1. **Recently used** (top section, from localStorage)
  2. **BCE-detected fonts** (from project `brandContext.typography`)
  3. **Google Fonts** (search results)
  4. **Custom uploaded fonts** (from user's uploaded font list)
- Each font row shows:
  - Font name in its own typeface (live preview via `loadFont()`)
  - Source badge: "Google", "Uploaded", "Brand"
- "Upload Font" action button at the bottom → opens `FontUploadModal`
- Props: `value: string`, `onChange: (family: string) => void`, `projectId?: string`

**Validation Criteria**:
- [ ] Typing "Rob" shows "Roboto", "Roboto Mono", "Roboto Slab" etc. within 300ms
- [ ] Each font row renders text in the actual font (not system font)
- [ ] Recently used fonts appear at the top of the dropdown
- [ ] "Upload Font" button opens the upload modal
- [ ] Selected font triggers `onChange` with the family name

---

### RV-302 — Font upload modal (Frontend)

**Files**: **[NEW]** [`components/modals/project/FontUploadModal.tsx`](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-fe/components/modals/project/FontUploadModal.tsx)

**Changes**:
Create upload modal that:
- Accepts `.woff2`, `.ttf`, `.otf`, `.woff` files via drag-and-drop or file picker
- Max file size: 2MB per file, max 5 files at once
- Shows upload progress per file
- Previews each font with sample text ("The quick brown fox jumps over the lazy dog") using `@font-face` injection
- Allows user to set a display name for each font
- On submit, calls `fontService.uploadFont()` for each file
- Shows success/error state per file

**Validation Criteria**:
- [ ] Drag-and-drop works for `.woff2` and `.ttf` files
- [ ] Files over 2MB are rejected with error message before upload
- [ ] Unsupported file types (`.png`, `.pdf`) are rejected
- [ ] Font preview renders correctly before upload is submitted
- [ ] Upload progress indicator shows for each file
- [ ] After successful upload, font appears in FontPicker immediately

---

### RV-303 — Font service (Frontend)

**Files**: **[NEW]** [`services/font.service.ts`](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-fe/services/font.service.ts)

**Changes**:
Create service with methods:
- `uploadFont(file: File, displayName: string): Promise<UploadedFont>` → POST `/api/fonts`
- `listFonts(): Promise<UploadedFont[]>` → GET `/api/fonts`
- `deleteFont(fontId: string): Promise<void>` → DELETE `/api/fonts/${fontId}`
- `injectFontFace(font: UploadedFont): void` — injects `@font-face` CSS rule into document for preview

Type:
```ts
interface UploadedFont {
  fontId: string;
  displayName: string;
  family: string;        // CSS font-family name
  format: "woff2" | "truetype" | "opentype" | "woff";
  url: string;           // S3 CDN URL
  fileSize: number;
  uploadedAt: string;
}
```

**Validation Criteria**:
- [ ] `uploadFont()` sends multipart/form-data to BFF
- [ ] `listFonts()` returns the user's uploaded fonts (org-scoped)
- [ ] `injectFontFace()` makes the font available for CSS rendering in the browser
- [ ] `deleteFont()` removes the font from backend and cleans up injected `@font-face`

---

### RV-304 — BFF proxy routes for font upload/list/delete

**Files**: **[NEW]** [`app/api/fonts/route.ts`](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-fe/app/api/fonts/route.ts)
**Files**: **[NEW]** [`app/api/fonts/[fontId]/route.ts`](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-fe/app/api/fonts/%5BfontId%5D/route.ts)

**Changes**:
- `GET /api/fonts` → proxies to backend `GET /fonts` with auth cookie forwarding
- `POST /api/fonts` → proxies multipart upload to backend `POST /fonts` with auth cookie
- `DELETE /api/fonts/:fontId` → proxies to backend `DELETE /fonts/:fontId`

**Validation Criteria**:
- [ ] Auth cookies are forwarded in all requests
- [ ] Multipart upload is correctly proxied (no body parsing — stream through)
- [ ] 401 responses trigger `strata:unauthorized` event
- [ ] Error responses forward the backend error message

---

### RV-305 — Backend font upload/list/delete endpoints

**Files**: **[NEW]** [`controllers/fontController.js`](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-be/controllers/fontController.js)

**Changes**:
Create controller with handlers:
- `uploadFont(req, res)`:
  - multer middleware for single file upload (field name: "file")
  - Validate: file type must be `.woff2`, `.ttf`, `.otf`, `.woff`; max 2MB
  - Generate `fontId` using `createId("fnt")`
  - Upload to S3: `fonts/{userId}/{fontId}.{ext}` with public-read ACL
  - Store metadata in user's DynamoDB record (append to `customFonts` array)
  - Return `{ data: { fontId, displayName, family, format, url, fileSize, uploadedAt } }`

- `listFonts(req, res)`:
  - Read `customFonts` array from user's DynamoDB record
  - Return `{ data: fonts[] }`

- `deleteFont(req, res)`:
  - Find font in user's `customFonts` by `fontId`
  - Delete from S3
  - Remove from DynamoDB array
  - Return `{ data: { fontId, deleted: true } }`

**Files**: **[MODIFY]** [`routes/v1/index.js`](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-be/routes/v1/index.js)
- Add routes: `POST /fonts`, `GET /fonts`, `DELETE /fonts/:fontId` (all authenticated)

**Validation Criteria**:
- [ ] Upload accepts `.woff2` file and returns CDN URL
- [ ] Upload rejects `.png` file with 400 error
- [ ] Upload rejects file > 2MB with 400 error
- [ ] List returns all user's fonts (empty array for new user)
- [ ] Delete removes font from S3 and DynamoDB
- [ ] Font S3 URL is publicly accessible for CSS `@font-face` src

---

### RV-306 — Replace hardcoded font list in DesignEditor

**Files**: [`DesignEditor.tsx`](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-fe/components/sections/settings/DesignEditor.tsx#L669)

**Changes**:
- Remove hardcoded `fontFamilyOptions` array
- Replace the `<select>` dropdown for font-family properties with `<FontPicker>` component
- Wire `onChange` to update the design token value
- Load BCE-detected fonts from project's `brandContext.typography` (if available)
- On font selection:
  - If Google Font: dynamically inject `<link>` tag so preview works
  - If uploaded font: inject `@font-face` rule so preview works

**Validation Criteria**:
- [ ] Font-family token editing uses `<FontPicker>` instead of `<select>`
- [ ] Selecting "Inter" from Google Fonts updates the token value to `"Inter"`
- [ ] The design preview renders text in the selected font
- [ ] BCE-detected fonts appear in the picker with "Brand" badge

---

### RV-307 — Font management section in Settings

**Files**: [`DesignEditor.tsx`](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-fe/components/sections/settings/DesignEditor.tsx) or **[NEW]** [`components/sections/settings/FontManager.tsx`](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-fe/components/sections/settings/FontManager.tsx)

**Changes**:
Add a "Fonts" section in the project settings or design editor:
- Shows list of all fonts available to the user:
  - Google Fonts used in this project
  - Uploaded custom fonts
- Each font shows: preview text, family name, source (Google / Uploaded), file size (for uploaded)
- "Upload Font" button → opens `FontUploadModal`
- Delete button for uploaded fonts (with confirmation)

**Validation Criteria**:
- [ ] Font manager shows all fonts currently used in the project
- [ ] Uploaded fonts show file size and delete action
- [ ] Deleting a font shows confirmation dialog
- [ ] "Upload Font" button opens the modal and refreshes list on success
