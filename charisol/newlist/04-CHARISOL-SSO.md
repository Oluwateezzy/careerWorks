# Phase 4 — Charisol SSO Auth Migration

> **Priority**: 🟠 Medium-High (Cross-org requirement)
> **Estimated Effort**: 3–4 days
> **Dependencies**: Manual dev portal registration (BLOCKER)
> **Tracking**: Strata is item #4 on the Charisol SSO rollout list

---

## Background

### Current Auth System
Strata currently uses a **custom auth system**:
- **Email/password** registration with OTP verification via Maildrip
- **Google OAuth** via `google-auth-library` ([`Signin.tsx:67-91`](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-fe/components/modals/auth/Signin.tsx#L67-L91))
- **JWT tokens** stored in httpOnly cookies (set by backend)
- **User data** cached in localStorage
- Backend: [`authController.js`](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-be/controllers/authController.js)
- Frontend: [`auth.service.ts`](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-fe/services/auth.service.ts), [`AuthContext.tsx`](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-fe/context/AuthContext.tsx)

### Target Auth System
- Replace Google OAuth with **Charisol SSO** via `@charisol/auth-sdk-web`
- SSO provides **centralized Google sign-in** for all Charisol products
- **Email/password login must be preserved** (existing users)
- **Maildrip should only use SSO for sign-in, not for email send requests** (due to prior verification)
- Strata session management (JWT cookies) continues as-is — SSO just replaces the identity provider

### Key Constraint
> "Maildrip should only use this for sign in and not the email send request due to the verification previously done on maildrip"
>
> For Strata specifically, SSO replaces the Google OAuth button and optionally becomes the primary sign-in method.

---

## Prerequisites (Manual — BLOCKER)

### RV-400 — Register Strata on Charisol Developer Portal

**Scope**: Manual (no code)

**Steps**:
1. Visit https://developers.charisol.io/
2. Create account using the app email (e.g., `maildripdev@gmail.com` or Strata's email)
3. Register "Strata" as an application
4. Configure callback URLs:
   - Production: `https://strata.charisol.io/auth/callback`
   - Staging: `https://staging.strata.charisol.io/auth/callback` (if applicable)
   - Local dev: `http://localhost:3000/auth/callback`
5. Save the **Client ID** and **Client Secret**
6. Add these as environment variables in both FE and BE `.env` files

**Validation Criteria**:
- [ ] Strata app is registered on developers.charisol.io
- [ ] Client ID and Client Secret are saved
- [ ] Callback URLs include production AND localhost
- [ ] Env vars `CHARISOL_SSO_CLIENT_ID`, `CHARISOL_SSO_CLIENT_SECRET` are set in backend `.env`
- [ ] Env var `NEXT_PUBLIC_CHARISOL_CLIENT_ID` is set in frontend `.env.local`

---

### RV-401 — Install and audit `@charisol/auth-sdk-web`

**Scope**: Research + package install

**Steps**:
1. Run `npm install @charisol/auth-sdk-web` in the frontend project
2. Inspect the installed package to understand the API surface:
   - `node_modules/@charisol/auth-sdk-web/dist/index.d.ts` (TypeScript types)
   - `node_modules/@charisol/auth-sdk-web/README.md` (if present)
3. Document the SDK's initialization API, methods, and event callbacks
4. Identify: does the SDK handle the full OAuth flow (redirect + callback) or just token exchange?

**Validation Criteria**:
- [ ] Package installs without errors
- [ ] SDK API surface is documented (init, login, callback, logout methods)
- [ ] TypeScript types are available (or manual `.d.ts` needed)
- [ ] OAuth flow type is identified (redirect-based vs popup)

---

## Implementation Tasks

### RV-402 — Create Charisol auth wrapper (Frontend)

**Files**: **[NEW]** [`lib/charisol-auth.ts`](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-fe/lib/charisol-auth.ts)

**Changes**:
Create wrapper module that:
- Initializes the Charisol SDK with `clientId` from `NEXT_PUBLIC_CHARISOL_CLIENT_ID`
- Sets `redirectUri` to `${window.location.origin}/auth/callback`
- Exports:
  - `initiateLogin()` — starts the SSO login flow (redirect to Charisol)
  - `handleCallback(code: string)` — exchanges auth code for session via backend
  - `getCharsolLogout()` — SSO logout URL (if applicable)

**Validation Criteria**:
- [ ] `initiateLogin()` redirects browser to Charisol SSO login page
- [ ] `handleCallback()` sends auth code to backend and returns user data
- [ ] Works in both production and localhost environments

---

### RV-403 — Create OAuth callback page (Frontend)

**Files**: **[NEW]** [`app/auth/callback/page.tsx`](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-fe/app/auth/callback/page.tsx)

**Changes**:
Create callback page that:
- Extracts `code` (and optionally `state`) from URL search params
- Shows a loading spinner ("Completing sign-in...")
- Calls `handleCallback(code)` from `charisol-auth.ts`
- On success:
  - Stores user data in localStorage (`user`, `loggedIn`)
  - Redirects to `/projects`
- On failure:
  - Shows error message
  - "Try again" button redirects to `/login`
- Handles edge cases:
  - Missing `code` parameter → error state
  - Expired/invalid code → error state with retry

**Validation Criteria**:
- [ ] `/auth/callback?code=xxx` processes the code and redirects to `/projects`
- [ ] Missing `code` param shows error immediately (no API call)
- [ ] Invalid code shows "Authentication failed" with retry button
- [ ] Loading state shows spinner while processing

---

### RV-404 — Backend SSO login endpoint

**Files**: **[MODIFY]** [`authController.js`](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-be/controllers/authController.js)

**Changes**:
Add `charsolSsoLogin(req, res)` handler:
- Receives `{ code: string }` from frontend
- Exchanges `code` for access token via Charisol SSO token endpoint:
  ```
  POST {CHARISOL_SSO_TOKEN_URL}
  Body: { code, client_id, client_secret, redirect_uri, grant_type: "authorization_code" }
  ```
- Fetches user profile from Charisol SSO userinfo endpoint using the access token
- Extracts: `email`, `firstName`, `lastName`, `picture`
- Looks up user in DynamoDB by email:
  - **Existing user**: Update `lastLogin`, set SSO link
  - **New user**: Create account (auto-verified, no OTP needed)
- Generate JWT and set httpOnly cookie (same as existing `googleLoginUser` flow at line ~700)
- Return user data in response

**Files**: **[MODIFY]** [`routes/v1/index.js`](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-be/routes/v1/index.js)
- Add route: `POST /auth/charisol-sso` → `charsolSsoLogin`

**Files**: **[MODIFY]** `.env` / `.env.example`
- Add: `CHARISOL_SSO_CLIENT_ID`, `CHARISOL_SSO_CLIENT_SECRET`
- Add: `CHARISOL_SSO_TOKEN_URL` (e.g., `https://auth.charisol.io/oauth/token`)
- Add: `CHARISOL_SSO_USERINFO_URL` (e.g., `https://auth.charisol.io/oauth/userinfo`)

**Validation Criteria**:
- [ ] Valid auth code → returns user data + sets JWT cookie
- [ ] Existing user email → links SSO, updates lastLogin, no duplicate created
- [ ] New user email → creates account, sets `isVerified: true`
- [ ] Invalid/expired code → returns 401 with error message
- [ ] Missing `code` in body → returns 400 with validation error

---

### RV-405 — BFF proxy for SSO login

**Files**: **[NEW]** [`app/api/auth/charisol-sso/route.ts`](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-fe/app/api/auth/charisol-sso/route.ts)

**Changes**:
- POST handler that proxies to backend `POST /auth/charisol-sso`
- Forwards the `code` in the request body
- Forwards `Set-Cookie` headers from backend response to browser

**Validation Criteria**:
- [ ] Proxies request body to backend
- [ ] `Set-Cookie` headers from backend are forwarded to browser (for JWT)
- [ ] Error responses return the backend error message

---

### RV-406 — Replace Google OAuth with Charisol SSO in Signin modal

**Files**: **[MODIFY]** [`Signin.tsx`](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-fe/components/modals/auth/Signin.tsx)

**Changes**:
- Remove the Google Identity Services script loading (`gsi/client` script tag)
- Remove `google.accounts.id.initialize()` and `google.accounts.id.renderButton()` (lines 67-91)
- Replace with a styled "Sign in with Charisol" button that calls `initiateLogin()` from `charisol-auth.ts`
- Button style: Match existing design language (rounded, accent color)
- Keep email/password login form unchanged (for existing users)
- Keep the `authService.loginWithGoogle()` method in auth.service.ts as deprecated (remove later)

**Validation Criteria**:
- [ ] "Sign in with Charisol" button is visible on the login page
- [ ] Clicking it redirects to Charisol SSO (not Google directly)
- [ ] Email/password login still works for existing users
- [ ] No reference to `google.accounts.id` or GSI script in the component
- [ ] The button copy and styling match the Charisol brand guidelines

---

### RV-407 — Update AuthContext for SSO session handling

**Files**: **[MODIFY]** [`AuthContext.tsx`](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-fe/context/AuthContext.tsx)

**Changes**:
- `refreshUser()` continues to call `authService.fetchUser()` (JWT cookie-based — no change needed)
- `logout()` should also call Charisol SSO logout endpoint (if the SDK provides one) to clear the SSO session
- Add `loginWithSso(code: string)` method that calls the BFF SSO endpoint and refreshes user

**Validation Criteria**:
- [ ] After SSO login, `isAuthenticated` is `true` and `user` is populated
- [ ] After logout, both JWT cookie and SSO session are cleared
- [ ] Tab sync still works (storage event for `loggedIn` key)

---

### RV-408 — Update auth.service.ts with SSO method

**Files**: **[MODIFY]** [`auth.service.ts`](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-fe/services/auth.service.ts)

**Changes**:
Add method:
```ts
async loginWithCharsolSso(code: string): Promise<LoginResponse | null> {
  const response = await fetch(`${this.baseUrl}/charisol-sso`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ code }),
  });
  // ... same pattern as loginWithGoogle()
}
```

**Validation Criteria**:
- [ ] `loginWithCharsolSso(code)` sends POST to `/api/auth/charisol-sso`
- [ ] Stores user in localStorage on success
- [ ] Throws with backend error message on failure

---

### RV-409 — Update signup flow for Charisol SSO

**Files**: **[MODIFY]** [`Signin.tsx`](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-fe/components/modals/auth/Signin.tsx) (signup tab within signin modal)

**Changes**:
- The signup tab should also show "Sign up with Charisol" button
- Uses the same `initiateLogin()` — SSO handles both login and signup
- Keep email/password registration form unchanged
- On SSO callback, if user doesn't exist, backend auto-creates (RV-404)

**Validation Criteria**:
- [ ] Signup tab has "Sign up with Charisol" button
- [ ] New users coming through SSO are auto-created without OTP
- [ ] Email/password registration still works with OTP flow

---

### RV-410 — End-to-end SSO testing & tracking

**Scope**: Testing + tracking update

**Steps**:
1. Test SSO login flow end-to-end (localhost)
2. Test SSO with new user (auto-creation)
3. Test SSO with existing user (account linking)
4. Test logout (JWT + SSO session cleared)
5. Test email/password login still works alongside SSO
6. Deploy to staging and test with production Charisol SSO
7. Mark Strata as Done on the tracking list:
   ```
   4. Strata - Done ✅
   ```

**Validation Criteria**:
- [ ] New user: SSO → callback → auto-create → lands on `/projects`
- [ ] Existing user: SSO → callback → links account → lands on `/projects`
- [ ] Logout clears both JWT cookie and localStorage
- [ ] Email/password login works independently
- [ ] Cross-tab logout sync works
- [ ] Production deployment: SSO works with `strata.charisol.io` callback URL
