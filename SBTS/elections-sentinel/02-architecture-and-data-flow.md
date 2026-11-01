# ElectionsSentinel (ES360) — System Architecture & Data Flow

**Document Version:** 1.0  
**Date:** 27 August 2026  
**Classification:** Internal — Engineering Reference  

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [System Context Diagram](#2-system-context-diagram)
3. [Technology Stack](#3-technology-stack)
4. [Application Architecture](#4-application-architecture)
5. [Backend Module Architecture](#5-backend-module-architecture)
6. [Data Flow Diagrams](#6-data-flow-diagrams)
7. [Database Architecture](#7-database-architecture)
8. [Security Architecture](#8-security-architecture)
9. [Offline-First Architecture](#9-offline-first-architecture)
10. [Infrastructure & Deployment Architecture](#10-infrastructure--deployment-architecture)
11. [Observability Architecture](#11-observability-architecture)

---

## 1. Architecture Overview

ES360 follows a **monolithic modular** architecture with a clear frontend/backend separation. This is a deliberate choice for the current scale — the domain boundaries are well-defined within NestJS modules, enabling future extraction to microservices if needed.

### Design Principles

1. **Election-Scoped Multi-Tenancy**: Every operational record is scoped to an `Election` entity. A single deployment serves multiple electoral events without data leakage.
2. **Geography-First Access Control**: Non-national roles are constrained to their assigned `GeographicUnit` subtree, enforced at the API layer, not the frontend.
3. **Fail-Closed Compliance**: Silence period enforcement, finance cap checks, and deployment gating default to blocking the action, not permitting it.
4. **Immutable Audit Trail**: Every mutation creates an append-only `AuditEvent`. Evidence custody events are never updated or deleted.
5. **Offline-First Data Capture**: Field operations assume zero connectivity. All write operations queue locally and sync when network is available.
6. **Advisory AI, Human Decision**: OCR pre-fills forms but never auto-commits. Language detection routes to human review when uncertain.

### High-Level Architecture

```mermaid
graph TB
    subgraph "Client Layer"
        WEB["Next.js Web App<br/>(App Router + Tailwind + Zustand)"]
        PWA["PWA/Mobile Web<br/>(Same codebase, offline-capable)"]
    end

    subgraph "API Layer"
        API["NestJS REST API<br/>(Port 4000)"]
        WS["WebSocket Gateway<br/>(Real-time push — planned)"]
    end

    subgraph "Data Layer"
        PG["PostgreSQL 15+<br/>(Primary data store)"]
        S3["S3-Compatible Storage<br/>(Evidence vault)"]
        REDIS["Redis<br/>(Session cache, rate limiting — planned)"]
    end

    subgraph "External Systems"
        IREV["INEC IReV Portal<br/>(Results comparison — planned)"]
        SMS["SMS Gateway<br/>(Agent notifications — planned)"]
        TOTP["TOTP Auth App<br/>(Google Authenticator)"]
    end

    WEB --> API
    PWA --> API
    API --> PG
    API --> S3
    API --> WS
    API -.-> REDIS
    API -.-> IREV
    API -.-> SMS
    WEB --> TOTP
```

---

## 2. System Context Diagram

```mermaid
C4Context
    title System Context — ElectionsSentinel (ES360)

    Person(director, "National/State Director", "Commands situation room operations")
    Person(agent, "Field/PU Agent", "Submits results, incidents, and evidence from polling units")
    Person(legal, "Legal Counsel", "Manages evidence holds, custody, and compliance")
    Person(finance, "Finance Lead", "Records expenditures and monitors spending ceilings")
    Person(analyst, "Intelligence Analyst", "Triages multilingual signals and narratives")

    System(es360, "ElectionsSentinel (ES360)", "Election operations, situation room command, and legal compliance platform")

    System_Ext(inec_irev, "INEC IReV", "Public election result viewing portal")
    System_Ext(totp_app, "TOTP Authenticator", "Google Authenticator / hardware key")
    System_Ext(sms_gw, "SMS Gateway", "Agent notifications and alerts")

    Rel(director, es360, "Monitors dashboard, manages agents, reviews results")
    Rel(agent, es360, "Submits EC8A forms, reports incidents, uploads evidence")
    Rel(legal, es360, "Applies legal holds, exports evidence with certificates")
    Rel(finance, es360, "Records transactions, monitors spending caps")
    Rel(analyst, es360, "Reviews intelligence signals, triages narratives")

    Rel(es360, inec_irev, "Compares captured results against published data")
    Rel(es360, sms_gw, "Sends alert notifications to field agents")
    Rel(totp_app, es360, "Provides TOTP codes for MFA")
```

---

## 3. Technology Stack

### 3.1 Current Stack

| Layer | Technology | Version | Purpose |
|-------|-----------|---------|---------|
| **Frontend Framework** | Next.js (App Router) | 15.x | Server-side rendering, routing, file-based pages |
| **Frontend Styling** | Tailwind CSS | 4.x | Utility-first CSS framework |
| **Frontend State** | Zustand | 5.x | Lightweight state management per module |
| **Frontend HTTP** | Axios | 1.7.x | API client with JWT interceptor |
| **Frontend Offline** | idb (IndexedDB) | 8.x | Offline write queue persistence |
| **Backend Framework** | NestJS | 11.x | Modular REST API with dependency injection |
| **Backend ORM** | TypeORM | 0.3.x | Entity mapping, migrations, query builder |
| **Database** | PostgreSQL | 15+ | Primary relational data store |
| **Authentication** | Passport + JWT | — | Strategy-based auth with JWT tokens |
| **MFA** | otplib + qrcode | 12.x | TOTP generation, QR code enrollment |
| **File Storage** | @aws-sdk/client-s3 | 3.x | S3-compatible evidence storage (MinIO / AWS S3 / local disk fallback) |
| **OCR** | Tesseract.js | 5.x | Client/server-side optical character recognition |
| **Security** | Helmet + ThrottlerModule | — | HTTP headers, rate limiting |
| **Password Hashing** | bcryptjs | 2.x | Argon2-equivalent password hashing |

### 3.2 Production Additions (Planned)

| Layer | Technology | Purpose |
|-------|-----------|---------|
| **Real-Time** | NestJS WebSocket Gateway (Socket.IO or ws) | Live dashboard push, SLA breach notifications |
| **Caching** | Redis 7+ | Session store, rate limiter backend, dashboard cache |
| **Logging** | Winston or Pino | Structured JSON logging with correlation IDs |
| **Health Checks** | @nestjs/terminus | Readiness/liveness probes for orchestration |
| **API Docs** | @nestjs/swagger | Auto-generated OpenAPI documentation |
| **Task Scheduling** | @nestjs/schedule | Recurring compliance rule generation, heartbeat checks |
| **PWA** | next-pwa or custom service worker | Offline asset caching for field app |
| **Monitoring** | Prometheus + Grafana | Metrics collection and visualization |
| **Containerization** | Docker + Docker Compose | Reproducible deployment packaging |
| **Orchestration** | Kubernetes or AWS ECS | Auto-scaling, blue-green deployments |
| **CI/CD** | GitHub Actions | Automated testing, linting, and deployment |
| **CDN** | Cloudflare or AWS CloudFront | Static asset delivery, DDoS protection |
| **Backup** | pgBackRest or AWS RDS automated backups | Point-in-time recovery |

---

## 4. Application Architecture

### 4.1 Frontend Architecture (Next.js)

```mermaid
graph TB
    subgraph "Next.js App Router"
        LAYOUT["Root Layout<br/>(layout.tsx)"]
        LOGIN["Login Page<br/>(/login)"]
        APP["Dashboard App<br/>(/app)"]
        MARKETING["Marketing Pages<br/>(/(marketing))"]
    end

    subgraph "State Layer (Zustand)"
        AUTH["authStore"]
        AGENTS["agentsStore"]
        INCIDENTS["incidentsStore"]
        RESULTS["resultsStore"]
        EVIDENCE["evidenceStore"]
        COMPLIANCE["complianceStore"]
        FINANCE["financeStore"]
        INTEL["intelligenceStore"]
        COMMS["commsStore"]
        GOVERNANCE["governanceStore"]
        DASHBOARD["dashboardStore"]
        GEO["geographyStore"]
        ELECTIONS["electionsStore"]
        CAMPAIGN["campaignStructureStore"]
    end

    subgraph "Data Layer"
        AXIOS["Axios Instance<br/>(JWT interceptor, 401 logout)"]
        OFFLINE["Offline Queue<br/>(IndexedDB via idb)"]
        SYNC["Sync Manager<br/>(online/offline events, 30s sweep)"]
        REPLAY["Replay Engine<br/>(synced/pending/conflict routing)"]
    end

    APP --> AUTH
    APP --> AGENTS
    APP --> INCIDENTS
    APP --> RESULTS
    APP --> EVIDENCE
    APP --> COMPLIANCE
    APP --> FINANCE
    APP --> INTEL
    APP --> COMMS
    APP --> GOVERNANCE
    APP --> DASHBOARD
    APP --> GEO
    APP --> ELECTIONS
    APP --> CAMPAIGN

    AUTH --> AXIOS
    AGENTS --> AXIOS
    INCIDENTS --> AXIOS
    RESULTS --> AXIOS
    EVIDENCE --> AXIOS

    AXIOS --> OFFLINE
    OFFLINE --> SYNC
    SYNC --> REPLAY
    REPLAY --> AXIOS
```

### 4.2 Backend Architecture (NestJS)

```mermaid
graph TB
    subgraph "HTTP Entry"
        REQ["Incoming HTTP Request"]
    end

    subgraph "Middleware Pipeline"
        HELMET["Helmet<br/>(Security headers)"]
        CORS["CORS<br/>(Origin filtering)"]
        THROTTLE["ThrottlerGuard<br/>(Rate limiting)"]
        VALIDATION["ValidationPipe<br/>(DTO whitelist + transform)"]
    end

    subgraph "Auth Pipeline"
        JWT_GUARD["JwtAuthGuard<br/>(Token validation)"]
        ROLES_GUARD["RolesGuard<br/>(RBAC check)"]
        GEO_GUARD["GeographyScopeGuard<br/>(Geographic boundary enforcement)"]
    end

    subgraph "Domain Modules"
        AUTH_MOD["AuthModule"]
        AGENTS_MOD["AgentsModule"]
        INCIDENTS_MOD["IncidentsModule"]
        RESULTS_MOD["ResultsModule"]
        EVIDENCE_MOD["EvidenceModule"]
        COMPLIANCE_MOD["ComplianceModule"]
        FINANCE_MOD["FinanceModule"]
        COMMS_MOD["CommsModule"]
        INTEL_MOD["IntelligenceModule"]
        GOV_MOD["GovernanceModule"]
        DASH_MOD["DashboardModule"]
        GEO_MOD["GeographyModule"]
        ELECT_MOD["ElectionsModule"]
        CAMP_MOD["CampaignStructureModule"]
        AUDIT_MOD["AuditModule"]
    end

    subgraph "Cross-Cutting Services"
        AUDIT_SVC["AuditService<br/>(Append-only event logging)"]
        STORAGE_SVC["StorageService<br/>(S3/local evidence vault)"]
        SILENCE_SVC["ElectionSilenceService<br/>(24h pre-polling enforcement)"]
        LANG_DET["LanguageDetector<br/>(5-language heuristic)"]
        OCR_SVC["OcrService<br/>(Tesseract.js form scanning)"]
        SLA_POLICY["SLA Policy<br/>(Severity-based timers)"]
        DEDUP["Dedup Engine<br/>(Title similarity check)"]
    end

    REQ --> HELMET --> CORS --> THROTTLE --> VALIDATION
    VALIDATION --> JWT_GUARD --> ROLES_GUARD --> GEO_GUARD

    GEO_GUARD --> AUTH_MOD
    GEO_GUARD --> AGENTS_MOD
    GEO_GUARD --> INCIDENTS_MOD
    GEO_GUARD --> RESULTS_MOD
    GEO_GUARD --> EVIDENCE_MOD
    GEO_GUARD --> COMPLIANCE_MOD
    GEO_GUARD --> FINANCE_MOD
    GEO_GUARD --> COMMS_MOD
    GEO_GUARD --> INTEL_MOD
    GEO_GUARD --> GOV_MOD
    GEO_GUARD --> DASH_MOD
    GEO_GUARD --> GEO_MOD
    GEO_GUARD --> ELECT_MOD
    GEO_GUARD --> CAMP_MOD
    GEO_GUARD --> AUDIT_MOD

    AGENTS_MOD --> AUDIT_SVC
    INCIDENTS_MOD --> AUDIT_SVC
    INCIDENTS_MOD --> DEDUP
    INCIDENTS_MOD --> SLA_POLICY
    RESULTS_MOD --> AUDIT_SVC
    RESULTS_MOD --> OCR_SVC
    EVIDENCE_MOD --> AUDIT_SVC
    EVIDENCE_MOD --> STORAGE_SVC
    COMMS_MOD --> SILENCE_SVC
    INTEL_MOD --> LANG_DET
    FINANCE_MOD --> AUDIT_SVC
```

---

## 5. Backend Module Architecture

Each NestJS module follows a consistent internal pattern:

```
module-name/
├── module-name.module.ts       # NestJS module declaration
├── module-name.controller.ts   # REST endpoint definitions
├── module-name.service.ts      # Business logic
├── entity-name.entity.ts       # TypeORM entity (database mapping)
├── dto/
│   ├── create-entity.dto.ts    # Create request validation
│   └── update-entity.dto.ts    # Update request validation
└── [domain-specific files]     # e.g., sla-policy.ts, language-detector.ts
```

### Module Dependency Graph

```mermaid
graph LR
    subgraph "Foundation"
        AUTH["AuthModule"]
        GEO["GeographyModule"]
        ELECT["ElectionsModule"]
        AUDIT["AuditModule"]
    end

    subgraph "Core Operations"
        AGENTS["AgentsModule"]
        INCIDENTS["IncidentsModule"]
        RESULTS["ResultsModule"]
        EVIDENCE["EvidenceModule"]
    end

    subgraph "Compliance & Finance"
        COMPLIANCE["ComplianceModule"]
        COMMS["CommsModule"]
        FINANCE["FinanceModule"]
    end

    subgraph "Intelligence & Command"
        INTEL["IntelligenceModule"]
        DASH["DashboardModule"]
        CAMPAIGN["CampaignStructureModule"]
    end

    subgraph "Post-Election"
        GOV["GovernanceModule"]
    end

    AGENTS --> AUTH
    AGENTS --> AUDIT
    INCIDENTS --> AUTH
    INCIDENTS --> AUDIT
    RESULTS --> AUTH
    RESULTS --> AUDIT
    EVIDENCE --> AUTH
    EVIDENCE --> AUDIT
    COMPLIANCE --> AUTH
    COMPLIANCE --> AUDIT
    COMPLIANCE --> ELECT
    FINANCE --> AUTH
    FINANCE --> AUDIT
    COMMS --> AUTH
    COMMS --> ELECT
    INTEL --> AUTH
    INTEL --> AUDIT
    DASH --> AGENTS
    DASH --> INCIDENTS
    DASH --> RESULTS
    DASH --> COMPLIANCE
    DASH --> GEO
    CAMPAIGN --> AUTH
    GOV --> AUTH
    GOV --> AUDIT
```

---

## 6. Data Flow Diagrams

### 6.1 Form EC8A Result Capture Flow

```mermaid
sequenceDiagram
    participant Agent as Field Agent (Mobile)
    participant IDB as IndexedDB Queue
    participant API as NestJS API
    participant DB as PostgreSQL
    participant S3 as Evidence Vault
    participant Dash as Dashboard

    Note over Agent: At Polling Unit after vote count

    Agent->>Agent: Photograph signed Form EC8A
    Agent->>API: POST /api/results/ocr-preview (image)
    API->>API: Tesseract.js OCR extraction
    API-->>Agent: Extracted fields + confidence scores

    Agent->>Agent: Review & confirm/correct OCR output
    Agent->>Agent: Enter result figures manually

    alt Online
        Agent->>API: POST /api/results (result data)
        API->>API: Mathematical integrity check
        Note over API: valid_votes + rejected_votes ≤ accredited_voters?
        alt Over-voting detected
            API->>API: Flag alert, reference §51(2)
        end
        API->>DB: INSERT result_forms
        API->>DB: INSERT audit_events
        API-->>Agent: 201 Created
    else Offline
        Agent->>IDB: Queue result submission
        Note over IDB: Stored with file blob
        IDB-->>Agent: Queued for sync
    end

    Note over Agent: Upload evidence photo
    Agent->>API: POST /api/evidence (file upload)
    API->>API: Compute SHA-256 hash
    API->>S3: PutObject (file bytes)
    API->>DB: INSERT evidence_items
    API->>DB: INSERT custody_events (Uploaded)
    API-->>Agent: 201 Created with hash

    Note over Dash: Dashboard auto-refreshes
    Dash->>API: GET /api/dashboard/summary
    API->>DB: Aggregate queries
    API-->>Dash: Updated metrics
```

### 6.2 Double-Blind Verification Flow

```mermaid
sequenceDiagram
    participant Op1 as Operator 1 (First Entry)
    participant Op2 as Operator 2 (Verifier)
    participant API as NestJS API
    participant DB as PostgreSQL

    Note over Op1: Enters result from paper form

    Op1->>API: POST /api/results
    API->>DB: INSERT result_forms (verification_status='Quality review')
    API-->>Op1: 201 Created

    Note over Op2: Independently enters same form (cannot see Op1's data)

    Op2->>API: POST /api/results/:id/verify
    API->>DB: SELECT original result_forms
    API->>API: Compare Op2 figures vs. original

    alt Figures match
        API->>DB: UPDATE result_forms SET verification_status='Verified'
        API->>DB: INSERT result_verifications (matches_original=true)
    else Figures differ
        API->>DB: UPDATE result_forms SET verification_status='Failed', reconciliation_status='Exception'
        API->>DB: INSERT result_verifications (matches_original=false)
        Note over API: Flagged for supervisory review
    end

    API->>DB: INSERT audit_events
    API-->>Op2: 200 OK with match result
```

### 6.3 Evidence Chain of Custody Flow

```mermaid
sequenceDiagram
    participant Agent as Field Agent
    participant API as NestJS API
    participant S3 as Evidence Vault
    participant DB as PostgreSQL
    participant Legal as Legal Team
    participant Court as Court/Tribunal

    Agent->>API: POST /api/evidence (file + metadata)
    API->>API: Compute SHA-256 hash
    API->>S3: Store file with hash metadata
    API->>DB: INSERT evidence_items (sha256, legal_hold=false)
    API->>DB: INSERT custody_events (action=Uploaded, actor=Agent)
    API-->>Agent: 201 Created

    Legal->>API: PATCH /api/evidence/:id (legal_hold=true)
    API->>DB: UPDATE evidence_items SET legal_hold=true
    API->>DB: INSERT custody_events (action=Legal hold applied, actor=Legal)
    Note over DB: Item now immutable — cannot be deleted or modified

    Legal->>API: POST /api/evidence/:id/transfer (to_email=external_counsel)
    API->>DB: INSERT custody_events (action=Transferred, to_email=external_counsel)

    Legal->>API: POST /api/evidence/:id/export
    API->>S3: GetObject (re-download file)
    API->>API: Recompute SHA-256 from downloaded bytes
    API->>API: Compare with stored hash

    alt Hash matches
        API->>DB: INSERT custody_events (action=Hash verified)
        API-->>Legal: Export manifest with confirmed integrity
    else Hash mismatch
        API->>DB: INSERT custody_events (action=Hash mismatch detected)
        API-->>Legal: Export manifest with MISMATCH WARNING
    end

    Note over Court: Legal presents certificate + custody chain
    Legal->>API: GET /api/evidence/:id/custody
    API->>DB: SELECT all custody_events ORDER BY created_at
    API-->>Legal: Full chain of custody log

    Note over Legal: Generate §84 Digital Evidence Certificate
    Legal->>Legal: Certificate includes:<br/>1. Device identification<br/>2. Regular use attestation<br/>3. Proper functioning attestation<br/>4. Data accuracy confirmation<br/>5. Signatory details
```

### 6.4 Incident SLA & Escalation Flow

```mermaid
sequenceDiagram
    participant Agent as Field Agent
    participant API as NestJS API
    participant DB as PostgreSQL
    participant Director as National Director

    Agent->>API: POST /api/incidents (severity=Critical)
    API->>API: Dedup check (same geography, last 6 hours, similar title)

    alt Duplicate detected
        API->>DB: INSERT incident (duplicate_status=Flagged, possible_duplicate_of=existing_id)
        Note over API: Reviewer must confirm or reject
    else No duplicate
        API->>DB: INSERT incident (duplicate_status=None)
    end

    API->>API: Compute SLA timers (Critical: 5 min ack, 2 hr resolve)
    API->>DB: INSERT audit_events
    API-->>Agent: 201 Created with SLA metadata

    Note over API: SLA clock starts at created_at

    Director->>API: GET /api/incidents
    API->>API: computeSla() for each incident
    API-->>Director: Incidents with slaState, ackDueAt, resolveDueAt

    Director->>API: POST /api/incidents/:id/acknowledge
    API->>DB: UPDATE incidents SET acked_at=NOW()
    Note over API: Ack SLA timer stopped

    alt Resolution within SLA
        Director->>API: PATCH /api/incidents/:id (status=Resolved)
        API->>DB: UPDATE incidents SET resolved_at=NOW(), status=Resolved
        Note over API: slaState = Resolved
    else SLA breached
        Note over API: slaState transitions:<br/>On track → At risk → Ack breached → Resolve breached
        Note over Director: Dashboard highlights breached SLAs in red
    end
```

### 6.5 Election Silence Enforcement Flow

```mermaid
sequenceDiagram
    participant Comms as Comms/Media Team
    participant API as NestJS API
    participant DB as PostgreSQL
    participant Legal as Legal Team

    Comms->>API: POST /api/comms (electionId, channel, subject, body)
    API->>DB: SELECT election WHERE id=electionId
    API->>API: ElectionSilenceService.isInSilencePeriod(election)
    Note over API: Check: Is current time within<br/>24h before polling through end of polling day?

    alt Outside silence period
        API->>DB: INSERT comms_messages (status=Sent)
        API-->>Comms: 201 Created
    else Inside silence period
        alt No override provided
            API->>DB: INSERT comms_messages (status='Blocked - Silence Period')
            API-->>Comms: 403 Forbidden (silence period active, §94(1))
        else Legal override with reason
            Legal->>API: POST /api/comms (silenceOverrideReason="Court order XYZ")
            API->>DB: INSERT comms_messages (status='Sent - Counsel Override', silence_override_by=Legal)
            API->>DB: INSERT audit_events (detail="Silence override: Court order XYZ")
            API-->>Legal: 201 Created (override logged)
        end
    end
```

### 6.6 Offline Sync Data Flow

```mermaid
sequenceDiagram
    participant App as Web App (Browser)
    participant IDB as IndexedDB
    participant Net as Network Monitor
    participant API as NestJS API

    Note over App: Agent is at rural PU — no connectivity

    App->>API: POST /api/incidents (fails — network error)
    App->>App: networkFailure.ts: Is this a network error?

    alt Network error (not validation error)
        App->>IDB: offlineDb.enqueue({method, url, body, blob})
        IDB-->>App: Queued (pending sync)
        App->>App: Update topbar: "1 pending sync"
    else Validation error (4xx)
        App-->>App: Surface error to user normally
    end

    Note over App: 30-second periodic sweep runs
    Net->>Net: Check navigator.onLine

    alt Still offline
        Note over Net: Skip replay, wait for next sweep
    else Online detected
        Net->>IDB: offlineDb.getAll()
        IDB-->>Net: [queued items]

        loop For each queued item
            Net->>API: Replay original request
            alt Success (2xx)
                Net->>IDB: offlineDb.remove(item.id)
                Note over Net: Status: synced
            else Transient failure (5xx, timeout)
                Note over Net: Status: pending (retry on next sweep)
            else Non-retryable (4xx validation)
                Net->>IDB: offlineDb.markConflict(item.id, error)
                Note over Net: Status: conflict (human review required)
            end
        end

        App->>App: Update topbar: "Synced" or "N conflicts"
    end
```

### 6.7 Campaign Finance Cap Check Flow

```mermaid
sequenceDiagram
    participant Finance as Finance Lead
    participant API as NestJS API
    participant DB as PostgreSQL

    Finance->>API: POST /api/finance/preview (election, office, amount)
    API->>DB: SUM(amountNgn) WHERE election_id=X AND office_type=Y
    API->>API: Compare current_total + new_amount vs. statutory_ceiling

    alt Within cap
        API-->>Finance: { withinCap: true, currentTotal, ceiling, remaining }
    else Would exceed cap
        API-->>Finance: { withinCap: false, currentTotal, ceiling, excess }
    end

    Finance->>API: POST /api/finance (full entry)

    alt Within cap
        API->>DB: INSERT finance_entries (cap_exceeded=false)
    else Over cap without approval
        API-->>Finance: 400 Bad Request (cap exceeded, approval required)
    else Over cap with capOverrideApproval
        API->>DB: INSERT finance_entries (cap_exceeded=true, approved_by=approver)
        API->>DB: INSERT audit_events (detail="Cap override: reason")
    end
```

---

## 7. Database Architecture

### 7.1 Schema Layout

```mermaid
graph TB
    subgraph "Foundation Schema"
        USERS["users"]
        REFRESH["refresh_tokens"]
        ELECTIONS["elections"]
        GEO["geographic_units"]
        AUDIT["audit_events"]
    end

    subgraph "Operations Schema"
        AGENTS["field_agents"]
        INCIDENTS["incidents"]
        RESULTS["result_forms"]
        VERIFICATIONS["result_verifications"]
        EVIDENCE["evidence_items"]
        CUSTODY["custody_events"]
    end

    subgraph "Compliance Schema"
        RULES["compliance_rules"]
        TASKS["compliance_tasks"]
        FINANCE["finance_entries"]
        COMMS["comms_messages"]
    end

    subgraph "Intelligence Schema"
        SIGNALS["intelligence_signals"]
    end

    subgraph "Structure Schema"
        ROLES["campaign_roles"]
        UNITS["command_units"]
        GOV["governance_cases"]
    end
```

### 7.2 Index Strategy

| Table | Index | Columns | Purpose |
|-------|-------|---------|---------|
| `geographic_units` | `IDX_geo_level_parent` | `level, parent_id` | Hierarchy traversal |
| `geographic_units` | `IDX_geo_path` | `path` | Geographic scope checks (LIKE 'NG/LA/%') |
| `geographic_units` | `UQ_geo_code` | `official_code` | UNIQUE — INEC code lookup |
| `field_agents` | `IDX_agent_election` | `election_id` | Election-scoped queries |
| `field_agents` | `IDX_agent_geo` | `geographic_unit_id` | Geography-scoped queries |
| `field_agents` | `IDX_agent_state` | `state` | State-level aggregation |
| `incidents` | `IDX_inc_severity` | `severity` | SLA-priority filtering |
| `incidents` | `IDX_inc_status` | `status` | Dashboard status counts |
| `result_forms` | `IDX_res_pu` | `polling_unit` | PU-level lookup |
| `result_forms` | `IDX_res_verification` | `verification_status` | Quality review filtering |
| `evidence_items` | `IDX_evi_sha256` | `sha256` | Hash-based integrity lookup |
| `evidence_items` | `IDX_evi_linked` | `linked_record_id` | Record-to-evidence association |
| `audit_events` | `IDX_audit_record` | `record_type, record_id` | Record-specific audit trail retrieval |
| `compliance_rules` | `IDX_rule_jurisdiction` | `jurisdiction_level, status` | Active rule filtering |
| `finance_entries` | `IDX_fin_election_office` | `election_id, office_type` | Spending cap aggregation |

### 7.3 Connection & Pooling Configuration

```
Production configuration targets:
├── Max connections: 100 (per API instance)
├── Connection timeout: 5,000ms
├── Idle timeout: 30,000ms
├── Statement timeout: 30,000ms (prevent runaway queries)
└── SSL: Required (sslmode=verify-full)
```

---

## 8. Security Architecture

### 8.1 Authentication Flow

```mermaid
sequenceDiagram
    participant Client as Browser
    participant API as NestJS API
    participant DB as PostgreSQL

    Client->>API: POST /api/auth/login (email, password)
    API->>DB: SELECT user WHERE email=?
    API->>API: bcrypt.compare(password, password_hash)

    alt Invalid credentials
        API-->>Client: 401 Unauthorized
    else Valid credentials
        API->>API: Check if role ∈ PRIVILEGED_ROLES

        alt MFA required but not enrolled
            API-->>Client: { mfaSetupRequired: true, otpauthUrl, qrCodeDataUrl, secret }
            Client->>Client: User enrolls TOTP app
            Client->>API: POST /api/auth/mfa/:userId/confirm (otp_code)
            API->>API: Verify TOTP code
            API->>DB: UPDATE user SET mfa_enabled=true
            API->>API: Generate JWT
            API-->>Client: { access_token, refresh_token }

        else MFA required and enrolled
            API-->>Client: { mfaRequired: true, userId }
            Client->>API: POST /api/auth/mfa/:userId/verify (otp_code)
            API->>API: Verify TOTP code
            API->>API: Generate JWT
            API-->>Client: { access_token, refresh_token }

        else MFA not required (non-privileged role)
            API->>API: Generate JWT
            API-->>Client: { access_token, refresh_token }
        end
    end
```

### 8.2 Authorization Pipeline

```
Request → Helmet (security headers)
        → CORS (origin check)
        → ThrottlerGuard (rate limit: 120 req/min)
        → ValidationPipe (DTO whitelist, forbid unknown properties)
        → JwtAuthGuard (token validation, user extraction)
        → RolesGuard (@Roles decorator — RBAC check)
        → GeographyScopeGuard (geographic boundary enforcement)
        → Controller method
```

### 8.3 Security Controls Summary

| Control | Implementation | Status |
|---------|---------------|--------|
| Password hashing | bcryptjs (cost factor 10) | ✅ Active |
| JWT access tokens | 12-hour expiry, HS256 | ✅ Active |
| Refresh tokens | 30-day expiry, stored hashed | ✅ Active |
| TOTP MFA | otplib, QR enrollment, 6-digit codes | ✅ Active |
| Rate limiting | NestJS ThrottlerModule (120 req/60s) | ✅ Active |
| Input validation | class-validator + whitelist + forbidNonWhitelisted | ✅ Active |
| Security headers | Helmet (HSTS, X-Frame-Options, CSP, etc.) | ✅ Active |
| RBAC | @Roles decorator + RolesGuard | ✅ Active |
| Geography scoping | GeographyScopeGuard on create operations | ✅ Active |
| Audit trail | Append-only audit_events on all mutations | ✅ Active |
| CORS | Currently `origin: true` | ⚠️ Needs hardening |
| MFA secret encryption | Plaintext in DB | ❌ Needs AES-256 |
| TLS | Not enforced at app level (infrastructure concern) | ⚠️ Needs infra config |
| SQL injection | TypeORM parameterized queries | ✅ Active |
| XSS | React auto-escaping + Helmet CSP | ✅ Active |

---

## 9. Offline-First Architecture

### 9.1 Component Overview

```mermaid
graph TB
    subgraph "Browser"
        UI["React UI Components"]
        STORE["Zustand Stores"]
        AXIOS["Axios Instance"]
        NET_CHECK["networkFailure.ts<br/>(Distinguish offline vs. validation error)"]
        IDB["offlineDb.ts<br/>(IndexedDB via idb library)"]
        REPLAY["replay.ts<br/>(Drain function: synced/pending/conflict)"]
        SYNC_MGR["syncManager.ts<br/>(Online/offline events, 30s sweep)"]
    end

    UI --> STORE
    STORE --> AXIOS
    AXIOS --> NET_CHECK

    NET_CHECK -->|Network error| IDB
    NET_CHECK -->|Validation error| UI

    SYNC_MGR --> IDB
    SYNC_MGR --> REPLAY
    REPLAY --> AXIOS

    SYNC_MGR -.->|online event| REPLAY
    SYNC_MGR -.->|30s timer| REPLAY
```

### 9.2 Offline Queue Item Schema

```typescript
interface QueuedItem {
  id: string;          // Auto-generated UUID
  method: string;      // HTTP method (POST, PATCH)
  url: string;         // API endpoint
  body: object;        // Request payload
  blob?: Blob;         // File attachment (evidence uploads)
  status: 'pending' | 'conflict';
  error?: string;      // Server rejection reason (conflicts only)
  createdAt: number;   // Timestamp
}
```

### 9.3 Replay Decision Logic

| Server Response | Classification | Action |
|----------------|----------------|--------|
| 2xx | **Synced** | Remove from queue |
| 5xx, timeout, network error | **Pending** | Keep in queue, retry on next sweep |
| 4xx (validation, auth) | **Conflict** | Mark as conflict, surface for human review |

---

## 10. Infrastructure & Deployment Architecture

### 10.1 Production Deployment (Target)

```mermaid
graph TB
    subgraph "Edge Layer"
        CDN["Cloudflare CDN<br/>(Static assets, DDoS protection)"]
        LB["Load Balancer<br/>(TLS termination, health checks)"]
    end

    subgraph "Application Layer"
        WEB_1["Next.js Instance 1"]
        WEB_2["Next.js Instance 2"]
        API_1["NestJS API Instance 1"]
        API_2["NestJS API Instance 2"]
    end

    subgraph "Data Layer"
        PG_PRIMARY["PostgreSQL Primary<br/>(Writes)"]
        PG_REPLICA["PostgreSQL Replica<br/>(Reads)"]
        S3["S3 Evidence Vault<br/>(Encrypted at rest)"]
        REDIS["Redis Cluster<br/>(Sessions, cache, rate limits)"]
    end

    subgraph "Observability"
        PROM["Prometheus<br/>(Metrics)"]
        GRAFANA["Grafana<br/>(Dashboards)"]
        LOGS["Centralized Logging<br/>(ELK/Loki)"]
    end

    CDN --> LB
    LB --> WEB_1
    LB --> WEB_2
    LB --> API_1
    LB --> API_2

    API_1 --> PG_PRIMARY
    API_2 --> PG_PRIMARY
    API_1 --> PG_REPLICA
    API_2 --> PG_REPLICA
    API_1 --> S3
    API_2 --> S3
    API_1 --> REDIS
    API_2 --> REDIS

    API_1 --> PROM
    API_2 --> PROM
    PROM --> GRAFANA
    API_1 --> LOGS
    API_2 --> LOGS
```

### 10.2 Environment Strategy

| Environment | Purpose | Database | Deployment |
|-------------|---------|----------|------------|
| **Local** | Developer workstation | PostgreSQL 15 (local), S3=local disk | `npm run start:dev` + `npm run dev` |
| **Staging** | Pre-production validation | PostgreSQL (cloud instance), S3 bucket | Docker Compose or K8s namespace |
| **Production** | Live election operations | PostgreSQL HA cluster, S3 encrypted bucket | K8s or ECS with auto-scaling |

### 10.3 Docker Compose (Current)

```
docker-compose.yml
├── postgres (port 5435→5432) — PostgreSQL 16 Alpine
├── minio (port 9002→9000, 9003→9001) — Evidence vault
├── api (port 4001→4000) — NestJS backend
└── web (port 3001→3000) — Next.js frontend
```

---

## 11. Observability Architecture

### 11.1 Logging Strategy (Target)

```
Log Levels:
├── ERROR — Unhandled exceptions, DB connection failures, storage errors
├── WARN  — SLA breaches, cap exceeded, hash mismatches, silence period blocks
├── INFO  — Request lifecycle, auth events, CRUD operations
├── DEBUG — Query execution, SLA computation details, language detection scores
└── TRACE — Raw HTTP request/response bodies (dev only)

Log Format:
{
  "timestamp": "2027-02-27T08:15:30.000Z",
  "level": "WARN",
  "correlationId": "req-abc123",
  "module": "FinanceService",
  "message": "Spending cap exceeded",
  "context": {
    "electionId": "EL-2027-PRES",
    "officeType": "President",
    "currentTotal": 9500000000,
    "ceiling": 10000000000,
    "newAmount": 600000000
  }
}
```

### 11.2 Key Metrics (Target)

| Metric | Type | Alert Threshold |
|--------|------|-----------------|
| `api_request_duration_seconds` | Histogram | p99 > 2s |
| `api_request_total` | Counter | Error rate > 5% |
| `db_connection_pool_active` | Gauge | > 80% utilization |
| `incidents_sla_breached_total` | Counter | Any Critical SLA breach |
| `results_overvoting_detected_total` | Counter | Any detection |
| `evidence_hash_mismatch_total` | Counter | Any mismatch |
| `offline_queue_pending_total` | Gauge | > 100 pending items |
| `auth_failed_login_total` | Counter | > 10/min from single IP |
