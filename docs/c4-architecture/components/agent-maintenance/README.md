# Maintenance Agent · Component Drill-Down

**Type:** Standalone CLI tool — scheduled execution
**Technology:** CLI (Node/Python), scanner adapters, spec-kit maintenance preset
**Lifecycle:** Invoked on schedule (cron / Harness pipeline / Orchestrator trigger) — runs, produces bundle, exits
**Deployment:** Installed as a CLI tool; executed as a scheduled job
**Role:** Scans codebase for health issues, triages findings, generates spec-kit maintenance bundles for human approval

[← Back to System Overview](../../README.md) · [Phase M flow context](../../phase-m-maintenance/README.md)

---

## Overview

The Maintenance Agent is a **standalone CLI tool** — similar in spirit to tools like [Janitor](https://github.com/nicholasgasior/janitor) — that scans codebases for health issues, triages findings, and produces spec-kit maintenance bundles. It is not a long-running service. It runs, does its work, registers results with the Orchestrator, and exits.

### Execution Model

```
┌──────────────┐     ┌──────────────────────┐     ┌──────────────┐
│  Scheduler    │     │  Maintenance Agent    │     │ Orchestrator │
│              │     │  (CLI)                │     │              │
│  cron        │────▶│  scan → triage →      │────▶│  Registers   │
│  Harness     │     │  bundle → register    │     │  maintenance │
│  Orchestrator│     │  → exit               │     │  job         │
└──────────────┘     └──────────────────────┘     └──────────────┘
```

The CLI can be invoked by:
- **Cron job** — `0 2 * * 1` (weekly Monday 2am)
- **Harness pipeline** — scheduled pipeline step
- **Orchestrator trigger** — on-demand via API (e.g., after a critical CVE disclosure)
- **Manual** — developer runs `maintenance-agent scan --repos api-gateway,web-app`

### What It Produces

The Maintenance Agent does not fix anything. It **discovers, triages, and packages**. The output is a spec-kit maintenance bundle — a set of artifacts that describe what needs fixing, with acceptance criteria and test expectations. Humans approve the bundle in FeatureOutlook before any autonomous execution happens.

---

## L3 — Component Diagram

### CLI Architecture

```mermaid
graph TB
    subgraph cli["Maintenance Agent CLI"]
        direction TB

        subgraph commands["Commands"]
            SCAN_CMD["scan\n─────────────\n· Run all scanners\n· Target: repo list or fleet\n· Output: scan-report.json"]
            TRIAGE_CMD["triage\n─────────────\n· Score + dedup + group\n· Conflict detection\n· Output: triage-report.json"]
            BUNDLE_CMD["bundle\n─────────────\n· Generate spec-kit artifacts\n· maintenance preset\n· Output: maintenance bundle"]
            REGISTER_CMD["register\n─────────────\n· Push bundle to Orchestrator\n· Create maintenance job\n· Output: job ID"]
            RUN_CMD["run\n─────────────\n· Full pipeline:\n  scan → triage → bundle\n  → register\n· Default command"]
        end

        subgraph scanners["Scanner Adapters"]
            QOD["Qodana\n─────────────\n· Code quality\n· Code smells\n· Complexity"]
            SNYK["Snyk\n─────────────\n· Dependency vulns\n· License risk\n· Container scan"]
            SONAR["SonarQube\n─────────────\n· Static analysis\n· Coverage gaps\n· Duplications"]
            CONV["Convention Analyzer\n─────────────\n· Custom rules\n· Naming standards\n· Architectural fit"]
        end

        subgraph core["Core"]
            NORMALIZER["Finding Normalizer\n─────────────\n· Common severity scale\n· Unified location format\n· Scanner-agnostic ID\n· Dedup fingerprint"]
            TRIAGE_ENGINE["Triage Engine\n─────────────\n· Severity scoring\n· Dedup across scanners\n· Group by repo/module\n· Rank by impact+complexity\n· Conflict detection"]
            SPECGEN["Spec-Kit Generator\n─────────────\n· maintenance preset\n· acceptance-criteria.md\n  per finding\n· qa-tests.md per finding\n· No ux.md"]
        end

        subgraph integration["Integration"]
            ORCH_CLIENT["Orchestrator Client\n─────────────\n· Register maintenance job\n· Query active jobs\n  (conflict check)\n· Fetch repo registry"]
            REPO_ACCESS["Repo Access\n─────────────\n· Clone / fetch repos\n· Read package.json / pom.xml\n· Provide to scanners"]
        end
    end

    SCAN_CMD --> QOD & SNYK & SONAR & CONV
    QOD & SNYK & SONAR & CONV --> NORMALIZER
    TRIAGE_CMD --> TRIAGE_ENGINE
    NORMALIZER --> TRIAGE_ENGINE
    TRIAGE_ENGINE --> ORCH_CLIENT
    BUNDLE_CMD --> SPECGEN
    TRIAGE_ENGINE --> SPECGEN
    REGISTER_CMD --> ORCH_CLIENT
    RUN_CMD --> SCAN_CMD
    RUN_CMD --> TRIAGE_CMD
    RUN_CMD --> BUNDLE_CMD
    RUN_CMD --> REGISTER_CMD

    ORCH_CLIENT --> ORCH["Orchestrator API"]
    REPO_ACCESS --> GH["GitHub"]

    style SCAN_CMD fill:#0e1420,stroke:#38bdf8,color:#d4d6de
    style TRIAGE_CMD fill:#0e1420,stroke:#38bdf8,color:#d4d6de
    style BUNDLE_CMD fill:#0e1420,stroke:#38bdf8,color:#d4d6de
    style REGISTER_CMD fill:#0e1420,stroke:#38bdf8,color:#d4d6de
    style RUN_CMD fill:#0e1420,stroke:#38bdf8,color:#d4d6de
    style QOD fill:#111115,stroke:#f97316,color:#f97316
    style SNYK fill:#111115,stroke:#4ade80,color:#4ade80
    style SONAR fill:#111115,stroke:#38bdf8,color:#7dd3fc
    style CONV fill:#111115,stroke:#fbbf24,color:#fbbf24
    style NORMALIZER fill:#0e1420,stroke:#38bdf8,color:#d4d6de
    style TRIAGE_ENGINE fill:#0e1420,stroke:#38bdf8,color:#d4d6de
    style SPECGEN fill:#0e1420,stroke:#38bdf8,color:#d4d6de
    style ORCH_CLIENT fill:#0e1420,stroke:#22d3ee,color:#d4d6de
    style REPO_ACCESS fill:#111115,stroke:#888780,color:#d4d6de
    style ORCH fill:#0e1420,stroke:#22d3ee,color:#d4d6de
    style GH fill:#111115,stroke:#888780,color:#d4d6de
```

### CLI Commands

```
maintenance-agent <command> [options]

Commands:
  scan        Run scanners against target repos
  triage      Triage scan results (score, dedup, group, conflict-check)
  bundle      Generate spec-kit maintenance bundle from triaged findings
  register    Register bundle with Orchestrator as a maintenance job
  run         Full pipeline: scan → triage → bundle → register (default)

Options:
  --repos <list>          Target repos (comma-separated, or "all")
  --scanners <list>       Scanners to run (default: all available)
  --severity <min>        Minimum severity to include (default: medium)
  --output <dir>          Output directory for reports/bundle
  --orchestrator <url>    Orchestrator API URL
  --dry-run               Generate bundle but don't register with Orchestrator
  --config <path>         Config file path (default: .maintenance-agent.json)
```

### Full Pipeline Sequence

```mermaid
sequenceDiagram
    participant SCHED as Scheduler (cron / Harness)
    participant CLI as Maintenance Agent CLI
    participant SCAN as Scanner Adapters
    participant ORCH as Orchestrator
    participant GH as GitHub

    SCHED->>CLI: maintenance-agent run --repos all

    rect rgb(15, 18, 32)
    Note over CLI: scan
    CLI->>GH: Fetch/clone target repos
    CLI->>SCAN: Run Qodana, Snyk, SonarQube, Convention Analyzer
    SCAN-->>CLI: Raw scan results per scanner
    CLI->>CLI: Normalize findings (common severity, dedup fingerprint)
    CLI->>CLI: Write scan-report.json
    end

    rect rgb(30, 25, 15)
    Note over CLI: triage
    CLI->>ORCH: GET /api/jobs?status=active (conflict check)
    ORCH-->>CLI: Active job list
    CLI->>CLI: Dedup across scanners
    CLI->>CLI: Score by impact + complexity
    CLI->>CLI: Group by repo / module
    CLI->>CLI: Flag conflicts with active jobs
    CLI->>CLI: Write triage-report.json
    end

    rect rgb(15, 22, 16)
    Note over CLI: bundle
    CLI->>CLI: Generate spec-kit artifacts per finding
    CLI->>CLI: maintenance preset (acceptance-criteria.md + qa-tests.md)
    CLI->>CLI: Package as maintenance bundle
    end

    rect rgb(20, 15, 25)
    Note over CLI: register
    CLI->>ORCH: POST /api/jobs/maintenance (bundle + triage summary)
    ORCH->>ORCH: Create maintenance job (SCAN_COMPLETE state)
    ORCH->>ORCH: Assign risk tier per finding
    ORCH-->>CLI: Job ID
    CLI->>CLI: Log: "Registered maintenance job {id}, awaiting approval"
    end

    CLI->>CLI: Exit 0

    Note over ORCH: Human reviews in FeatureOutlook<br/>Approves/rejects per finding<br/>Approved findings → Phase 3
```

---

## L4 — Code Level

### Scanner Adapter Interface

All scanners implement the same interface. New scanners (Trivy, Semgrep, etc.) plug in without changing the CLI core.

```mermaid
classDiagram
    class ScannerAdapter {
        <<interface>>
        +name() String
        +scan(repos: RepoRef[]) ScanResult
        +isAvailable() boolean
    }

    class ScanResult {
        +String scannerName
        +Instant timestamp
        +Finding[] findings
        +ScanStatus status
        +Duration elapsed
    }

    class Finding {
        +String id
        +String scanner
        +Severity severity
        +String title
        +String description
        +Location location
        +String fingerprint
        +Map metadata
    }

    class Location {
        +String repo
        +String filePath
        +int startLine
        +int endLine
        +String module
    }

    class Severity {
        <<enumeration>>
        CRITICAL
        HIGH
        MEDIUM
        LOW
        INFO
    }

    ScannerAdapter ..> ScanResult : produces
    ScanResult "1" --> "*" Finding
    Finding --> Location
    Finding --> Severity

    ScannerAdapter <|.. QodanaAdapter
    ScannerAdapter <|.. SnykAdapter
    ScannerAdapter <|.. SonarQubeAdapter
    ScannerAdapter <|.. ConventionAdapter
```

### Triage Engine

```mermaid
classDiagram
    class TriageEngine {
        +triage(report: ScanReport, activeJobs: Job[]) TriageResult
        -dedup(findings: Finding[]) Finding[]
        -score(finding: Finding) int
        -group(findings: Finding[]) Map
        -detectConflicts(findings: Finding[], activeJobs: Job[]) Conflict[]
    }

    class TriagedFinding {
        +Finding finding
        +int priorityScore
        +Severity adjustedSeverity
        +String groupKey
        +boolean autoFixCandidate
        +FixType fixType
    }

    class Conflict {
        +Finding finding
        +Job activeJob
        +String reason
    }

    class FixType {
        <<enumeration>>
        BUG_FIX
        DEPENDENCY_UPGRADE
        LIBRARY_MIGRATION
        CONVENTION_FIX
        SECURITY_PATCH
    }

    class TriageResult {
        +TriagedFinding[] findings
        +Conflict[] conflicts
        +TriageSummary summary
    }

    TriageEngine ..> TriageResult : produces
    TriageResult "1" --> "*" TriagedFinding
    TriageResult "1" --> "*" Conflict
    TriagedFinding --> FixType
```

### Spec-Kit Bundle Generation

The bundle generator produces one spec-kit maintenance artifact set per triaged finding:

```
maintenance-bundle/
├── metadata.json                      # Bundle summary, severity breakdown, conflict flags
├── findings/
│   ├── snyk-CVE-2026-1234/
│   │   ├── acceptance-criteria.md     # "Dependency X upgraded to >= 2.1.0, CVE resolved"
│   │   └── qa-tests.md               # "npm audit shows no findings for CVE-2026-1234"
│   ├── qodana-complexity-auth-middleware/
│   │   ├── acceptance-criteria.md     # "Cyclomatic complexity of handleAuth reduced to <= 10"
│   │   └── qa-tests.md               # "Qodana reports no complexity warning for handleAuth"
│   └── convention-naming-user-service/
│       ├── acceptance-criteria.md     # "All exported functions use camelCase"
│       └── qa-tests.md               # "Convention analyzer reports 0 naming violations"
└── summary.md                         # Human-readable overview for FeatureOutlook review
```

Each finding's acceptance criteria is **verifiable** — it describes a condition that can be mechanically checked after the fix. The qa-tests describe how to verify the fix was applied (often re-running the scanner that found it).

### SLA Configuration

```json
{
  "sla": {
    "critical": { "responseHours": 48, "escalateAfterHours": 24 },
    "high":     { "responseDays": 5 },
    "medium":   { "target": "next-sprint" },
    "low":      { "target": "backlog" }
  },
  "scanners": {
    "qodana":     { "enabled": true },
    "snyk":       { "enabled": true, "severityThreshold": "medium" },
    "sonarqube":  { "enabled": true, "qualityGate": "default" },
    "convention": { "enabled": true, "rulesPath": ".convention-rules.json" }
  },
  "scheduling": {
    "cadence": "weekly",
    "day": "monday",
    "hour": 2
  }
}
```

### Key Design Decisions

**Why a CLI (not a long-running service)?**
The Maintenance Agent has no state between runs. It scans, triages, bundles, registers, and exits. A long-running service would idle between scheduled runs, consuming resources for no benefit. A CLI is simpler to deploy, version, test, and debug. It's also composable — you can run `scan` and `triage` independently, pipe outputs, or invoke it from any scheduler.

**Why does it register with the Orchestrator (not dispatch directly to Phase 3)?**
The Maintenance Agent discovers issues — it doesn't decide what to fix. Humans must approve each finding before autonomous execution begins. By registering a maintenance job with the Orchestrator, the bundle enters the standard approval flow: it shows up in FeatureOutlook, the Tech Lead reviews per-finding, and only approved findings get dispatched to Phase 3. This preserves the governance model.

**Why conflict detection against active jobs?**
If a feature job is actively modifying `auth-service/src/middleware.ts` and a maintenance scan flags complexity in the same file, the maintenance fix could create a merge conflict or invalidate the feature work. The CLI queries the Orchestrator for active jobs and flags overlaps, letting the Tech Lead decide whether to defer the maintenance fix.

**Why spec-kit maintenance preset (not full artifact set)?**
Maintenance fixes don't need `ux.md`, `architecture.md`, or `feature.md` — they're scoped changes to existing code with known root causes (a CVE, a lint violation, a complexity spike). The maintenance preset includes only `acceptance-criteria.md` and `qa-tests.md`, which is sufficient for the agent to know what to fix and how to verify it. This reduces the approval burden on Tech Leads.

**Why one acceptance-criteria + qa-tests per finding (not per bundle)?**
Granular artifacts enable per-finding approval. A Tech Lead might approve the critical security patches immediately, defer the convention fixes to next sprint, and reject the low-priority style issues entirely. If the bundle had one monolithic acceptance-criteria, the only choice would be approve-all or reject-all.
