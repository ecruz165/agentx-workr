# Phase 3b — Integration & Preview · C4 Drill-Down

**Sub-phase:** Cross-repo PR collection → combined preview → E2E validation → merge to develop
**Actor:** Orchestrator (coordination) + [Harness](https://www.harness.io/) (CI/CD execution)
**Trigger:** All Phase 3a PRs ready (Orchestrator detects via PR refs collection)
**Output:** All PRs merged to `develop`, combined dev environment updated

[← Back to Phase 3 Overview](./README.md) · [← Phase 3a](./phase-3a-agent-execution.md)

---

## Overview

Phase 3b is the integration layer. While 3a produces per-repo PRs independently, 3b assembles them into a coherent whole — verifying that cross-repo changes work together before merging anything.

This is fundamentally an **Orchestrator-driven** phase. The Agentic Delivery Workspace produced the PRs; now the Orchestrator coordinates their integration, preview, and merge.

### Stages

| Stage | Name | Actor | Gate | Output |
|-------|------|-------|------|--------|
| 05 | Collect PRs | Orchestrator | Integration gate | PR manifest (all repos verified ready) |
| 06 | Preview Env | Harness | Integration gate | `preview-dev` — full-stack combined preview |
| 07 | Merge → develop | Orchestrator + CI | End of scope | All PRs merged, `develop` CI green |

### Why Phase 3b Exists

Per-repo PRs can individually pass all tests but fail together. Common cross-repo failure modes:

- **Contract mismatch:** API gateway expects `userId: string`, backend returns `userId: number`
- **Port collision:** Two services claim the same port in the preview environment
- **Dependency version conflict:** Shared library upgraded in one repo but not another
- **Feature flag inconsistency:** Frontend checks `enableNewDashboard`, backend checks `new_dashboard_enabled`

Phase 3b catches these by assembling all PRs into a combined preview and running cross-repo E2E tests.

---

## L3 — Component Diagram

### Integration Pipeline

```mermaid
graph TB
    subgraph p3a_output["Phase 3a Output"]
        PR1["PR: api-gateway\n─────────────\n· Branch: feat/XYZ-api\n· Tests: passing\n· Review: auto-approved"]
        PR2["PR: web-app\n─────────────\n· Branch: feat/XYZ-ui\n· Tests: passing\n· Review: auto-approved"]
        PR3["PR: shared-lib\n─────────────\n· Branch: feat/XYZ-lib\n· Tests: passing\n· Review: auto-approved"]
    end

    subgraph stage5["Stage 05 — Collect PRs"]
        COLLECTOR["PR Collector\n─────────────\n· Awaits all PRs from 3a\n· Verifies all repos ready\n· Checks no PR is blocked\n· Assembles PR manifest"]
        MANIFEST["PR Manifest\n─────────────\n· List of PR refs\n· Repo → branch mapping\n· Dependency order\n· Risk tier constraints"]
    end

    subgraph stage6["Stage 06 — Preview Environment"]
        HARNESS["Harness\n─────────────\n· Checks out all branches\n· Builds combined stack\n· Deploys preview env\n· Runs cross-repo E2E"]
        PREVIEW["Preview Environment\n─────────────\n· Full-stack deployment\n· Combined preview URL\n· All services running\n· Integration test suite"]
        E2E["Cross-Repo E2E\n─────────────\n· Contract validation\n· API integration tests\n· UI integration tests\n· Performance baseline"]
    end

    subgraph stage7["Stage 07 — Merge to develop"]
        MERGECOORD["Merge Coordinator\n─────────────\n· Dependency-ordered merge\n· Per-repo CI verification\n· Rollback on failure\n· Status reporting"]
        DEVENV["develop Environment\n─────────────\n· Combined dev CI passes\n· Dev environment updated\n· Feature complete in dev"]
    end

    PR1 & PR2 & PR3 --> COLLECTOR
    COLLECTOR --> MANIFEST
    MANIFEST --> HARNESS
    HARNESS --> PREVIEW
    PREVIEW --> E2E

    E2E -->|"Pass"| MERGECOORD
    E2E -->|"Fail"| GATE6{"Integration\nGate"}
    GATE6 -->|"Re-test"| HARNESS
    GATE6 -->|"Escalate"| ESC["→ Orchestrator\nescalation routing"]

    MERGECOORD --> DEVENV

    style PR1 fill:#181c24,stroke:#3b82f6,color:#60a5fa
    style PR2 fill:#181c24,stroke:#3b82f6,color:#60a5fa
    style PR3 fill:#181c24,stroke:#3b82f6,color:#60a5fa
    style COLLECTOR fill:#0e1610,stroke:#34d399,color:#4ade80
    style MANIFEST fill:#0e1610,stroke:#34d399,color:#4ade80
    style HARNESS fill:#0e1610,stroke:#34d399,color:#4ade80
    style PREVIEW fill:#0e1610,stroke:#34d399,color:#4ade80
    style E2E fill:#0e1610,stroke:#34d399,color:#4ade80
    style MERGECOORD fill:#0e1610,stroke:#34d399,color:#4ade80
    style DEVENV fill:#0e1610,stroke:#34d399,color:#4ade80
    style GATE6 fill:#1e1600,stroke:#f59e0b,color:#fbbf24
    style ESC fill:#241818,stroke:#f87171,color:#f87171
```

### PR Collection Strategy

```mermaid
sequenceDiagram
    participant WS as Agentic Delivery Workspace
    participant ORCH as Orchestrator
    participant GH as GitHub
    participant CI as Harness

    Note over WS: Phase 3a completes per-repo PRs

    loop For each repo in scope
        WS->>GH: Create PR (branch → develop)
        WS->>ORCH: Report PR ref (repo, PR number, branch)
        ORCH->>ORCH: Store PR ref in job record
    end

    ORCH->>ORCH: Check: all expected repos reported?

    alt All PRs collected
        ORCH->>ORCH: Assemble PR manifest
        ORCH->>ORCH: State: EXECUTING → INTEGRATING

        ORCH->>CI: Trigger preview deployment (PR manifest)
        CI->>GH: Checkout all feature branches
        CI->>CI: Build combined stack
        CI->>CI: Deploy preview environment
        CI->>CI: Run cross-repo E2E tests

        alt E2E passes
            CI-->>ORCH: Preview + E2E results: PASS
            ORCH->>ORCH: State: INTEGRATING → MERGING

            loop Dependency-ordered merge
                ORCH->>GH: Merge PR (repo N)
                GH-->>ORCH: Merge status
                ORCH->>GH: Verify develop CI (repo N)
            end

            ORCH->>ORCH: State: MERGING → COMPLETE
        else E2E fails
            CI-->>ORCH: Preview + E2E results: FAIL
            ORCH->>ORCH: Evaluate: re-test or escalate?
        end

    else Timeout / missing PR
        ORCH->>ORCH: Escalate: missing repo PR
    end
```

---

## L4 — Code Level

### PR Manifest

The PR manifest is the data structure that bridges Phase 3a and 3b. It captures everything the Harness needs to build a combined preview.

```mermaid
classDiagram
    class PRManifest {
        +String jobId
        +String featureId
        +RiskTier riskTier
        +List~PRRef~ pullRequests
        +List~RepoDependency~ mergeOrder
        +Instant assembledAt
        +isComplete() boolean
        +getMergeOrder() List~PRRef~
    }

    class PRRef {
        +String repo
        +int prNumber
        +String branch
        +String baseBranch
        +PRStatus status
        +String sha
        +List~String~ modifiedPaths
    }

    class RepoDependency {
        +String repo
        +List~String~ dependsOn
    }

    class PreviewResult {
        +String previewUrl
        +PreviewStatus status
        +List~E2ETestResult~ e2eResults
        +List~ContractCheck~ contractChecks
        +Instant deployedAt
        +Duration buildTime
    }

    class E2ETestResult {
        +String testSuite
        +String testName
        +TestStatus status
        +String errorMessage
        +List~String~ affectedRepos
    }

    class MergeResult {
        +String repo
        +int prNumber
        +MergeStatus status
        +String mergeSha
        +boolean developCIPassed
    }

    PRManifest "1" --> "*" PRRef
    PRManifest "1" --> "*" RepoDependency
    PRManifest ..> PreviewResult : triggers
    PreviewResult "1" --> "*" E2ETestResult
    PRManifest ..> MergeResult : produces per repo
```

### Merge Ordering

PRs must merge in dependency order. If `web-app` depends on `shared-lib`, the shared library PR must merge first so that `web-app`'s develop CI picks up the new version.

```mermaid
graph LR
    LIB["shared-lib\n(merge first)"] --> API["api-gateway\n(merge second)"]
    LIB --> WEB["web-app\n(merge third)"]
    API --> WEB

    style LIB fill:#0e1610,stroke:#34d399,color:#4ade80
    style API fill:#0e1610,stroke:#34d399,color:#4ade80
    style WEB fill:#0e1610,stroke:#34d399,color:#4ade80
```

The merge coordinator performs a topological sort on the repo dependency graph to determine order. If a merge fails (develop CI breaks), it stops and escalates rather than continuing with downstream repos.

### Integration Gate Decisions

| Failure Type | Gate Decision | Action |
|-------------|---------------|--------|
| Flaky test (known flaky suite) | Re-test | Retry E2E with same preview |
| Contract mismatch between repos | Escalate | Route to Phase 2 — API spec needs revision |
| Build failure in combined stack | Re-test (once) then Escalate | May be transient; if persistent, needs human investigation |
| Preview deployment timeout | Re-test | Infrastructure issue, retry |
| E2E failure with no safe fix | Escalate | Fundamental integration issue |
| Product ambiguity revealed late | Escalate | Route to Phase 2 — spec refinement needed |
| Merge conflict on develop | Escalate | Another change landed; needs human resolution |

### Risk Tier Effect on Phase 3b

| Aspect | Low Risk | Medium Risk | High Risk |
|--------|----------|-------------|-----------|
| Preview window | Standard (auto-merge after E2E pass) | Extended (24h preview before merge) | Manual merge approval required |
| E2E depth | Standard suite | Standard + security scans | Full suite + penetration test hooks |
| Merge authorization | Automated | Automated with extra reviewer | Architect + Tech Lead sign-off |
| Rollback plan | Automated revert PR | Automated with notification | Manual rollback with incident review |

### Key Design Decisions

**Why does the Orchestrator (not the Workspace) coordinate Phase 3b?**
The Workspace is ephemeral — it exists for the duration of a job. Phase 3b may span multiple hours (preview deployment, E2E suite, human review for high-risk). The Orchestrator is the persistent system that can wait for CI callbacks, handle timeouts, and coordinate merge ordering across repos. The Workspace's job is done once PRs are created.

**Why dependency-ordered merge (not simultaneous)?**
Simultaneous merge of multiple PRs to `develop` creates race conditions in CI. If `shared-lib` and `web-app` merge at the same time, `web-app`'s CI might run against the old `shared-lib` version. Sequential, dependency-ordered merge ensures each repo's CI runs against the correct versions of its dependencies.

**Why preview environment (not just CI tests)?**
CI tests verify functional correctness. Preview environments verify integration — can the services actually talk to each other? Do the ports line up? Does the UI render correctly against the new API responses? For cross-repo features, the combined preview catches classes of bugs that per-repo CI cannot.

**Why is scope bounded at merge to develop?**
Release to production involves different governance: change control boards, production readiness reviews, canary deployments, rollback procedures. These are organizational processes that vary widely between teams. By stopping at `develop`, the agentic delivery model has a clean handoff point and doesn't need to model every organization's release process.
