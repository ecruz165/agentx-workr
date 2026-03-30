# Phase M — Maintenance · C4 Drill-Down

**Phase:** Agent-driven, HITL-gated
**Owner:** Maintenance Agent (automated) · Tech Lead (approval gate)
**Trigger:** Cron schedule (weekly cadence)
**Output:** Approved `maintenance-plan.md` (transition artifact → dispatched to Phase 3)

[← Back to System Overview](../README.md) · [Maintenance Agent component](../components/agent-maintenance/README.md)

---

## Overview

Phase M runs in parallel with the feature delivery pipeline (Phases 2→3). It is the system's **immune system** — continuously scanning the codebase for code health issues, security vulnerabilities, dependency risks, and convention violations, then packaging approved fixes for autonomous execution.

The key governance constraint: the Maintenance Agent can **discover and propose** freely, but it cannot **execute** without human approval. The HITL gate at Stage M3 ensures that maintenance fixes don't conflict with in-progress feature work and that automated triage aligns with human judgment about priority and scope.

### Phase M Stages

| Stage | Name | Actor | Output |
|-------|------|-------|--------|
| M1 | Scheduled Scan | Agent (cron) | `scan-report.json` |
| M2 | Triage & Prioritize | Agent | spec-kit bundle (maintenance preset) |
| M3 | Maintenance Approval Gate | Human (Tech Lead) | Approved / rejected individual fixes |
| M4 | Plan Generation | Agent | `maintenance-plan.md` (transition artifact) |

### SLA by Severity

| Severity | Response SLA | Routing |
|----------|-------------|---------|
| Critical | 48 hours | Immediate escalation to Tech Lead |
| High | 5 business days | Next available maintenance window |
| Medium | Next sprint | Queued for batch processing |
| Low | Backlog | Tracked, no SLA |

---

## L3 — Component Diagram

### Maintenance Pipeline

```mermaid
graph TB
    subgraph trigger["Trigger"]
        CRON["Cron Scheduler\n─────────────\n· Weekly cadence\n· Per-repo or fleet-wide\n· Configurable schedule"]
    end

    subgraph m1["Stage M1 — Scheduled Scan"]
        SCANORCH["Scan Orchestrator\n─────────────\n· Kicks off all scanners\n· Collects unified report\n· Normalizes findings"]

        subgraph scanners["Scanner Adapters"]
            QOD["Qodana Adapter\n· Code quality\n· Code smells\n· Complexity"]
            SNYK["Snyk Adapter\n· Dependency vulns\n· License risk\n· Container scan"]
            SONAR["SonarQube Adapter\n· Static analysis\n· Coverage gaps\n· Duplications"]
            CONV["Convention Analyzer\n· Custom rules\n· Naming conventions\n· Architectural fit"]
        end
    end

    subgraph m2["Stage M2 — Triage & Prioritize"]
        TRIAGE["Triage Engine\n─────────────\n· Severity scoring\n· Dedup across scanners\n· Group by repo/module\n· Rank by impact+complexity"]
        SPECGEN["Spec-Kit Generator\n─────────────\n· maintenance preset\n· acceptance-criteria.md\n· qa-tests.md per fix\n· No ux.md (maintenance)"]
    end

    subgraph m3["Stage M3 — Approval Gate"]
        FOREV["FeatureOutlook Review\n─────────────\n· View maintenance bundle\n· Approve/reject per fix\n· Set scope boundaries\n· Check feature conflicts"]
    end

    subgraph m4["Stage M4 — Plan Generation"]
        MPLAN["maintenance-plan.md\n─────────────\n· Bug fix specs per issue\n· Dependency upgrade paths\n· Library migration steps\n· Convention violation fixes"]
    end

    CRON --> SCANORCH
    SCANORCH --> QOD & SNYK & SONAR & CONV
    QOD & SNYK & SONAR & CONV --> TRIAGE
    TRIAGE --> SPECGEN
    SPECGEN --> FOREV
    FOREV -->|"approved"| MPLAN
    FOREV -->|"rejected"| TRIAGE

    MPLAN -->|"dispatches to\nPhase 3 via\nOrchestrator"| ORCH["Orchestrator"]

    style CRON fill:#0e1420,stroke:#38bdf8,color:#d4d6de
    style SCANORCH fill:#0e1420,stroke:#38bdf8,color:#d4d6de
    style QOD fill:#111115,stroke:#f97316,color:#f97316
    style SNYK fill:#111115,stroke:#4ade80,color:#4ade80
    style SONAR fill:#111115,stroke:#38bdf8,color:#7dd3fc
    style CONV fill:#111115,stroke:#fbbf24,color:#fbbf24
    style TRIAGE fill:#0e1420,stroke:#38bdf8,color:#d4d6de
    style SPECGEN fill:#0e1420,stroke:#38bdf8,color:#d4d6de
    style FOREV fill:#201c28,stroke:#a78bfa,color:#c084fc
    style MPLAN fill:#0e1420,stroke:#38bdf8,color:#7dd3fc
    style ORCH fill:#0e1420,stroke:#22d3ee,color:#d4d6de
```

### Scanner Integration Architecture

```mermaid
graph LR
    subgraph adapters["Scanner Adapter Layer"]
        direction TB
        IFACE["ScannerAdapter\n(interface)"]
        QOD["QodanaAdapter"]
        SNYK["SnykAdapter"]
        SONAR["SonarQubeAdapter"]
        CONV["ConventionAdapter"]
    end

    subgraph normalization["Normalization"]
        NORM["Finding Normalizer\n─────────────\n· Common severity scale\n· Unified location format\n· Scanner-agnostic ID\n· Dedup fingerprint"]
    end

    subgraph output["Unified Output"]
        REPORT["scan-report.json\n─────────────\n· Findings[]\n· Scanner metadata\n· Repo scope\n· Timestamp"]
    end

    IFACE --> QOD & SNYK & SONAR & CONV
    QOD & SNYK & SONAR & CONV --> NORM
    NORM --> REPORT

    style IFACE fill:#0e1420,stroke:#38bdf8,color:#d4d6de
    style NORM fill:#0e1420,stroke:#38bdf8,color:#d4d6de
    style REPORT fill:#0e1420,stroke:#38bdf8,color:#7dd3fc
```

---

## L4 — Code Level

### Scanner Adapter Interface

All scanners implement the same interface. This allows adding new scanners (e.g., Trivy, Semgrep) without changing the orchestration logic.

```mermaid
classDiagram
    class ScannerAdapter {
        <<interface>>
        +name() String
        +scan(repos: List~RepoRef~) ScanResult
        +isAvailable() boolean
    }

    class ScanResult {
        +String scannerName
        +Instant timestamp
        +List~Finding~ findings
        +ScanStatus status
    }

    class Finding {
        +String id
        +String scanner
        +Severity severity
        +String title
        +String description
        +Location location
        +String fingerprint
        +Map~String,String~ metadata
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

    class QodanaAdapter {
        +scan(repos) ScanResult
        +parseQodanaReport(json) List~Finding~
    }
    class SnykAdapter {
        +scan(repos) ScanResult
        +parseSnykOutput(json) List~Finding~
    }
    class SonarQubeAdapter {
        +scan(repos) ScanResult
        +querySonarAPI(projectKey) List~Finding~
    }
    class ConventionAdapter {
        +scan(repos) ScanResult
        +loadRules(configPath) List~ConventionRule~
    }

    ScannerAdapter <|.. QodanaAdapter
    ScannerAdapter <|.. SnykAdapter
    ScannerAdapter <|.. SonarQubeAdapter
    ScannerAdapter <|.. ConventionAdapter
```

### Triage Engine

The triage engine deduplicates findings across scanners, groups them by repo/module, and produces a ranked list of actionable fixes.

```mermaid
classDiagram
    class TriageEngine {
        +triage(report: ScanReport) TriageResult
        -dedup(findings: List~Finding~) List~Finding~
        -score(finding: Finding) int
        -group(findings: List~Finding~) Map~String, List~Finding~~
        -detectConflicts(findings: List~Finding~, activeJobs: List~Job~) List~Conflict~
    }

    class TriageResult {
        +List~TriagedFinding~ findings
        +List~Conflict~ conflicts
        +Map~String, List~TriagedFinding~~ byRepo
        +TriageSummary summary
    }

    class TriagedFinding {
        +Finding finding
        +int priorityScore
        +String suggestedFix
        +Severity adjustedSeverity
        +String groupKey
        +boolean autoFixCandidate
    }

    class Conflict {
        +Finding finding
        +Job activeJob
        +String reason
        +ConflictResolution suggestedResolution
    }

    class MaintenanceBundle {
        +String bundleId
        +List~MaintenanceSpec~ specs
        +Instant generatedAt
        +TriageSummary summary
        +toSpecKit() SpecKitBundle
    }

    class MaintenanceSpec {
        +TriagedFinding finding
        +String acceptanceCriteria
        +String qaTests
        +FixType fixType
    }

    class FixType {
        <<enumeration>>
        BUG_FIX
        DEPENDENCY_UPGRADE
        LIBRARY_MIGRATION
        CONVENTION_FIX
        SECURITY_PATCH
    }

    TriageEngine ..> TriageResult : produces
    TriageResult "1" --> "*" TriagedFinding
    TriageResult "1" --> "*" Conflict
    TriagedFinding --> Finding
    MaintenanceBundle "1" --> "*" MaintenanceSpec
    MaintenanceSpec --> TriagedFinding
    MaintenanceSpec --> FixType
```

### Maintenance Approval Flow

```mermaid
sequenceDiagram
    participant CRON as Cron Scheduler
    participant SCAN as Scan Orchestrator
    participant TRIAGE as Triage Engine
    participant ORCH as Orchestrator
    participant FO as FeatureOutlook
    participant TL as Tech Lead

    CRON->>SCAN: Trigger weekly scan
    SCAN->>SCAN: Run all scanner adapters
    SCAN-->>TRIAGE: scan-report.json

    TRIAGE->>ORCH: Query active jobs (conflict check)
    ORCH-->>TRIAGE: Active job list
    TRIAGE->>TRIAGE: Dedup + score + group + conflict detect

    TRIAGE-->>ORCH: Register maintenance bundle as job
    ORCH->>ORCH: Assign risk tier to bundle
    ORCH->>ORCH: State: SCAN_COMPLETE → AWAITING_APPROVAL

    ORCH->>FO: Notify: maintenance bundle ready for review
    FO->>TL: Show bundle in queue

    TL->>FO: Review individual fixes
    TL->>FO: Approve / reject each fix
    TL->>FO: Set scope boundaries
    TL->>FO: Confirm no feature conflicts

    FO->>ORCH: POST /jobs/{id}/gates/maintenance-approval
    Note over FO,ORCH: Per-fix decisions + scope constraints

    ORCH->>ORCH: State: AWAITING_APPROVAL → APPROVED
    ORCH->>ORCH: Generate maintenance-plan.md from approved fixes
    ORCH->>ORCH: State: APPROVED → DISPATCHING
    ORCH-->>ORCH: Dispatch to Phase 3
```

### Key Design Decisions

**Why spec-kit maintenance preset (not full artifact set)?**
Maintenance fixes don't need `ux.md` or `architecture.md` — they're scoped changes to existing code. The maintenance preset includes only `acceptance-criteria.md` and `qa-tests.md`, which are sufficient for the agent to know what to fix and how to verify the fix. This reduces the approval burden on Tech Leads.

**Why dedup across scanners?**
Multiple scanners often flag the same issue. Qodana and SonarQube may both flag code complexity; Snyk and the Convention Analyzer may both flag an outdated dependency. Without dedup, the maintenance bundle would contain duplicate work orders, wasting agent execution time and human review time.

**Why conflict detection against active jobs?**
If a feature job is actively modifying `auth-service/src/middleware.ts` and a maintenance scan flags a security issue in the same file, the maintenance fix could create a merge conflict or invalidate the feature work. The triage engine detects these overlaps and flags them as conflicts, letting the Tech Lead decide whether to defer the maintenance fix or coordinate with the feature team.

**Why per-fix approval (not bulk)?**
Bulk approval is faster but riskier. A maintenance bundle might contain 15 findings: 3 critical security patches, 5 medium convention fixes, and 7 low-priority style issues. The Tech Lead should be able to approve the critical patches immediately, defer the convention fixes to the next sprint, and reject the style issues entirely. Per-fix granularity respects this judgment.

### Maintenance → Phase 3 Convergence

The maintenance pipeline's output (`maintenance-plan.md`) is structurally identical to Phase 2's output (`plan.md`) from the Orchestrator's perspective. Both are spec-kit validated artifact bundles that the Orchestrator dispatches to Phase 3 through the same state machine transitions. The only differences:

| Aspect | Feature (Phase 2) | Maintenance (Phase M) |
|--------|-------------------|----------------------|
| Trigger | Human submits curated feature.md | Cron-triggered scan |
| Artifact set | Full (core + conditional + extensions) | Maintenance preset (acceptance-criteria + qa-tests) |
| Approval depth | Full NFR review | Per-fix approval with scope boundaries |
| Risk tier | Full scoring (repo criticality, customer impact, etc.) | Typically Low/Medium unless security-critical |
| Phase 3 execution | Same pipeline | Same pipeline |
