# Connected Apps Tracking — Implementation Plan

> **Prerequisite**: Read [ARCHITECTURE.md](file:///Users/oluwatobiloba/Desktop/charisol/design_system/docsplan/connected_app/ARCHITECTURE.md) first for the full system design and data model.

---

## Phase Overview

| Phase | Name                        | Scope                                    | Est. Effort |
|:------|:----------------------------|:-----------------------------------------|:------------|
| 1     | Backend: Data Model & API   | DynamoDB model, controller, routes       | 2–3 days    |
| 2     | NPM Package: Heartbeat SDK  | Telemetry module, StrataProvider changes | 2 days      |
| 3     | Frontend: Dashboard UI       | "See It in the Wild" + Settings tab     | 2–3 days    |
| 4     | Polish & Documentation       | README, error handling, tests           | 1–2 days    |

**Total estimated effort: 7–10 days**

---

## Phase 1: Backend — Data Model & API

### 1.1 Create ConnectedSurface Model

**File**: `models/ConnectedSurface.js` [NEW]

```javascript
const dynamoose = require("./helpers/dynamoose");

const tableName = process.env.DYNAMODB_PROJECT_TABLE_NAME || process.env.DYNAMODB_TABLE_NAME;

const connectedSurfaceSchema = new dynamoose.Schema(
  {
    pk: { type: String, hashKey: true },        // PROJECT#<projectId>
    sk: { type: String, rangeKey: true },        // SURFACE#<surfaceId>
    surfaceId: { type: String, required: true },
    origin: { type: String, required: true },
    hostname: { type: String },
    displayName: { type: String },
    emoji: { type: String, default: "🌐" },
    environment: { type: String, default: "unknown" },
    status: { type: String, default: "live" },
    sdkVersion: { type: String },
    syncMode: { type: String },
    syncInterval: { type: Number },
    lastSeenAt: { type: String },
    firstSeenAt: { type: String },
    heartbeatCount: { type: Number, default: 0 },
    userAgent: { type: String },
    metadata: { type: Object, default: () => ({}) },
    source: { type: String, default: "auto" },   // "auto" | "manual"
  },
  {
    saveUnknown: false,
    timestamps: true,
  }
);

module.exports = dynamoose.model(tableName, connectedSurfaceSchema, {
  create: false,
  waitForActive: false,
  initialize: true,
});
```

> [!IMPORTANT]
> **Design Decision — Same Table, Different Sort Key Prefix**
> 
> We reuse the existing project table rather than provisioning a new DynamoDB table. The `pk` is `PROJECT#<projectId>` and the `sk` is `SURFACE#<surfaceId>`. This co-locates surfaces with their owning project and requires zero infrastructure changes.
> 
> **Tradeoff**: We are adding a new entity type to an existing table. This means `dynamoose.model()` will share the same table — we must be careful not to conflict with existing Project schema's `saveUnknown: true` behaviour. The ConnectedSurface schema explicitly sets `saveUnknown: false` to prevent attribute bleed.

---

### 1.2 Create Connected Surface Controller

**File**: `controllers/connectedSurfaceController.js` [NEW]

#### `POST /v1/connect/:projectId/heartbeat`

This is the primary endpoint called by the SDK runtime. It performs an **upsert**: if the surface already exists, it updates `lastSeenAt` and `heartbeatCount`; if not, it creates a new record.

```javascript
const crypto = require("crypto");

function deriveSurfaceId(origin) {
  return crypto.createHash("sha256").update(origin).digest("hex").substring(0, 16);
}

async function heartbeat(req, res) {
  const { projectId } = req.params;
  const { origin, sdkVersion, syncMode, syncInterval, userAgent, metadata, appName } = req.body;

  if (!origin) {
    return res.status(400).json({ status: false, message: "origin is required" });
  }

  const surfaceId = deriveSurfaceId(origin);
  const hostname = new URL(origin).hostname;
  const now = new Date().toISOString();

  // Enforce max 100 surfaces per project
  const existingCount = await ConnectedSurface.query("pk")
    .eq(`PROJECT#${projectId}`)
    .where("sk").beginsWith("SURFACE#")
    .count()
    .exec();

  const existingSurface = await ConnectedSurface.query("pk")
    .eq(`PROJECT#${projectId}`)
    .where("sk").eq(`SURFACE#${surfaceId}`)
    .exec();

  if (existingSurface.length === 0 && existingCount.count >= 100) {
    return res.status(429).json({ status: false, message: "Maximum connected surfaces reached" });
  }

  // Upsert
  await ConnectedSurface.update(
    { pk: `PROJECT#${projectId}`, sk: `SURFACE#${surfaceId}` },
    {
      surfaceId,
      origin,
      hostname,
      displayName: appName || hostname,
      sdkVersion: sdkVersion || "unknown",
      syncMode: syncMode || "unknown",
      syncInterval: syncInterval || 5000,
      lastSeenAt: now,
      userAgent: userAgent || "",
      metadata: metadata || {},
      source: "auto",
      $ADD: { heartbeatCount: 1 },
      // Only set firstSeenAt if this is a new record (use conditional expression)
      ...(existingSurface.length === 0 ? { firstSeenAt: now } : {}),
    }
  );

  return res.json({ status: true, data: { surfaceId } });
}
```

> [!NOTE]
> **Rate Limiting Decision**
>
> We do NOT add a per-request rate limiter at the Express middleware level for heartbeats. Instead, the SDK self-throttles (default: 1 heartbeat per 5 minutes) and the backend enforces a max-100-surfaces cap per project. This avoids the complexity of Redis/Memcached for rate limiting while still preventing abuse.
>
> **Tradeoff**: A malicious actor could spam heartbeats from the same origin. However, since the upsert is idempotent (same origin → same surfaceId), this only results in rapid `lastSeenAt` updates — not data growth. If abuse becomes an issue, we can add throttling later.

#### `GET /v1/projects/:projectId/surfaces`

Dashboard endpoint for project owners. Returns all surfaces with computed status.

```javascript
function computeStatus(surface) {
  const now = Date.now();
  const lastSeen = new Date(surface.lastSeenAt).getTime();
  const staleness = now - lastSeen;
  const staleThreshold = Math.max((surface.syncInterval || 5000) * 2, 60_000);
  const dormantThreshold = 7 * 24 * 60 * 60 * 1000;

  if (staleness <= staleThreshold) return "live";
  if (staleness <= dormantThreshold) return "stale";
  return "dormant";
}

async function listSurfaces(req, res) {
  const { projectId } = req.params;
  const results = await ConnectedSurface.query("pk")
    .eq(`PROJECT#${projectId}`)
    .where("sk").beginsWith("SURFACE#")
    .exec();

  const surfaces = results.map(s => ({
    ...s.toJSON(),
    status: computeStatus(s),
  }));

  // Sort: live first, then stale, then dormant; within each group, most recent first
  const statusOrder = { live: 0, stale: 1, dormant: 2 };
  surfaces.sort((a, b) => {
    const orderDiff = statusOrder[a.status] - statusOrder[b.status];
    if (orderDiff !== 0) return orderDiff;
    return new Date(b.lastSeenAt) - new Date(a.lastSeenAt);
  });

  return res.json({ status: true, data: { surfaces, total: surfaces.length } });
}
```

#### Other endpoints

| Endpoint | Implementation Notes |
|:---------|:---------------------|
| `PATCH /v1/projects/:projectId/surfaces/:surfaceId` | Update `displayName`, `emoji` only. Validated by `authenticateUser`. |
| `DELETE /v1/projects/:projectId/surfaces/:surfaceId` | Hard delete. Validated by `authenticateUser`. |
| `POST /v1/projects/:projectId/surfaces/manual` | Accept `url`, `name`, `emoji`. Generate surfaceId from URL. Set `source: "manual"`. |
| `POST /v1/connect/:projectId/register` | Simplified heartbeat for build-time. Accept `hostname`, `sdkVersion`, `environment`. No origin required (CI has no browser origin). Use hostname-based surfaceId. |

### 1.3 Register Routes

**File**: `routes/v1/index.js` [MODIFY]

```diff
+ const connectedSurfaceController = require("../../controllers/connectedSurfaceController");

  // Existing sync routes...
  router.get("/sync/:projectId/variables", authenticateSync, getPublicVariables);

+ // Connected Surfaces — SDK heartbeat (uses sync auth)
+ router.post("/connect/:projectId/heartbeat", authenticateSync, connectedSurfaceController.heartbeat);
+ router.post("/connect/:projectId/register", authenticateSync, connectedSurfaceController.register);

+ // Connected Surfaces — Dashboard management (uses user auth)
+ router.get("/projects/:projectId/surfaces", authenticateUser, connectedSurfaceController.listSurfaces);
+ router.patch("/projects/:projectId/surfaces/:surfaceId", authenticateUser, connectedSurfaceController.updateSurface);
+ router.delete("/projects/:projectId/surfaces/:surfaceId", authenticateUser, connectedSurfaceController.deleteSurface);
+ router.post("/projects/:projectId/surfaces/manual", authenticateUser, connectedSurfaceController.manualLink);
```

---

## Phase 2: NPM Package — Heartbeat SDK

### 2.1 Create Heartbeat Reporter

**File**: `src/telemetry/heartbeat.ts` [NEW]

```typescript
interface HeartbeatConfig {
  projectId: string;
  syncToken?: string;
  syncMode: "cdn" | "legacy" | "static";
  syncInterval: number;
  appName?: string;
  environment?: string;
  apiBase?: string; // defaults to production API
}

const DEFAULT_HEARTBEAT_INTERVAL = 5 * 60 * 1000; // 5 minutes
const DEFAULT_API_BASE = "https://api-production-charisol-design-system-be.aws.charisol.io";

export class HeartbeatReporter {
  private intervalId: ReturnType<typeof setInterval> | null = null;
  private config: HeartbeatConfig;
  private disabled = false;

  constructor(config: HeartbeatConfig) {
    this.config = config;
  }

  async start(): Promise<void> {
    // Guard: browser only
    if (typeof window === "undefined") return;

    // Guard: production only (unless explicitly enabled)
    if (this.config.environment === "development") return;

    // Send initial heartbeat
    await this.sendHeartbeat();

    // Schedule recurring heartbeats
    this.intervalId = setInterval(
      () => this.sendHeartbeat(),
      DEFAULT_HEARTBEAT_INTERVAL
    );
  }

  stop(): void {
    if (this.intervalId) {
      clearInterval(this.intervalId);
      this.intervalId = null;
    }
  }

  private async sendHeartbeat(): Promise<void> {
    if (this.disabled) return;

    try {
      const apiBase = this.config.apiBase || DEFAULT_API_BASE;
      const url = `${apiBase}/v1/connect/${this.config.projectId}/heartbeat`;

      const headers: Record<string, string> = {
        "Content-Type": "application/json",
      };
      if (this.config.syncToken) {
        headers["Authorization"] = `Bearer ${this.config.syncToken}`;
      }

      const body = {
        origin: window.location.origin,
        sdkVersion: SDK_VERSION, // injected at build time via rollup
        syncMode: this.config.syncMode,
        syncInterval: this.config.syncInterval,
        userAgent: navigator.userAgent,
        appName: this.config.appName,
        metadata: {
          pageUrl: window.location.href,
          referrer: document.referrer,
          framework: detectFramework(),
        },
      };

      const res = await fetch(url, {
        method: "POST",
        headers,
        body: JSON.stringify(body),
        // Don't let heartbeat affect page performance
        keepalive: true,
      });

      // If we get 401, disable heartbeat for this session
      if (res.status === 401) {
        console.warn("[strata] Heartbeat disabled: invalid sync token");
        this.disabled = true;
        this.stop();
      }
    } catch (e) {
      // Silently swallow — heartbeat failure should never affect the app
    }
  }
}

function detectFramework(): string {
  if (typeof window === "undefined") return "unknown";
  // @ts-ignore
  if (window.__NEXT_DATA__) return "nextjs";
  // @ts-ignore
  if (window.__REMIX_DEV_TOOLS) return "remix";
  // @ts-ignore
  if (window.__GATSBY) return "gatsby";
  return "react";
}
```

> [!IMPORTANT]
> **Critical Decision — `keepalive: true`**
>
> The heartbeat fetch uses `keepalive: true` so the browser can complete the request even if the user navigates away. This is the same pattern used by analytics libraries (Google Analytics, Segment). It ensures heartbeats don't get cancelled by navigation.
>
> **Tradeoff**: `keepalive` requests have a 64KB body limit in most browsers. Our heartbeat body is ~300 bytes, so this is not a concern.

> [!WARNING]
> **SDK Version Injection**
>
> `SDK_VERSION` must be injected at build time via the Rollup config (using `@rollup/plugin-replace`). This avoids a runtime `require()` of `package.json` which would bloat the bundle.
>
> Add to `rollup.config.js`:
> ```javascript
> import replace from '@rollup/plugin-replace';
> const pkg = require('./package.json');
> 
> // In plugins array:
> replace({
>   preventAssignment: true,
>   SDK_VERSION: JSON.stringify(pkg.version),
> })
> ```

### 2.2 Integrate into StrataProvider

**File**: `src/providers/StrataProvider.tsx` [MODIFY]

```diff
+ import { HeartbeatReporter } from "../telemetry/heartbeat";

  interface StrataProviderProps {
    children: React.ReactNode;
    // ... existing props ...
+   /** Whether to report usage telemetry (heartbeat). Defaults to true in production. */
+   reportUsage?: boolean;
+   /** Human-readable app name shown in the Strata dashboard. */
+   appName?: string;
+   /** Override automatic environment detection. */
+   environment?: string;
  }

  export const StrataProvider: React.FC<StrataProviderProps> = ({
    // ... existing destructured props ...
+   reportUsage = true,
+   appName,
+   environment,
  }) => {

+   // Heartbeat reporter — ref to avoid re-creation on each render
+   const heartbeatRef = useRef<HeartbeatReporter | null>(null);
+
+   useEffect(() => {
+     if (!reportUsage || !syncEnabled || !projectId) return;
+
+     const syncMode = snapshotCdnBase ? "cdn" : syncUrl ? "legacy" : "static";
+     const reporter = new HeartbeatReporter({
+       projectId,
+       syncToken,
+       syncMode,
+       syncInterval,
+       appName,
+       environment,
+     });
+
+     heartbeatRef.current = reporter;
+     reporter.start();
+
+     return () => {
+       reporter.stop();
+       heartbeatRef.current = null;
+     };
+   }, [reportUsage, syncEnabled, projectId, syncToken, snapshotCdnBase, syncUrl, syncInterval, appName, environment]);

    // ... rest of existing provider code unchanged ...
  };
```

### 2.3 Export from Package Index

**File**: `src/index.ts` [MODIFY]

```diff
+ export { HeartbeatReporter } from "./telemetry/heartbeat";
```

### 2.4 Extend postinstall for Build-Time Registration

**File**: `src/cli/postinstall.ts` [MODIFY]

After the existing snapshot retrieval, add an optional registration ping:

```diff
  try {
    await retrieveSnapshot({ ... });
    console.log('[Strata] Postinstall snapshot retrieval complete.');
+
+   // Register build-time presence (non-blocking, best-effort)
+   try {
+     const os = await import('os');
+     const apiBase = config.apiBase || 'https://api-production-charisol-design-system-be.aws.charisol.io';
+     const headers: Record<string, string> = { 'Content-Type': 'application/json' };
+     if (syncToken) headers['Authorization'] = `Bearer ${syncToken}`;
+     
+     await fetch(`${apiBase}/v1/connect/${projectId}/register`, {
+       method: 'POST',
+       headers,
+       body: JSON.stringify({
+         hostname: os.hostname(),
+         sdkVersion: require('../../package.json').version,
+         environment: 'build',
+       }),
+     });
+     console.log('[Strata] Build-time registration complete.');
+   } catch (e) {
+     // Registration failure is non-fatal
+   }
  } catch (error: any) {
    console.warn(`[Strata] Postinstall snapshot failed: ${error.message || error}`);
  }
```

---

## Phase 3: Frontend — Dashboard UI

### 3.1 Add API Service Function

**File**: `services/connectedSurfaces.ts` [NEW]

```typescript
export async function fetchConnectedSurfaces(projectId: string): Promise<ConnectedSurface[]> {
  const res = await fetch(`/api/v1/projects/${projectId}/surfaces`, {
    headers: { Authorization: `Bearer ${getAuthToken()}` },
  });
  if (!res.ok) throw new Error(`Failed to fetch surfaces: ${res.status}`);
  const data = await res.json();
  return data.data.surfaces;
}
```

### 3.2 Update "See It in the Wild" Card

**File**: `app/explore/project/[id]/page.tsx` [MODIFY]

Replace the static `(project as any)?.connectedSurfaces || []` references with a `useEffect` that fetches from the new API endpoint:

```tsx
const [connectedSurfaces, setConnectedSurfaces] = useState<any[]>([]);
const [surfacesLoading, setSurfacesLoading] = useState(true);

useEffect(() => {
  if (!projectId) return;
  fetchConnectedSurfaces(projectId)
    .then(setConnectedSurfaces)
    .catch(() => setConnectedSurfaces([]))
    .finally(() => setSurfacesLoading(false));
}, [projectId]);
```

Update the rendering to show status badges:

```tsx
{connectedSurfaces.map((surface) => (
  <div key={surface.surfaceId} className="...">
    <div className="px-4 pb-4 flex flex-col gap-1">
      <p className="text-sm font-bold">{surface.displayName}</p>
      <p className="text-xs font-mono text-gray-400">{surface.origin}</p>
      <div className="flex items-center gap-1.5 mt-1">
        <span className={`w-2 h-2 rounded-full ${
          surface.status === 'live' ? 'bg-emerald-500' :
          surface.status === 'stale' ? 'bg-amber-400' : 'bg-gray-400'
        }`} />
        <span className={`text-xs font-medium ${
          surface.status === 'live' ? 'text-emerald-500' :
          surface.status === 'stale' ? 'text-amber-500' : 'text-gray-400'
        }`}>
          {surface.status === 'live' ? 'Live' :
           surface.status === 'stale' ? 'Stale' : 'Dormant'}
        </span>
        <span className="text-xs text-gray-400 ml-auto">
          v{surface.sdkVersion}
        </span>
      </div>
    </div>
  </div>
))}
```

### 3.3 Add Settings Tab (Connected Apps)

**File**: `components/sections/settings/ConnectedApps.tsx` [NEW]

A new settings sub-component that displays:
- Table of all connected surfaces with status, last seen, SDK version
- Inline edit for display name and emoji
- Delete button with confirmation
- "Manually Link Surface" form (URL + name + emoji)

---

## Phase 4: Polish & Documentation

### 4.1 Update NPM Package README

Add a new section documenting:
- The `reportUsage` prop and what data is sent
- How to opt out
- The `appName` prop
- What "Connected Apps" means in the Strata dashboard

### 4.2 Add CHANGELOG Entry

```markdown
## [2.2.0] - 2026-09-XX

### Added
- **Connected Apps Tracking**: StrataProvider now sends a lightweight heartbeat to report which
  production apps are consuming the design system. This powers the "See It in the Wild" dashboard
  section. Set `reportUsage={false}` to opt out.
- New props: `reportUsage`, `appName`, `environment`
- Build-time registration via postinstall (requires `strata.json`)
```

### 4.3 Error Handling Checklist

- [ ] Heartbeat never throws — all errors silently swallowed
- [ ] Heartbeat 401 disables reporter for the session
- [ ] Dashboard gracefully handles empty surfaces list
- [ ] Dashboard gracefully handles API errors (shows empty state)
- [ ] Build-time registration failure is non-fatal
- [ ] Max 100 surfaces enforced server-side
- [ ] Invalid origins rejected (non-URL values)

### 4.4 Testing Plan

| Test                              | Type        | Description                                             |
|:----------------------------------|:------------|:--------------------------------------------------------|
| Heartbeat sends on first sync     | Unit        | Mock fetch, verify POST called after sync succeeds      |
| Heartbeat respects opt-out        | Unit        | Set `reportUsage=false`, verify no fetch calls          |
| Heartbeat skips in SSR            | Unit        | Mock `typeof window === 'undefined'`, verify no calls   |
| Heartbeat skips in dev mode       | Unit        | Set `environment=development`, verify no calls          |
| Heartbeat 401 disables reporter   | Unit        | Mock 401 response, verify no subsequent calls           |
| Backend upsert creates surface    | Integration | POST heartbeat, verify DynamoDB record created          |
| Backend upsert updates existing   | Integration | POST heartbeat twice, verify single record updated      |
| Backend enforces 100-surface cap  | Integration | Create 100 surfaces, verify 101st rejected (429)        |
| Dashboard renders live badge      | E2E         | Create surface, load dashboard, verify green badge      |
| Dashboard renders stale badge     | E2E         | Create surface with old lastSeenAt, verify amber badge  |
| Manual link creates surface       | Integration | POST manual surface, verify DynamoDB record             |
| Delete removes surface            | Integration | DELETE surface, verify DynamoDB record removed          |

---

## Critical Design Decisions & Tradeoffs

### Decision 1: Heartbeat-Based Detection vs. CDN Log Analysis

**Chosen**: Heartbeat (active reporting from SDK)

| Approach         | Pros                                                       | Cons                                                      |
|:-----------------|:-----------------------------------------------------------|:----------------------------------------------------------|
| **Heartbeat**    | Real-time status, SDK version, app metadata, works with any CDN | Requires SDK changes, adds network calls, opt-out needed |
| **CDN Log Analysis** | Zero SDK changes, works retroactively on existing installs | CloudFront logs are delayed (minutes–hours), no app metadata, requires log pipeline (S3 + Lambda), can't distinguish browsers from bots |

**Rationale**: Heartbeat provides richer data (SDK version, sync mode, app name) with real-time freshness. CDN logs would require a separate data pipeline and only tell us "someone fetched strata.json from this IP" — not useful for the dashboard.

---

### Decision 2: Same DynamoDB Table vs. Separate Table

**Chosen**: Same table, different sort key prefix (`SURFACE#`)

| Approach       | Pros                                                  | Cons                                                     |
|:---------------|:------------------------------------------------------|:---------------------------------------------------------|
| **Same table** | No infrastructure changes, single query joins, zero cost | Schema sharing risks, must be careful with `saveUnknown` |
| **New table**  | Clean separation, independent scaling                 | Requires CloudFormation/Terraform changes, new table provisioning, cross-table queries for dashboard |

**Rationale**: Connected surfaces are tightly coupled to projects — they make no sense without a project context. Co-location in the same table enables single-query fetch and avoids infrastructure changes. The risk of schema bleed is mitigated by `saveUnknown: false` on the ConnectedSurface schema.

---

### Decision 3: Status Computed at Read Time vs. Stored with TTL

**Chosen**: Computed at read time

| Approach            | Pros                                            | Cons                                                  |
|:--------------------|:------------------------------------------------|:------------------------------------------------------|
| **Computed at read** | Simple, no background jobs, always fresh        | Slightly more CPU per dashboard read (negligible)     |
| **TTL + DynamoDB Streams** | Status always pre-computed, enables push notifications | Requires Streams + Lambda, complex to maintain, eventual consistency |

**Rationale**: The dashboard is viewed infrequently (a few times per day, by the project owner). Computing status from `lastSeenAt` for < 100 surfaces takes < 1ms. The simplicity of read-time computation far outweighs the operational complexity of DynamoDB Streams + Lambda.

---

### Decision 4: Opt-Out vs. Opt-In for Telemetry

**Chosen**: Opt-out (`reportUsage` defaults to `true`)

| Approach    | Pros                                                 | Cons                                                   |
|:------------|:-----------------------------------------------------|:-------------------------------------------------------|
| **Opt-out** | Feature works immediately, higher adoption            | Some users may object to default telemetry             |
| **Opt-in**  | Privacy-first, no surprises                           | Very low adoption (most devs won't enable manually)    |

**Rationale**: The data collected is minimal (public URLs, package version) and contains zero PII. Opt-out aligns with industry standard practice (React DevTools, Next.js telemetry, Vite telemetry). The opt-out is simple (`reportUsage={false}`) and clearly documented.

> [!WARNING]
> If Strata targets enterprise customers in regulated industries (healthcare, finance), consider switching to opt-in. Document the data collection in a dedicated privacy section of the README.

---

### Decision 5: Heartbeat Frequency — 5 Minutes vs. Per-Sync-Cycle

**Chosen**: Fixed 5-minute interval, independent of sync cycle

| Approach             | Pros                                              | Cons                                                 |
|:---------------------|:--------------------------------------------------|:-----------------------------------------------------|
| **Fixed 5-minute**   | Predictable load, easy to reason about             | May not reflect real-time status if sync is faster   |
| **Per-sync-cycle**   | Heartbeat matches actual usage pattern             | Could be every 3 seconds (default syncInterval) — far too frequent, ~28,800 heartbeats/day |

**Rationale**: Sending a heartbeat every 3 seconds (matching the default `syncInterval`) would generate enormous write load. A 5-minute interval provides sufficient freshness (stale detected within 10 minutes) while keeping costs negligible (~288 writes/day/surface).

---

### Decision 6: Surface ID from Origin Hash vs. Server-Generated UUID

**Chosen**: Deterministic hash of origin (`sha256(origin).substring(0, 16)`)

| Approach              | Pros                                             | Cons                                                |
|:----------------------|:-------------------------------------------------|:----------------------------------------------------|
| **Origin hash**       | Idempotent, no state needed on client            | Collision risk (mitigated by 64-bit entropy)        |
| **Server UUID**       | Guaranteed unique, can handle multiple apps on same origin | Client must store and resend the UUID, complexity |

**Rationale**: Idempotency is critical. If 10 users visit `https://app.acme.com`, all 10 browsers should update the *same* surface record, not create 10 separate entries. A deterministic hash achieves this without any client-side state management.

**Limitation**: Two different apps hosted on the same origin (e.g., `https://app.acme.com/crm` and `https://app.acme.com/hr`) will be treated as one surface. This is acceptable because:
1. Same-origin apps share the same design system deployment
2. The surface represents the *deployment*, not the *app*
3. If needed, the `appName` prop can distinguish them in the dashboard display

---

## File Summary

### New Files

| File | Location | Purpose |
|:-----|:---------|:--------|
| `ConnectedSurface.js` | `charisol-design-system-be/models/` | DynamoDB model for connected surfaces |
| `connectedSurfaceController.js` | `charisol-design-system-be/controllers/` | API controller for heartbeat, listing, management |
| `heartbeat.ts` | `strata-react/src/telemetry/` | SDK heartbeat reporter class |
| `ConnectedApps.tsx` | `charisol-design-system-fe/components/sections/settings/` | Dashboard settings UI for connected apps |
| `connectedSurfaces.ts` | `charisol-design-system-fe/services/` | Frontend API service for fetching surfaces |

### Modified Files

| File | Location | Changes |
|:-----|:---------|:--------|
| `index.js` | `charisol-design-system-be/routes/v1/` | Register new routes for heartbeat and surface management |
| `StrataProvider.tsx` | `strata-react/src/providers/` | Add heartbeat reporter lifecycle, new props |
| `index.ts` | `strata-react/src/` | Export HeartbeatReporter |
| `postinstall.ts` | `strata-react/src/cli/` | Add build-time registration ping |
| `rollup.config.js` | `strata-react/` | Add `@rollup/plugin-replace` for SDK_VERSION injection |
| `package.json` | `strata-react/` | Bump version, add `@rollup/plugin-replace` devDep |
| `page.tsx` | `charisol-design-system-fe/app/explore/project/[id]/` | Replace static connectedSurfaces with API fetch |
| `README.md` | `strata-react/` | Document reportUsage, appName, connected apps |
| `CHANGELOG.md` | `strata-react/` | Add 2.2.0 entry |
