# Context Server · Component Drill-Down

**Type:** Graph intelligence layer + project registry — fleet-wide + per-job
**Technology:** PostgreSQL (project registry, tenancy), Neo4j (graph intelligence), Kuzu embedded (per-container Context Edge), MCP protocol
**Lifecycle:** Always-running fleet service
**Role:** Project/tenant management, cross-repo graph intelligence — patterns, dependencies, prior runs, RunState persistence

[← Back to System Overview](../../README.md) · [Context Edge (in-container)](../context-edge-server/README.md) · [Phase 3a flow context](../../phase-3-execution/phase-3a-agent-execution.md)

---

## Overview

The Context Server is the system's **long-term memory** and **project registry**. It has two distinct data domains:

1. **Project Registry (PostgreSQL)** — relational data: tenant management, project registration, repo membership, team assignments, access control, API key management, and billing/usage tracking. This is the administrative backbone that answers "who owns what" and "who can access what."

2. **Graph Intelligence (Neo4j)** — cross-repo intelligence that individual repos and containers don't have: which patterns work across services, how modules depend on each other, what previous runs discovered, and where port allocations live.

This polyglot persistence design uses each database for what it does best: PostgreSQL for relational/transactional concerns with row-level security for tenant isolation; Neo4j for graph traversals across the code intelligence layer.

The architecture has three tiers:

### Project Registry (PostgreSQL)
- **Technology:** PostgreSQL with row-level security (RLS)
- **Location:** Always running alongside the Orchestrator
- **Scope:** All tenants, projects, repos, teams, API keys
- **Role:** Administrative source of truth — who owns what, who can access what, usage tracking
- **Multi-tenancy:** Row-level security per tenant; each query is scoped automatically

### Graph Intelligence (Neo4j)
- **Technology:** Neo4j
- **Location:** Always running alongside the Orchestrator
- **Scope:** Full fleet graph — every repo, every module, every pattern, every run (tenant-scoped via project labels)
- **Writes:** Immediate consistency
- **Role:** Source of truth for cross-repo code intelligence

### Context Edge (Per-Container)
- **Technology:** Kuzu embedded database
- **Location:** Inside each DevContainer, ephemeral per job
- **Scope:** Scoped snapshot — only the repos/modules relevant to the current spec (single tenant)
- **Writes:** Buffered locally, flushed to Context Server at teardown
- **Role:** Fast local queries during agent execution via MCP protocol

The Context Edge exists because the DevContainer should not make live network calls to the Context Server during code generation — that would add latency and create a hard dependency. Instead, the Edge pulls a scoped snapshot at provision time, serves local queries, buffers writes, and flushes back at teardown. The Edge only ever receives data for its own tenant — tenant isolation is enforced at the snapshot export boundary.

---

## L3 — Component Diagram

### Three-Tier Architecture

```mermaid
graph TB
    subgraph fleet["Context Server (Fleet)"]
        direction TB

        subgraph pg_layer["Project Registry · PostgreSQL"]
            PG["PostgreSQL\n─────────────\n· Tenant management\n· Project registration\n· Repo membership\n· Team assignments\n· API key management\n· Usage / billing tracking\n· Row-level security (RLS)"]
        end

        subgraph neo_layer["Graph Intelligence · Neo4j"]
            NEO4J["Neo4j Database\n─────────────\n· Full fleet graph\n· All repos, modules, patterns\n· All run history\n· Immediate consistency\n· Tenant-scoped via\n  project labels"]
        end

        CS_API["Context Server API\n─────────────\n· Project CRUD (→ PG)\n· Tenant resolution (→ PG)\n· Snapshot export (→ Neo4j)\n· Write ingestion (→ Neo4j)\n· Graph query interface\n· Schema management"]
        CS_SYNC["Sync Coordinator\n─────────────\n· Scope calculator\n· Tenant-scoped snapshot\n· Snapshot packaging\n· Write conflict resolution\n· Flush ingestion"]
        TENANT["Tenant Resolver\n─────────────\n· API key → tenant\n· Repo → project → tenant\n· RLS policy enforcement\n· Cross-tenant isolation"]
    end

    subgraph container["Context Edge (Per-Container · Kuzu)"]
        direction TB
        KUZU["Kuzu Embedded DB\n─────────────\n· Scoped snapshot\n  (single tenant only)\n· Repos relevant to spec\n· Fast local queries\n· WAL recovery"]
        MCP_SRV["MCP Server\n─────────────\n· FastAPI + uvicorn\n· Tool definitions\n· Request routing"]
        WRITE_BUF["Write Buffer\n─────────────\n· Local write queue\n· Dedup on flush\n· Retry on failure"]
        SEEDER["Seeder\n─────────────\n· Repo-local graph\n  extraction\n· File → Module mapping\n· Dependency detection"]

        subgraph mcp_tools["MCP Tools (exposed to OpenCode + LangChain)"]
            T_ENRICH["context_query_enrichment\n· Patterns for module\n· Dependency edges\n· Prior run results"]
            T_LIVE["context_query_live\n· Port allocations\n· API surface\n· Active branch status"]
            T_WRITE["context_write_*\n· Task results\n· Discovered patterns\n· RunState updates"]
        end
    end

    %% PG ↔ Neo4j coordination
    CS_API --> PG
    CS_API --> NEO4J
    TENANT --> PG
    CS_SYNC --> NEO4J
    CS_SYNC --> TENANT

    %% Sync flow
    CS_SYNC -->|"1. Tenant-scoped\nsnapshot (at provision)"| KUZU
    WRITE_BUF -->|"2. Buffered writes\n(at teardown)"| CS_SYNC

    %% Query flow
    MCP_SRV --> KUZU
    MCP_SRV --> WRITE_BUF
    T_ENRICH --> MCP_SRV
    T_LIVE --> MCP_SRV
    T_WRITE --> MCP_SRV

    %% Consumers
    LC["LangChain\nOrchestrator"] -->|"Read #1\n(enrichment)"| T_ENRICH
    OC["OpenCode\nCLI"] -->|"Read #2\n(live query)"| T_LIVE
    LC -->|"Write #3\n(write-back)"| T_WRITE

    %% Seeding
    SEEDER -->|"seed from\nrepo-local files"| KUZU

    %% Orchestrator
    ORCH["Orchestrator"] -->|"project registration\njob state\nescalation events"| CS_API
    FO["FeatureOutlook"] -->|"project/repo queries\nteam membership"| CS_API

    style PG fill:#0e1220,stroke:#3b82f6,color:#d4d6de
    style NEO4J fill:#16140e,stroke:#f59e0b,color:#d4d6de
    style CS_API fill:#16140e,stroke:#f59e0b,color:#d4d6de
    style CS_SYNC fill:#16140e,stroke:#f59e0b,color:#d4d6de
    style TENANT fill:#0e1220,stroke:#3b82f6,color:#d4d6de
    style KUZU fill:#16140e,stroke:#d97706,color:#d4d6de
    style MCP_SRV fill:#16140e,stroke:#d97706,color:#d4d6de
    style WRITE_BUF fill:#16140e,stroke:#d97706,color:#d4d6de
    style SEEDER fill:#16140e,stroke:#d97706,color:#d4d6de
    style T_ENRICH fill:#111115,stroke:#888780,color:#d4d6de
    style T_LIVE fill:#111115,stroke:#888780,color:#d4d6de
    style T_WRITE fill:#111115,stroke:#888780,color:#d4d6de
    style LC fill:#0e1220,stroke:#3b82f6,color:#d4d6de
    style OC fill:#0e1220,stroke:#3b82f6,color:#d4d6de
    style ORCH fill:#0e1420,stroke:#22d3ee,color:#d4d6de
    style FO fill:#201c28,stroke:#a78bfa,color:#d4d6de
```

### Data Domain Split — PostgreSQL vs Neo4j

| Concern | Store | Rationale |
|---------|-------|-----------|
| Tenant registration | PostgreSQL | Relational CRUD, RLS isolation, ACID transactions |
| Project definitions | PostgreSQL | Foreign keys to tenant, structured metadata |
| Repo → project membership | PostgreSQL | Many-to-many relational join |
| Team assignments | PostgreSQL | User/role management, access control |
| API key management | PostgreSQL | Encrypted storage, tenant scoping |
| Usage tracking / billing | PostgreSQL | Append-only metered events, aggregation queries |
| Module dependency graph | Neo4j | Graph traversal (N-hop neighbor queries) |
| Code patterns | Neo4j | Pattern → module relationship traversal |
| Run history | Neo4j | Run → task → pattern discovery chains |
| Cross-repo intelligence | Neo4j | Subgraph export for Context Edge snapshots |
| Constitution rules | Neo4j | Graph-linked governance (rule → module scope) |
| RunState | Neo4j + file | Dual-write for resume reliability |

### Snapshot Sync Protocol

```mermaid
sequenceDiagram
    participant CS as Context Server (Neo4j)
    participant SYNC as Sync Coordinator
    participant CE as Context Edge (Kuzu)
    participant BUF as Write Buffer

    Note over CS,CE: ── Provision (snapshot pull) ──

    CE->>SYNC: Request snapshot for spec scope
    SYNC->>SYNC: Calculate scope (repos, modules, depth)
    SYNC->>CS: Export subgraph (Cypher → JSON)
    CS-->>SYNC: Scoped graph data
    SYNC-->>CE: Snapshot package

    CE->>CE: Load into Kuzu
    CE->>CE: Run seeder (repo-local extraction)

    Note over CS,CE: ── Execution (local queries + buffered writes) ──

    CE->>CE: Serve MCP queries locally (Kuzu)
    CE->>BUF: Buffer writes locally

    Note over CS,CE: ── Teardown (flush) ──

    CE->>BUF: Collect buffered writes
    BUF->>SYNC: Flush write batch
    SYNC->>SYNC: Conflict resolution (last-write-wins per entity)
    SYNC->>CS: Ingest writes into Neo4j

    alt Context Server unreachable
        BUF->>BUF: Persist to context-data volume
        Note over BUF: Next container run picks up<br/>stale data + unflushed writes
    end
```

### Degradation Modes

```mermaid
graph TB
    subgraph full["Full Mode"]
        F_DESC["Context Edge + Context Server\n─────────────\n· Scoped snapshot from Neo4j\n· All MCP tools active\n· Write buffer flushes at teardown\n· RunState dual-write"]
    end

    subgraph stale["Stale Mode"]
        S_DESC["Stale Context Provider\n─────────────\n· Local snapshot from volume\n  (may be outdated)\n· RunState from local file\n  (resume works)\n· No enrichment queries\n· Writes buffer locally\n  (flush next run)"]
    end

    subgraph null_mode["Null Mode"]
        N_DESC["Null Context Provider\n─────────────\n· No enrichment\n· No live queries\n· No write-back\n· No resume\n· Day-1 experience\n· Pipeline works from spec only"]
    end

    DETECT{"Context Edge\navailable?"}
    DETECT -->|"Kuzu + Server"| full
    DETECT -->|"Local data only"| stale
    DETECT -->|"Nothing"| null_mode

    style full fill:#0e1610,stroke:#34d399,color:#d4d6de
    style stale fill:#1e1600,stroke:#f59e0b,color:#d4d6de
    style null_mode fill:#111115,stroke:#888780,color:#d4d6de
```

---

## L4 — Code Level

### Project Registry Schema (PostgreSQL)

The relational layer manages tenants, projects, repos, and access control. Row-level security (RLS) ensures every query is automatically scoped to the caller's tenant.

```mermaid
erDiagram
    Tenant {
        uuid id PK
        string name
        string slug
        string plan
        timestamp created_at
        boolean active
    }
    Project {
        uuid id PK
        uuid tenant_id FK
        string name
        string description
        string default_branch
        timestamp created_at
    }
    RepoRegistration {
        uuid id PK
        uuid project_id FK
        string repo_url
        string repo_name
        string default_branch
        string stack
        string criticality
        timestamp registered_at
    }
    Team {
        uuid id PK
        uuid tenant_id FK
        string name
        string role
    }
    TeamMember {
        uuid id PK
        uuid team_id FK
        string user_id
        string email
        string role
    }
    ApiKey {
        uuid id PK
        uuid tenant_id FK
        string key_hash
        string label
        string scope
        timestamp created_at
        timestamp expires_at
        boolean active
    }
    UsageEvent {
        uuid id PK
        uuid tenant_id FK
        uuid project_id FK
        string event_type
        int tokens_used
        string model
        timestamp created_at
    }

    Tenant ||--o{ Project : "owns"
    Tenant ||--o{ Team : "has"
    Tenant ||--o{ ApiKey : "authenticates via"
    Project ||--o{ RepoRegistration : "contains"
    Team ||--o{ TeamMember : "includes"
    Tenant ||--o{ UsageEvent : "generates"
    Project ||--o{ UsageEvent : "tracked per"
```

### PostgreSQL DDL (key tables)

```sql
-- Tenant isolation via RLS
CREATE TABLE tenants (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name        TEXT NOT NULL,
    slug        TEXT UNIQUE NOT NULL,
    plan        TEXT NOT NULL DEFAULT 'starter',
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    active      BOOLEAN NOT NULL DEFAULT true
);

CREATE TABLE projects (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            TEXT NOT NULL,
    description     TEXT,
    default_branch  TEXT NOT NULL DEFAULT 'main',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name)
);

CREATE TABLE repo_registrations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id),
    repo_url        TEXT NOT NULL,
    repo_name       TEXT NOT NULL,
    default_branch  TEXT NOT NULL DEFAULT 'main',
    stack           TEXT,          -- 'node', 'java', 'python', etc.
    criticality     TEXT NOT NULL DEFAULT 'standard',  -- 'low', 'standard', 'high', 'critical'
    registered_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (project_id, repo_name)
);

CREATE TABLE api_keys (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id   UUID NOT NULL REFERENCES tenants(id),
    key_hash    TEXT NOT NULL,       -- bcrypt hash, never store plaintext
    label       TEXT NOT NULL,
    scope       TEXT NOT NULL DEFAULT 'full',  -- 'full', 'read', 'ci'
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at  TIMESTAMPTZ,
    active      BOOLEAN NOT NULL DEFAULT true
);

CREATE TABLE usage_events (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id   UUID NOT NULL REFERENCES tenants(id),
    project_id  UUID REFERENCES projects(id),
    event_type  TEXT NOT NULL,       -- 'codegen', 'enrichment', 'validation', 'scan'
    tokens_used INT,
    model       TEXT,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Row-Level Security
ALTER TABLE projects ENABLE ROW LEVEL SECURITY;
ALTER TABLE repo_registrations ENABLE ROW LEVEL SECURITY;
ALTER TABLE api_keys ENABLE ROW LEVEL SECURITY;
ALTER TABLE usage_events ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation_projects ON projects
    USING (tenant_id = current_setting('app.current_tenant_id')::uuid);

CREATE POLICY tenant_isolation_repos ON repo_registrations
    USING (project_id IN (
        SELECT id FROM projects
        WHERE tenant_id = current_setting('app.current_tenant_id')::uuid
    ));

CREATE POLICY tenant_isolation_keys ON api_keys
    USING (tenant_id = current_setting('app.current_tenant_id')::uuid);

CREATE POLICY tenant_isolation_usage ON usage_events
    USING (tenant_id = current_setting('app.current_tenant_id')::uuid);
```

### Tenant Resolution Flow

```mermaid
sequenceDiagram
    participant CLIENT as API Client
    participant API as Context Server API
    participant TR as Tenant Resolver
    participant PG as PostgreSQL
    participant NEO as Neo4j

    CLIENT->>API: Request (API key in header)
    API->>TR: Resolve tenant from API key
    TR->>PG: SELECT tenant_id FROM api_keys WHERE key_hash = $hash
    PG-->>TR: tenant_id
    TR->>PG: SET app.current_tenant_id = $tenant_id (session var)
    Note over PG: RLS now active — all queries scoped to tenant

    alt Graph query
        API->>NEO: Cypher with tenant label filter
        Note over NEO: WHERE m.tenant_id = $tenant_id
        NEO-->>API: Scoped results
    else Project CRUD
        API->>PG: SQL query (RLS auto-scopes)
        PG-->>API: Tenant-scoped results
    end

    API-->>CLIENT: Response
```

### Graph Schema (Kuzu / Neo4j)

Both graph tiers use the same schema. Neo4j nodes carry a `tenant_id` property for tenant isolation. Kuzu snapshots are pre-filtered to a single tenant — no tenant_id needed in Kuzu queries.

```mermaid
erDiagram
    Module {
        string name PK
        string repo
        string stack
        string path
    }
    Pattern {
        string id PK
        string content
        float confidence
        string category
    }
    Task {
        string id PK
        string type
        string title
        string status
        string specId
    }
    Run {
        string id PK
        timestamp timestamp
        string status
        int64 duration
        string specId
    }
    Constitution {
        string rule PK
        int64 priority
        string scope
    }
    RunState {
        string id PK
        int64 task_index
        int64 retry_count
        timestamp updated_at
    }
    Dependency {
        string name PK
        string version
        string type
    }

    Module ||--o{ Module : "DEPENDS_ON"
    Pattern }o--o{ Module : "APPLIES_TO"
    Run ||--o{ Task : "EXECUTED"
    Module ||--o{ Dependency : "USES"
    Run ||--o{ Pattern : "DISCOVERED"
```

### Cypher Schema DDL (Neo4j — tenant-scoped)

All node tables include `tenant_id` for tenant isolation. Queries from the Context Server API always filter by the resolved tenant.

```cypher
CREATE NODE TABLE Module(name STRING, repo STRING, stack STRING, path STRING, tenant_id STRING, PRIMARY KEY(name));
CREATE NODE TABLE Pattern(id STRING, content STRING, confidence FLOAT, category STRING, tenant_id STRING, PRIMARY KEY(id));
CREATE NODE TABLE Task(id STRING, type STRING, title STRING, status STRING, specId STRING, tenant_id STRING, PRIMARY KEY(id));
CREATE NODE TABLE Run(id STRING, timestamp TIMESTAMP, status STRING, duration INT64, specId STRING, tenant_id STRING, PRIMARY KEY(id));
CREATE NODE TABLE Constitution(rule STRING, priority INT64, scope STRING, tenant_id STRING, PRIMARY KEY(rule));
CREATE NODE TABLE RunState(id STRING, task_index INT64, retry_count INT64, updated_at TIMESTAMP, tenant_id STRING, PRIMARY KEY(id));
CREATE NODE TABLE Dependency(name STRING, version STRING, type STRING, tenant_id STRING, PRIMARY KEY(name));

CREATE REL TABLE DEPENDS_ON(FROM Module, TO Module);
CREATE REL TABLE APPLIES_TO(FROM Pattern, TO Module);
CREATE REL TABLE EXECUTED(FROM Run, TO Task);
CREATE REL TABLE USES(FROM Module, TO Dependency);
CREATE REL TABLE DISCOVERED(FROM Run, TO Pattern);

-- All queries include tenant filter:
-- MATCH (m:Module {tenant_id: $tenantId}) WHERE m.repo IN $repos RETURN m
```

### Context Edge (In-Container)

The Context Edge is a separate component that runs inside each DevContainer. It pulls a tenant-scoped snapshot from this Context Server, serves MCP queries locally via Kuzu, and flushes buffered writes back at teardown.

For full details on the Context Edge's internal architecture — MCP tool definitions, write buffer, seeder, degradation modes, and ContextProvider interface — see the dedicated component doc:

**[→ Context Edge Component](../context-edge-server/README.md)**

### Scope Calculator (Tenant-Aware)

When the Context Edge requests a snapshot, the Sync Coordinator resolves the tenant from the API key, then calculates what to include — scoped to that tenant only.

```python
class ScopeCalculator:
    def calculate(self, spec: Spec, tenant_id: str,
                  context_server: Neo4jClient) -> Scope:
        # 1. Direct repos from the spec (validated against tenant's registered repos)
        direct_repos = spec.repos  # e.g., ["api-gateway", "web-app"]

        # 2. Dependency neighbors (1-hop, tenant-scoped)
        dep_repos = set()
        for repo in direct_repos:
            deps = context_server.query(
                "MATCH (m:Module {repo: $repo, tenant_id: $tid})"
                "-[:DEPENDS_ON]->(d:Module {tenant_id: $tid}) "
                "RETURN DISTINCT d.repo",
                repo=repo, tid=tenant_id
            )
            dep_repos.update(deps)

        # 3. All modules in scoped repos (tenant-scoped)
        all_repos = direct_repos | dep_repos
        modules = context_server.query(
            "MATCH (m:Module {tenant_id: $tid}) "
            "WHERE m.repo IN $repos RETURN m",
            repos=list(all_repos), tid=tenant_id
        )

        # 4. Patterns that apply to any scoped module (tenant-scoped)
        patterns = context_server.query(
            "MATCH (p:Pattern {tenant_id: $tid})-[:APPLIES_TO]->"
            "(m:Module {tenant_id: $tid}) "
            "WHERE m.repo IN $repos RETURN p",
            repos=list(all_repos), tid=tenant_id
        )

        # 5. Recent runs (tenant-scoped)
        runs = context_server.query(
            "MATCH (r:Run {tenant_id: $tid})-[:EXECUTED]->(t:Task) "
            "WHERE t.specId CONTAINS ANY(repo IN $repos) "
            "RETURN r ORDER BY r.timestamp DESC LIMIT 50",
            repos=list(all_repos), tid=tenant_id
        )

        return Scope(tenant_id=tenant_id, repos=all_repos,
                     modules=modules, patterns=patterns, runs=runs)
```

### Key Design Decisions

**Why PostgreSQL + Neo4j (polyglot persistence)?**
Tenant management, project registration, and access control are inherently relational: foreign keys, unique constraints, row-level security, ACID transactions. Neo4j would require workarounds for all of these. Conversely, graph traversals ("find all modules 2 hops from api-gateway that use Spring Security") are Neo4j's strength — PostgreSQL's recursive CTEs work but are slower and harder to express. Each database handles what it's best at. The Context Server API is the facade that routes to the right store.

**Why PostgreSQL RLS for tenant isolation (not application-level filtering)?**
Application-level `WHERE tenant_id = ?` is fragile — a missing filter in one query leaks data across tenants. RLS enforces isolation at the database level: every query is automatically scoped by `SET app.current_tenant_id`. Even a bug in the API layer cannot bypass RLS. This is the same pattern used by Supabase, Neon, and other multi-tenant SaaS platforms.

**Why tenant_id on Neo4j nodes (not separate databases)?**
Separate Neo4j instances per tenant would provide the strongest isolation but at significant operational cost (N databases to manage, no cross-tenant analytics). A shared database with `tenant_id` labels on every node provides sufficient isolation for the Context Server's use case — code patterns and dependency graphs don't contain PII or financial data. The Sync Coordinator always filters by tenant_id before exporting snapshots.

**Why scoped snapshot export (not full graph access)?**
A DevContainer running for 30 minutes doesn't need — and shouldn't query — the full fleet. The Sync Coordinator exports a tenant-scoped subgraph (direct repos + 1-hop dependency neighbors) as JSON. This keeps the Context Edge lightweight while providing sufficient cross-repo awareness. For details on the Edge's internal architecture, see [Context Edge Component](../context-edge-server/README.md).

**Why 1-hop dependency neighbors in the snapshot?**
If `api-gateway` depends on `shared-lib`, and the spec modifies `api-gateway`, the agent needs to know `shared-lib`'s API surface to generate correct code. The 1-hop neighbor expansion ensures the snapshot contains enough context without pulling the entire fleet graph.

**Why last-write-wins conflict resolution on flush?**
Multiple DevContainers may write to the same pattern or module concurrently. Sophisticated conflict resolution (CRDTs, vector clocks) adds complexity for low payoff — the writes are typically pattern discoveries and run results, where the latest observation is the most relevant.

### Multi-Tenancy Boundaries

```
┌─────────────────────────────────────────────────────────────┐
│  Tenant A (Acme Corp)                                       │
│                                                             │
│  PostgreSQL (RLS)          Neo4j (tenant_id label)          │
│  ┌──────────────────┐     ┌──────────────────────────┐      │
│  │ projects          │     │ Module {tenant_id: "A"}  │      │
│  │ repo_registrations│     │ Pattern {tenant_id: "A"} │      │
│  │ teams             │     │ Run {tenant_id: "A"}     │      │
│  │ api_keys          │     │                          │      │
│  │ usage_events      │     │ (graph traversals only   │      │
│  └──────────────────┘     │  see tenant A nodes)     │      │
│                           └──────────────────────────┘      │
│                                                             │
│  Context Edge (Kuzu) ← snapshot contains only tenant A data │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  Tenant B (Widgets Inc)                                     │
│  (same databases, different RLS scope / tenant_id label)    │
│  Complete isolation — tenant A cannot see tenant B's data   │
└─────────────────────────────────────────────────────────────┘
```
