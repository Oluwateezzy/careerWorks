# Manual UI Test Guide: Collaborative Version Control System

This guide outlines the step-by-step manual validation procedures for testing the **Collaborative Branching & Multi-User VCS** features introduced in the design system. It covers both happy paths, edge cases, role-based access restrictions, and conflict resolution scenarios.

> 📖 **Related Test Guides**:
> - [Multi-Font Support & Font Integration Test Guide](file:///Users/oluwatobiloba/Desktop/charisol/design_system/docsplan/FONT_INTEGRATION_MANUAL_UI_TEST_GUIDE.md)

---

## 📋 Table of Contents
1. [Prerequisites & Environment Setup](#1-prerequisites--environment-setup)
2. [Role-Based Access Matrix](#2-role-based-access-matrix)
3. [Step-by-Step Test Scenarios](#3-step-by-step-test-scenarios)
   - [Scenario 1: Collaboration Setup & Invitation Flow](#scenario-1-collaboration-setup--invitation-flow)
   - [Scenario 2: Isolated Branch Operations](#scenario-2-isolated-branch-operations)
   - [Scenario 3: Auto-Merge (Non-Overlapping Changes)](#scenario-3-auto-merge-non-overlapping-changes)
   - [Scenario 4: Merge Conflicts & Manual Resolution](#scenario-4-merge-conflicts--manual-resolution)
   - [Scenario 5: Branch Rebase Flow](#scenario-5-branch-rebase-flow)
   - [Scenario 6: Viewer Role Restrictions](#scenario-6-viewer-role-restrictions)
   - [Scenario 7: Member Role Revocation](#scenario-7-member-role-revocation)
4. [Visual Checklist & Success Criteria](#4-visual-checklist--success-criteria)

---

## 1. Prerequisites & Environment Setup

Before starting, ensure both the frontend and backend are configured and running locally.

### Step A: Configure Backend
1. Open [charisol-design-system-be](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-be) and verify the `.env` file:
   ```ini
   PORT=5000
   DYNAMODB_PROJECT_TABLE_NAME=TestProjects
   DYNAMODB_PROJECT_MEMBER_TABLE_NAME=TestProjectMembers
   DYNAMODB_PROJECT_BRANCH_TABLE_NAME=TestProjectBranches
   ```
2. Start the backend:
   ```bash
   npm run dev
   ```

### Step B: Configure Frontend
1. Open [charisol-design-system-fe](file:///Users/oluwatobiloba/Desktop/charisol/design_system/charisol-design-system-fe) and edit `.env.local` to point to the local backend:
   ```ini
   BACKEND_API_URL=http://localhost:5000/v1
   ```
2. Start the frontend:
   ```bash
   npm run dev
   ```
3. Open `http://localhost:3000` in your web browser.

### Step C: Test Users Seeding
Create three test accounts using the signup/register screens or seed script:
*   **User A (Owner)**: `owner@example.com`
*   **User B (Editor)**: `editor@example.com`
*   **User C (Viewer)**: `viewer@example.com`

---

## 2. Role-Based Access Matrix

Verify that UI permissions align with user roles:

| Feature / Action | Owner | Editor | Viewer |
|------------------|:---:|:---:|:---:|
| Switch Branches | ✅ | ✅ | ✅ |
| Create Branch | ✅ | ✅ | ❌ (Button disabled) |
| Edit Token Values | ✅ | ✅ (On branch) | ❌ (Fields read-only) |
| Merge Branch | ✅ | ✅ | ❌ (Action hidden) |
| Rebase Branch | ✅ | ✅ | ❌ (Action hidden) |
| Invite Collaborators | ✅ | ❌ (Settings tab hidden/blocked) | ❌ (Settings tab hidden/blocked) |
| Update Collaborator Roles | ✅ | ❌ | ❌ |
| Revoke Collaborator Access | ✅ | ❌ | ❌ |

---

## 3. Step-by-Step Test Scenarios

### Scenario 1: Collaboration Setup & Invitation Flow
**Goal**: Verify that an owner can enable collaboration and invite members, and the invited user can accept.

1. **Log in as User A (Owner)** and navigate to the dashboard.
2. Select a project and click the **Settings** icon/link in the navbar.
3. Toggle the **Enable Collaboration** switch to `ON` and save settings.
4. Go to the new **Collaborators** tab (rendered via `MemberManagement.tsx`).
5. Click **Invite Member**. In the dialog:
   - Input `editor@example.com`.
   - Select **Editor** from the role dropdown.
   - Click **Send Invitation**.
6. Repeat the process to invite `viewer@example.com` with the **Viewer** role.
7. **Verification**:
   - Verify both users appear in the list with a status of `pending`.
   - Log out, then **log in as User B (Editor)**.
   - You should see an invitation banner: *"You've been invited to collaborate on [Project Name]"* with **Accept** and **Decline** buttons.
   - Click **Accept**. The banner disappears and the project displays in your dashboard list.
   - Log back in as User A, navigate to Collaborators, and verify `editor@example.com` status changed to `active`.

---

### Scenario 2: Isolated Branch Operations
**Goal**: Verify a collaborator can create a branch and make changes that do not affect the main branch.

1. **Log in as User B (Editor)** and open the collaborative project.
2. In the top bar toolbar, find the **Branch Selector** dropdown. It should display `main` by default.
3. Click the dropdown and select **Create new branch...** (opens `BranchCreateDialog`).
4. Type `feature/button-color` and click **Create**.
5. The Selector dropdown should update to show `feature/button-color` with an `active` badge.
6. Open the design tokens panel and modify the `--color-primary` variable to `#ff0000` (Red).
7. Notice the unsaved changes state, then click **Sync** (flushes patches to `/branches/feature/button-color/upload`).
8. Switch the Branch Selector dropdown back to `main`.
9. **Verification**:
   - Verify `--color-primary` on `main` remains unchanged (its original value, e.g., `#A9358D`).
   - Switch back to `feature/button-color` and verify `--color-primary` correctly displays `#ff0000`.

---

### Scenario 3: Auto-Merge (Non-Overlapping Changes)
**Goal**: Verify that a branch with modifications merges cleanly back into `main` if no changes occurred on `main`.

1. While on the `feature/button-color` branch as **User B (Editor)**, verify all changes are synced.
2. Click the **Compare & Merge** button in the Project Topbar.
3. The **Merge Diff View** modal will display:
   - Check that it correctly lists the modified token `--color-primary` (old value vs new value `#ff0000`).
   - Verification styling: the change should be highlighted in yellow.
4. Input a commit message: `Update button color to red`.
5. Click the **Merge** button.
6. **Verification**:
   - Verify a toast notification appears: *"Branch merged successfully!"*.
   - The editor should automatically redirect back to the `main` branch.
   - Check that the `--color-primary` value on `main` is now `#ff0000`.
   - The Branch Selector dropdown should now list `feature/button-color` with a `merged` badge (and it cannot be selected for edits).

---

### Scenario 4: Merge Conflicts & Manual Resolution
**Goal**: Trigger a 3-way merge conflict when two editors edit the same token, and manually resolve it using the UI panel.

1. As **User A (Owner)**:
   - Select the `main` branch. Set `--color-primary` to `#111111` and sync.
2. Create two new branches from `main`:
   - `branch-a` (for User A / Owner).
   - `branch-b` (for User B / Editor).
3. On `branch-a` (User A):
   - Change `--color-primary` to `#aaaaaa` and sync.
4. **Log in as User B (Editor)** and open `branch-b`:
   - Change `--color-primary` to `#bbbbbb` and sync.
5. **As User A (Owner)**:
   - Click **Compare & Merge** on `branch-a`.
   - Since `main` has not changed since the fork, click **Merge**. This merges cleanly. `main` is now `#aaaaaa`.
6. **As User B (Editor)**:
   - Click **Compare & Merge** on `branch-b`.
   - Type a commit message and click **Merge**.
   - **Verification**: The merge will fail, displaying a conflict warning screen: *"Merge Conflicted. Manual Resolution Required"*.
   - Click **Resolve Conflicts**. This opens the **Conflict Resolution Panel**.
   - Verify the conflicting token `--color-primary` is listed side-by-side:
     *   **Base value**: `#111111`
     *   **Main value (User A's changes)**: `#aaaaaa`
     *   **Branch value (User B's changes)**: `#bbbbbb`
   - Select the radio option **Use Branch value (#bbbbbb)**.
   - Click **Resolve & Merge**.
   - Verify a toast indicates successful merge.
   - Switch to `main` and check that the value is indeed `#bbbbbb`.

---

### Scenario 5: Branch Rebase Flow
**Goal**: Update a branch with latest changes from `main` before merging.

1. **User B (Editor)** creates branch `feature/rebase-test` from `main`.
2. While User B is working, **User A (Owner)** goes to `main` and adds a new color token `--color-accent: #00ff00` and syncs.
3. **User B** switches to `feature/rebase-test` and modifies an existing component padding.
4. In the Branch settings/selector dropdown, User B clicks **Rebase onto main**.
5. **Verification**:
   - Rebase completes without conflicts.
   - Check that `--color-accent: #00ff00` is now visible on User B's branch `feature/rebase-test` while keeping the padding edits intact.

---

### Scenario 6: Viewer Role Restrictions
**Goal**: Verify that users with the `viewer` role cannot edit or perform write operations.

1. **Log in as User C (Viewer)**.
2. Open the project.
3. **Verification**:
   - All input text boxes, color picker dialogs, and size fields in the tokens panel must be disabled (grayed out).
   - In the **Branch Selector**, verify that the **Create new branch...** button is disabled.
   - Verify the **Compare & Merge** and **Rebase** buttons are hidden from the Project Topbar.
   - Try to access the project settings URL `/project/[id]/manage` directly. Verify the page displays a **403 Access Denied** screen or redirects to the project editor view.

---

### Scenario 7: Member Role Revocation
**Goal**: Verify that removing a collaborator revokes their access immediately.

1. **Log in as User A (Owner)**.
2. Navigate to the **Collaborators** settings tab.
3. Find `editor@example.com` in the list and click the **Delete/Remove** icon.
4. Click **Confirm** on the popup.
5. **Verification**:
   - The user should disappear from the collaborator list immediately.
   - **Log in as User B (Editor)**.
   - Attempt to open the project page or make an API request. Verify the page blocks access, displaying *"403 Forbidden: You no longer have access to this project"*.

---

## 4. Visual Checklist & Success Criteria

Make sure the following UI states align with modern, premium design expectations:
*   [ ] **Conflict Panel Layout**: Conflict options (Main / Branch / Custom) are clearly demarcated side-by-side with crisp typography (e.g., using Outfit or Inter font).
*   [ ] **Diff Highlights**: Merged diff additions are color-coded in green (`rgba(34,197,94,0.1)` bg), deletions in red (`rgba(239,68,68,0.1)` bg), and modifications in yellow (`rgba(234,179,8,0.1)` bg).
*   [ ] **Badges**: Branch status badges (`active`, `merged`, `closed`) render with smooth border-radius, micro-paddings, and muted color backgrounds (e.g., HSL soft-greens and grays).
*   [ ] **Notifications**: Success/error feedback toasts from React Toastify animate cleanly and disappear after 3 seconds.
