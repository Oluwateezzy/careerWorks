# Branch & Publish Tab — Architecture Document

> **Version:** 1.0  
> **Date:** 2026-08-05  
> **Scope:** Create a dedicated "Branch & Publish" tab in the project sidebar for branch management and publish history, with branch rename, delete, and owner publish controls.

---

## 1. Problem Statement

### 1.1 Current State

Branch management and publish operations are scattered across the UI:
- **Branch switching** is done via a dropdown in `ProjectTopBar.tsx`
- **Branch creation** is via `BranchCreateDialog.tsx` in `components/collaboration/`
- **Merge/diff** is accessed through contextual buttons
- **Publish** is a button in the top bar
- **No dedicated view** for managing all branches or viewing publish history

### 1.2 What Users Need

A centralized **Branch & Publish** tab where project owners and collaborators can:
- View all active branches with metadata (creator, last updated, status)
- Edit branch names (owner or branch creator only)
- Delete branches (owner or branch creator only)
- View publish history (all versions with timestamps and messages)
- Publish from main (owner only)
- See which branch is currently active

---

## 2. Target Architecture

### 2.1 UI Layout

```
┌─────────────────────────────────────────────────────────────────┐
│  Branch & Publish Tab                                            │
│                                                                  │
│  ┌───────────────────────────────────────────────────────────┐   │
│  │  BRANCHES                                     [+ New]    │   │
│  │  ─────────────────────────────────────────────────────── │   │
│  │                                                          │   │
│  │  ★ main (active)                                         │   │
│  │    Last updated: 2m ago · Owner                          │   │
│  │                                                          │   │
│  │  ○ alice/typography-rework                    [⋯]        │   │
│  │    Created by alice@example.com · 3h ago                 │   │
│  │    Status: active · 5 changes ahead of main             │   │
│  │    [Switch] [View Diff] [Merge to Main]                  │   │
│  │    ⋯ menu: [Rename] [Delete]                             │   │
│  │                                                          │   │
│  │  ○ bob/color-update                           [⋯]        │   │
│  │    Created by bob@example.com · 1d ago                   │   │
│  │    Status: active · 2 changes ahead of main             │   │
│  │    [Switch] [View Diff] [Merge to Main]                  │   │
│  │    ⋯ menu: [Rename] [Delete]                             │   │
│  │                                                          │   │
│  │  ○ ai-brief-a1b2c3 (merged)                              │   │
│  │    Auto-created · Merged 2d ago                          │   │
│  │    [Delete]                                               │   │
│  └───────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌───────────────────────────────────────────────────────────┐   │
│  │  PUBLISH HISTORY                              [Publish]   │   │
│  │  ─────────────────────────────────────────────────────── │   │
│  │                                                          │   │
│  │  v3 — "Updated typography and spacing"                   │   │
│  │  Published 1h ago by owner@example.com                   │   │
│  │  ● Active (default version)                              │   │
│  │                                                          │   │
│  │  v2 — "Initial color palette setup"                      │   │
│  │  Published 3d ago by owner@example.com                   │   │
│  │  [Set as Default]                                         │   │
│  │                                                          │   │
│  │  v1 — "First publish"                                    │   │
│  │  Published 1w ago by owner@example.com                   │   │
│  │  [Set as Default]                                         │   │
│  └───────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 Component Structure

```
components/
  sections/
    project/
      BranchPublishSection.tsx       ← Main section (new)
      BranchListPanel.tsx            ← Branch listing with actions (new)
      BranchCard.tsx                 ← Individual branch card (new)
      BranchRenameDialog.tsx         ← Rename branch modal (new)
      PublishHistoryPanel.tsx         ← Publish version history (new)
      PublishVersionCard.tsx          ← Individual version card (new)
```

---

## 3. Backend Changes

### 3.1 New Endpoints

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| PATCH | `/v1/project/:id/branches/:name/rename` | editor+ (creator or owner) | Rename a branch |
| GET | `/v1/project/:id/publish-history` | viewer+ | List all published versions |

### 3.2 Branch Rename

```javascript
// PATCH /v1/project/:id/branches/:name/rename
// Body: { newName: string }
async function renameBranch(req, res) {
  const { branchName } = req.params;
  const { newName } = req.body;

  // Validation
  if (!newName || !branchNameRegex.test(newName)) {
    return res.status(400).json({ status: false, message: "Invalid branch name" });
  }
  if (branchName === "main" || newName === "main") {
    return res.status(400).json({ status: false, message: "Cannot rename main branch" });
  }

  // Auth: only owner or branch creator
  const branch = await ProjectBranch.get({ pk: projectId, sk: `BRANCH#${branchName}` });
  if (!branch) return res.status(404).json({ status: false, message: "Branch not found" });
  
  const isOwner = req.projectRole === "owner";
  const isCreator = branch.createdBy === req.user.pk;
  if (!isOwner && !isCreator) {
    return res.status(403).json({ status: false, message: "Only branch creator or owner can rename" });
  }

  // Check new name doesn't exist
  const existing = await ProjectBranch.get({ pk: projectId, sk: `BRANCH#${newName}` });
  if (existing && existing.status !== "closed") {
    return res.status(409).json({ status: false, message: "Branch name already exists" });
  }

  // 1. Copy S3 files to new key
  // 2. Create new ProjectBranch record with new name
  // 3. Mark old record as "closed" (or delete)
  // 4. Return success
}
```

### 3.3 Publish History

```javascript
// GET /v1/project/:id/publish-history
async function getPublishHistory(req, res) {
  const project = req.projectRecord;
  
  const versions = [];
  const projectData = project.projectData || {};
  
  for (const [versionKey, versionInfo] of Object.entries(projectData)) {
    versions.push({
      version: versionKey,
      s3Key: versionInfo.s3Key,
      hash: versionInfo.hash,
      message: versionInfo.message || `Version ${versionKey}`,
      publishedAt: versionInfo.updatedAt || versionInfo.createdAt,
      publishedBy: versionInfo.publishedBy || project.ownerUserId,
      isDefault: String(versionKey) === String(project.defaultVersion),
    });
  }

  // Sort by version descending
  versions.sort((a, b) => Number(b.version) - Number(a.version));

  return res.json({
    status: true,
    data: {
      versions,
      currentVersion: project.defaultVersion,
      totalVersions: project.versions || versions.length,
    },
  });
}
```

---

## 4. Frontend Service Updates

### 4.1 Branch Service Extensions

```typescript
// services/branch.service.ts — add:

async renameBranch(
  projectId: string,
  oldName: string,
  newName: string
): Promise<boolean> {
  try {
    const response = await fetch(
      `${baseUrl}/project/${projectId}/branches/${oldName}/rename`,
      {
        method: "PATCH",
        headers: { "Content-Type": "application/json" },
        credentials: "include",
        body: JSON.stringify({ newName }),
      }
    );
    return response.ok;
  } catch (error) {
    console.error(`Failed to rename branch ${oldName} to ${newName}:`, error);
    return false;
  }
}
```

### 4.2 Publish Service Extensions

```typescript
// services/publish.service.ts — add:

async getPublishHistory(projectId: string): Promise<PublishVersion[]> {
  try {
    const response = await fetch(
      `/api/project/${projectId}/publish-history`,
      { method: "GET", credentials: "include" }
    );
    if (!response.ok) return [];
    const data = await response.json();
    return data?.data?.versions || [];
  } catch (error) {
    console.error(`Failed to fetch publish history:`, error);
    return [];
  }
}

async setDefaultVersion(projectId: string, version: string): Promise<boolean> {
  try {
    const response = await fetch(
      `/api/project/${projectId}/set-default-version`,
      {
        method: "PATCH",
        headers: { "Content-Type": "application/json" },
        credentials: "include",
        body: JSON.stringify({ version }),
      }
    );
    return response.ok;
  } catch (error) {
    console.error(`Failed to set default version:`, error);
    return false;
  }
}
```

---

## 5. Sidebar Integration

Add "Branch & Publish" to `ProjectSidebar.tsx` nav items:

```typescript
// In NAV_ITEMS array, add after "collaboration":
{
  id: "branch-publish",
  label: "Branch & Publish",
  icon: <svg ...>...</svg> // Git branch icon
}
```

**Visibility rule**: Show to all project members (viewer can see read-only; editor+ can manage branches; owner can publish).

---

## 6. Permission Matrix

| Action | Owner | Editor | Viewer |
|--------|-------|--------|--------|
| View branches | ✅ | ✅ | ✅ |
| Create branch | ✅ | ✅ | ❌ |
| Switch branch | ✅ | ✅ | ✅ (read-only) |
| Rename branch (own) | ✅ | ✅ | ❌ |
| Rename branch (others') | ✅ | ❌ | ❌ |
| Delete branch (own) | ✅ | ✅ | ❌ |
| Delete branch (others') | ✅ | ❌ | ❌ |
| View diff | ✅ | ✅ | ✅ |
| Merge to main | ✅ | ✅ | ❌ |
| View publish history | ✅ | ✅ | ✅ |
| Publish | ✅ | ❌ | ❌ |
| Set default version | ✅ | ❌ | ❌ |

---

**Architecture Version**: 1.0  
**Last Updated**: 2026-08-05  
**Maintained By**: Design System Team
