# Phase 2 — Tech Design · C4 Drill-Down

**Phase:** Human-led, agent-assisted
**Owner:** Tech Lead / Architect + Specialists (UX, QA)
**Trigger:** Curated `feature.md` submitted from Phase 1
**Output:** Approved `plan.md` (execution-safe transition artifact)

[← Back to System Overview](../README.md)

---

## Overview

Phase 2 transforms a curated feature description into a set of execution-safe artifacts that autonomous agents can act on without ambiguity. This is where **meaning becomes structure** — human experts decompose intent into testable criteria, architectural boundaries, and scoped tasks.

The phase is built on three systems:
- **github/spec-kit** — the open-source foundation providing spec-driven templates, an extensions model, the `specify` CLI, and a devcontainer base
- **FeatureOutlook** — an internal VS Code extension + TUI that extends spec-kit with review workflows, NFR checklists, gate approvals, and queue management
- **Execution engine (drafting mode)** — the same spec-kit + LangChain + LangSmith engine that implements plans in Phase 3 also operates in Phase 2 to **draft artifacts** from `feature.md`. It generates suggested acceptance criteria, architectural patterns, and test strategies that humans then review and refine. See [Delivery Workspace → Execution Engine](../components/delivery-workspace/README.md#execution-engine-spec-kit--langchain--langsmith) for the dual-mode architecture.

### Why This Phase Exists

Autonomous agents are excellent executors but poor arbiters of product intent. Phase 2 exists to eliminate the class of failures where an agent builds the wrong thing correctly. By requiring human sign-off on testable acceptance criteria, architectural boundaries, and quality checklists before any code is generated, the system trades upfront human time for dramatically reduced rework in Phase 3.

### Phase 2 Stages

| Stage | Name | Owner | Output |
|-------|------|-------|--------|
| 04a | Tech Design — Core Artifacts | Tech Lead, Architect | `feature.md` (curated), `acceptance-criteria.md`, `architecture.md` |
| 04b | Specialists | UX Designer, QA Engineer | `ux.md`, `qa-tests.md` |
| 05 | Spec Review | Tech Lead, Architect | NFR-reviewed, approved artifact bundle |
| 06 | Plan Generation | Tech Lead, Architect | `plan.md` (transition artifact — triggers Phase 3) |

---

## L3 — Component Diagram

### FeatureOutlook Internals

```mermaid
graph TB
    subgraph fo["FeatureOutlook · VS Code Extension + TUI"]
        direction TB

        subgraph ui["Presentation Layer"]
            QUEUE["Queue View\n─────────────\n· Submitted featuresets\n· Self-assign from queue\n· Filter by team / priority\n· Status indicators"]
            EDITOR["Artifact Editor\n─────────────\n· Side-by-side feature.md\n  + spec-kit templates\n· Inline agent suggestions\n· Gap detection highlights"]
            REVIEW["Review Panel\n─────────────\n· NFR checklist renderer\n· Approval controls\n· Risk tier display\n· Gate sign-off buttons"]
            TUI_VIEW["TUI Interface\n─────────────\n· Terminal-based queue\n· Keyboard-driven review\n· Approval workflow\n· Same API, no VS Code"]
        end

        subgraph core["Core Logic"]
            ARTMAN["Artifact Manager\n─────────────\n· CRUD on spec-kit artifacts\n· Template instantiation\n· Extension resolution\n· Validation rules"]
            NFRENG["NFR Engine\n─────────────\n· Security checklist\n· Observability checklist\n· Performance checklist\n· Accessibility checklist\n· Data/Privacy checklist"]
            GATELOGIC["Gate Logic\n─────────────\n· Completeness check\n· All NFRs reviewed?\n· Required approvers met?\n· Risk tier constraints"]
            EXTMODEL["Extension Model\n─────────────\n· Conditional artifacts\n  by feature type\n· api-change → integration.md\n· ui-component → accessibility.md\n· data-migration → migration.md"]
        end

        subgraph integration["Integration Layer"]
            ORCHCLIENT["Orchestrator Client\n─────────────\n· REST API calls\n· Job registration\n· Gate decision push\n· Status polling"]
            SPECKIT["spec-kit CLI Bridge\n─────────────\n· specify init\n· specify validate\n· Template rendering\n· Constitution loading"]
            GITINT["Git Integration\n─────────────\n· Artifact branch mgmt\n· PR creation for plan.md\n· Merge triggers dispatch"]
        end
    end

    QUEUE --> ARTMAN
    EDITOR --> ARTMAN
    REVIEW --> NFRENG
    REVIEW --> GATELOGIC
    ARTMAN --> EXTMODEL
    ARTMAN --> SPECKIT
    GATELOGIC --> ORCHCLIENT
    ORCHCLIENT --> ORCH["Orchestrator API"]
    GITINT --> GH["GitHub"]
    TUI_VIEW --> ARTMAN
    TUI_VIEW --> GATELOGIC

    style QUEUE fill:#201c28,stroke:#a78bfa,color:#d4d6de
    style EDITOR fill:#201c28,stroke:#a78bfa,color:#d4d6de
    style REVIEW fill:#201c28,stroke:#a78bfa,color:#d4d6de
    style TUI_VIEW fill:#201c28,stroke:#a78bfa,color:#d4d6de
    style ARTMAN fill:#13101e,stroke:#c084fc,color:#d4d6de
    style NFRENG fill:#13101e,stroke:#c084fc,color:#d4d6de
    style GATELOGIC fill:#13101e,stroke:#c084fc,color:#d4d6de
    style EXTMODEL fill:#13101e,stroke:#c084fc,color:#d4d6de
    style ORCHCLIENT fill:#13101e,stroke:#5b21b6,color:#d4d6de
    style SPECKIT fill:#13101e,stroke:#5b21b6,color:#d4d6de
    style GITINT fill:#13101e,stroke:#5b21b6,color:#d4d6de
    style ORCH fill:#0e1420,stroke:#22d3ee,color:#d4d6de
    style GH fill:#111115,stroke:#888780,color:#d4d6de
```

### Artifact Flow Through Phase 2

```mermaid
graph LR
    FM["curated\nfeature.md"] --> INIT["spec-kit init\n(template instantiation)"]
    INIT --> CORE["Core Artifacts\n· acceptance-criteria.md\n· architecture.md"]
    INIT --> SPEC["Specialist Artifacts\n· ux.md\n· qa-tests.md"]
    INIT --> COND["Conditional Artifacts\n(by feature type)"]

    CORE --> CONVERGE["Artifact Bundle"]
    SPEC --> CONVERGE
    COND --> CONVERGE

    CONVERGE --> NFR["NFR Review\n· Security\n· Observability\n· Performance\n· Accessibility\n· Data/Privacy"]

    NFR --> GATE{"Gate:\nAll NFRs\nreviewed?"}
    GATE -->|"Yes"| PLAN["plan.md\n(transition artifact)"]
    GATE -->|"No"| NFR

    PLAN --> APPROVE{"Gate:\nArchitect\napproves?"}
    APPROVE -->|"Approved"| DISPATCH["PR merge →\nOrchestrator dispatch"]
    APPROVE -->|"Rejected"| CORE

    style FM fill:#2a2520,stroke:#f59e0b,color:#fde047
    style PLAN fill:#201c28,stroke:#a78bfa,color:#c084fc
    style DISPATCH fill:#0e1420,stroke:#22d3ee,color:#d4d6de
```

---

## L4 — Code Level

### FeatureOutlook Extension Model

The extension model determines which artifacts are required for a given feature. Core artifacts are always required. Conditional artifacts are activated based on feature type tags in the `feature.md` frontmatter.

```mermaid
classDiagram
    class ArtifactManifest {
        +String featureId
        +String featureType
        +List~ArtifactSpec~ coreArtifacts
        +List~ArtifactSpec~ conditionalArtifacts
        +List~ArtifactSpec~ extensionArtifacts
        +resolveRequired() List~ArtifactSpec~
        +isComplete() boolean
    }

    class ArtifactSpec {
        +String name
        +String templatePath
        +String owner
        +ArtifactStatus status
        +boolean required
        +validate() ValidationResult
    }

    class ExtensionRule {
        +String featureType
        +List~String~ requiredArtifacts
        +matches(feature: Feature) boolean
    }

    class NFRChecklist {
        +String artifactId
        +List~NFRItem~ items
        +String reviewedBy
        +Instant reviewedAt
        +boolean allPassing()
    }

    class NFRItem {
        +String category
        +String criterion
        +NFRStatus status
        +String notes
    }

    class GateDecision {
        +String jobId
        +String gateId
        +GateOutcome outcome
        +String decidedBy
        +Instant decidedAt
        +String evidence
    }

    ArtifactManifest "1" --> "*" ArtifactSpec
    ArtifactManifest "1" --> "*" ExtensionRule
    ArtifactSpec "1" --> "0..1" NFRChecklist
    NFRChecklist "1" --> "*" NFRItem
    ArtifactManifest ..> GateDecision : produces
```

### Extension Rules — Conditional Artifacts by Feature Type

| Feature Type | Additional Required Artifacts |
|-------------|------------------------------|
| `api-change` | `integration.md`, `security.md` |
| `ui-component` | `ux.md` (already core), `accessibility.md` |
| `data-migration` | `migration.md`, `performance.md` |
| `infra-change` | `observability.md`, `security.md` |

Extension candidates that any organization can add: `ops.md`, `change-control.md`, plus org-specific artifacts.

### Gate Approval Sequence

```mermaid
sequenceDiagram
    participant TL as Tech Lead
    participant FO as FeatureOutlook
    participant SK as spec-kit CLI
    participant ORCH as Orchestrator

    TL->>FO: Open feature from queue
    FO->>SK: specify validate (artifact bundle)
    SK-->>FO: Validation result

    alt Validation fails
        FO-->>TL: Show gaps / incomplete artifacts
        TL->>FO: Edit artifacts
        FO->>SK: specify validate (re-check)
    end

    TL->>FO: Complete NFR checklist
    FO->>FO: Gate Logic — all NFRs reviewed?

    TL->>FO: Approve plan.md
    FO->>FO: Gate Logic — required approvers met?

    FO->>ORCH: POST /jobs/{id}/gates/plan-approval
    Note over FO,ORCH: Decision: approved / rejected<br/>Evidence: NFR checklist + approver identity

    ORCH->>ORCH: Record gate decision in audit trail
    ORCH->>ORCH: Transition job state: DESIGNING → APPROVED

    alt Feature branch PR merged
        FO->>ORCH: Webhook: plan.md PR merged
        ORCH->>ORCH: Transition job state: APPROVED → DISPATCHING
    end
```

### Key Design Decisions

**Why VS Code Extension AND TUI?**
Not all reviewers use VS Code. The TUI provides the same artifact review and approval workflow in a terminal, backed by the same Orchestrator API. Both UIs are thin clients — the gate logic and artifact validation live in shared core modules.

**Why spec-kit as foundation?**
spec-kit provides an open-source standard for spec-driven development: templates, extensions model, CLI, and devcontainer base. FeatureOutlook extends it rather than replacing it, meaning teams that outgrow FeatureOutlook still have a standards-based artifact format. The `specify` CLI can validate artifacts in CI without FeatureOutlook installed.

**Why NFR review before plan.md generation?**
Plan.md is the transition artifact that triggers autonomous execution. If NFRs (security, observability, performance, accessibility, data/privacy) aren't reviewed before plan.md is approved, the agents may generate code that passes functional tests but violates non-functional requirements. Catching this in Phase 2 is dramatically cheaper than catching it in Phase 3 (where the fix requires re-specification, not just re-generation).

**Why is the risk tier overridable by Tech Lead?**
The Orchestrator's automated risk scoring uses heuristics (repo criticality, data surface, etc.) that can't capture organizational context. A Tech Lead might know that a seemingly low-risk change to an internal tool actually touches a shared library used by payments. The override is logged in the audit trail as a human judgment call.
