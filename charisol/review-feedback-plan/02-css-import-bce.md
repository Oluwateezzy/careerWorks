# Phase 2 — CSS Import + Font Extraction in Brand Context Engine

> **Priority**: 🟡 Medium (Backend-only, unlocks Phase 3)
> **Estimated Effort**: 1.5–2 days
> **Dependencies**: None

---

## Background

The Brand Context Engine's URL scraper ([`urlScraper.js`](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-be/utils/urlScraper.js)) extracts colors from inline HTML and up to 3 external `<link>` stylesheets. However:

1. **No `@import` recursion** — CSS files that use `@import url("...")` or `@import "..."` are not followed. Fonts and colors declared in imported stylesheets are missed entirely.
2. **No font extraction** — `font-family` declarations and `@font-face` rules are not parsed. The scraper output has "Detected Color Codes" and "Detected Color Variables" but no "Detected Font Families" section.
3. **The BCE prompt is not instructed to extract fonts** from the scraped content — even if fonts appeared in the raw HTML text, the LLM prompt doesn't emphasize them.

### Current Scraper Output Format (line 212-216)
```
Scraped Website Context for {url}:
Title: {title}
Description: {description}
Detected Color Codes: #1a237e, #3B82F6, ...
Detected Color Variables: --primary: #1a237e, ...
Content Snapshot: {bodyText}
```

### Target Scraper Output Format
```
Scraped Website Context for {url}:
Title: {title}
Description: {description}
Detected Color Codes: #1a237e, #3B82F6, ...
Detected Color Variables: --primary: #1a237e, ...
Detected Font Families: Inter, Outfit, Roboto
Detected Google Fonts: Inter (400,500,700), Outfit (600,700)
Detected @font-face Fonts: CustomBrand-Regular (woff2)
Content Snapshot: {bodyText}
```

---

## Tasks

### RV-200 — Extract `@import` URLs from CSS text

**Files**: [`urlScraper.js`](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-be/utils/urlScraper.js)

**Changes**:
Add function `extractCssImportUrls(cssText, baseUrl)`:
- Parse `@import url("...")` (with optional quotes, both single and double)
- Parse `@import "..."` (shorthand form without `url()`)
- Resolve relative URLs against `baseUrl`
- Return array of absolute URLs
- Skip `@import` inside comments (`/* ... */`)

```js
// Patterns to handle:
@import url("reset.css");
@import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;700');
@import "typography.css";
@import url(no-quotes.css);
```

**Validation Criteria**:
- [ ] Extracts URLs from all 4 `@import` syntax variants shown above
- [ ] Resolves relative URLs: `@import "sub/file.css"` on `https://example.com/css/main.css` → `https://example.com/css/sub/file.css`
- [ ] Ignores `@import` inside CSS comments
- [ ] Returns empty array for CSS without imports

---

### RV-201 — Recursively fetch imported CSS with depth/count limits

**Files**: [`urlScraper.js`](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-be/utils/urlScraper.js)

**Changes**:
Add function `fetchCssWithImports(url, options)`:
- Fetch the CSS file content
- Extract `@import` URLs using `extractCssImportUrls()`
- Recursively fetch imported CSS files
- Apply limits: `maxDepth: 2`, `maxFiles: 5`, `timeout: 2000ms` per file
- Track visited URLs to prevent infinite loops
- Return concatenated CSS text from all files

Modify `scrapeBrandUrl()` (line 145):
- After fetching external stylesheets via `<link>`, run `fetchCssWithImports()` on each
- Combine all CSS (inline `<style>` tags + linked stylesheets + their `@import` chains)
- Pass combined CSS to `extractColorsFromText()` and `extractCssVariables()`

**Validation Criteria**:
- [ ] `@import` chains are followed up to depth 2
- [ ] Maximum 5 total CSS files fetched (including root stylesheets)
- [ ] Circular `@import` references don't cause infinite loops
- [ ] Per-file timeout of 2 seconds is enforced
- [ ] Total scrape time stays under 15 seconds for typical websites

---

### RV-202 — Extract font-family declarations from CSS

**Files**: [`urlScraper.js`](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-be/utils/urlScraper.js)

**Changes**:
Add function `extractFontFamilies(cssText)`:
- Parse `font-family: "Inter", sans-serif;` declarations
- Parse `font: 700 1.5rem/1.4 "Outfit", sans-serif;` shorthand
- Parse `@font-face { font-family: "CustomBrand"; ... }` blocks
- Strip generic families (`serif`, `sans-serif`, `monospace`, `cursive`, `fantasy`, `system-ui`, `ui-sans-serif`, etc.)
- De-duplicate, return array of unique font family names

Add function `extractGoogleFontImports(cssText)`:
- Detect `@import url('https://fonts.googleapis.com/css2?...')`
- Parse the `family=` query parameters to extract font names + weights
- Return array of `{ family: string, weights: string[] }`

**Validation Criteria**:
- [ ] Extracts `Inter` and `Roboto` from `font-family: "Inter", "Roboto", sans-serif;`
- [ ] Extracts `Outfit` from `font: 700 1.5rem/1.4 "Outfit", sans-serif;`
- [ ] Extracts `CustomBrand` from `@font-face { font-family: "CustomBrand"; src: url(...); }`
- [ ] Does NOT include generic families (`serif`, `sans-serif`, `monospace`, etc.)
- [ ] Extracts `Inter` with weights `[400, 700]` from Google Fonts URL `?family=Inter:wght@400;700`
- [ ] Handles Google Fonts `css2` format with `+` separators for multiple families

---

### RV-203 — Integrate font extraction into scrapeBrandUrl output

**Files**: [`urlScraper.js`](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-be/utils/urlScraper.js)

**Changes**:
Modify `scrapeBrandUrl()` return string (line 212-216):
- Call `extractFontFamilies(combinedText)` on the combined CSS
- Call `extractGoogleFontImports(combinedText)` on the combined CSS
- Add "Detected Font Families: ..." line
- Add "Detected Google Fonts: ..." line (if any)
- Export `extractCssImportUrls`, `extractFontFamilies`, `extractGoogleFontImports` for testing

**Validation Criteria**:
- [ ] Output string includes `Detected Font Families: Inter, Outfit` line when fonts are found
- [ ] Output string includes `Detected Google Fonts: Inter (400,700)` when Google Fonts are detected
- [ ] Lines are omitted when no fonts/Google Fonts are detected (clean output)
- [ ] All new functions are exported from module for unit testing

---

### RV-204 — Update BCE website extraction prompt

**Files**: [`brandContextEngineController.js`](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-be/controllers/brandContextEngineController.js)

**Changes**:
Update the `extractFromWebsite` handler:
- The LLM prompt that processes scraped content should explicitly reference "Detected Font Families" and "Detected Google Fonts" sections
- Instruct the LLM to use directly-detected fonts as primary typography data (higher confidence than inferring from body text)
- If `extractFontFamilies()` returns results, include them as pre-parsed font hints alongside the LLM analysis

**Validation Criteria**:
- [ ] When a website uses Google Fonts, the BCE extraction result includes the correct `typography.heading.font.family` and `typography.body.font.family`
- [ ] Directly-detected fonts have `confidence: "high"` in the result
- [ ] Fallback to LLM inference when no fonts are directly detected

---

### RV-205 — Unit tests for scraper enhancements

**Files**: **[NEW]** [`tests/unit/urlScraper.test.js`](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-be/tests/unit/urlScraper.test.js)

**Changes**:
Create unit test file with tests for:
- `extractCssImportUrls()` — all 4 syntax variants, relative URL resolution, comment filtering
- `extractFontFamilies()` — font-family, font shorthand, @font-face
- `extractGoogleFontImports()` — css2 URL parsing, multiple families
- Integration: `scrapeBrandUrl()` output format includes new lines (requires mocking axios)

**Validation Criteria**:
- [ ] All unit tests pass: `node --test tests/unit/urlScraper.test.js`
- [ ] Test coverage includes edge cases: empty CSS, no fonts, comment-wrapped @import
- [ ] Tests are self-contained (no network calls — all mocked)
