# Orchestrator · C4 Drill-Down

**System:** Central nervous system of the Governed Agentic Delivery Platform
**Technology:** Spring Boot 3.x · Spring Modulith · Spring State Machine
**Lifecycle:** Always-running service (not ephemeral)
**Role:** Job lifecycle management, state transitions, dispatch, escalation routing, audit trail

[← Back to System Overview](../../README.md)

---

## Overview

The Orchestrator is the only system that has visibility across all phases. It does not generate code, review artifacts, or execute tests — it **manages the lifecycle of jobs** as they flow from curated feature submission through merge to `develop`.

Every phase transition, every gate decision, every escalation, and every retry is mediated by the Orchestrator. This makes it the single source of truth for "what is the current state of this feature?" and the complete audit trail for "how did it get there?"

### Core Responsibilities

| Responsibility | Description |
|---------------|-------------|
| **Job registration** | Receives curated `feature.md` from Phase 1, creates job record, assigns initial risk tier |
| **Queue management** | Maintains queue of jobs awaiting tech design review in FeatureOutlook |
| **Gate enforcement** | Records gate decisions from FeatureOutlook, validates preconditions before state transitions |
| **Dispatch** | Sends approved artifact bundles to the Agentic Delivery Workspace for Phase 3 execution |
| **Execution monitoring** | Tracks Phase 3 progress, retry depth, escalation events |
| **PR collection** | Collects PR refs from Phase 3a, triggers Phase 3b when all PRs are ready |
| **Merge coordination** | Orchestrates dependency-ordered merge in Phase 3b |
| **Escalation routing** | Routes escalations to Phase 2, Phase M, or HITL inline based on escalation type |
| **Audit trail** | Append-only log of every state transition, gate decision, and actor identity |

### Why Spring Modulith?

The Orchestrator's responsibilities are logically distinct but data-coupled. All modules operate on the same core aggregates (`Job`, `Gate`, `Escalation`). Spring Modulith gives:

- **Module boundaries** with explicit inter-module APIs (events, not direct calls)
- **Single deployable** — one JAR, one database, one transaction scope
- **Module verification** — compile-time checks that modules only communicate through declared interfaces
- **Event publication** — modules react to events rather than calling each other directly

This avoids the operational overhead of microservices (separate deployments, distributed transactions, service discovery) while maintaining clean separation of concerns.

### Why Spring State Machine?

The job lifecycle has well-defined states, transitions, and guards. Spring State Machine provides:

- **Declarative state graph** — states and transitions defined in config, not scattered through business logic
- **Guards** — precondition checks before transitions (e.g., "all NFRs must be reviewed before APPROVED")
- **Actions** — side effects on transitions (e.g., "dispatch to Workspace on APPROVED → DISPATCHING")
- **Persistence** — state machine state persisted to PostgreSQL, survives restarts
- **Audit events** — every transition emits an event with actor, timestamp, and context

---

## L3 — Component Diagram

### Modulith Module Map

```mermaid
graph TB
    subgraph orchestrator["Orchestrator · Spring Modulith"]
        direction TB

        subgraph job_module["Job Module"]
            JOBREG["Job Registry\n─────────────\n· Create job from feature.md\n· Create job from maintenance bundle\n· Assign risk tier\n· Store artifact refs"]
            JOBSTATE["Job State Machine\n─────────────\n· Spring State Machine\n· State transitions\n· Guard evaluation\n· Transition actions"]
            JOBQUERY["Job Query Service\n─────────────\n· Current state\n· History\n· By phase / status / team"]
        end

        subgraph gate_module["Gate Module"]
            GATEREG["Gate Registry\n─────────────\n· Define gate per phase\n· Required approvers\n· Risk-tier constraints"]
            GATEDECISION["Gate Decision Handler\n─────────────\n· Record approval/rejection\n· Validate approver authority\n· Check preconditions"]
            GATEQUERY["Gate Query Service\n─────────────\n· Pending gates\n· Decision history\n· Approver audit"]
        end

        subgraph dispatch_module["Dispatch Module"]
            DISPATCHER["Job Dispatcher\n─────────────\n· Package artifact bundle\n· Provision Workspace\n· Send to DevContainer\n· Track dispatch status"]
            WSMONITOR["Workspace Monitor\n─────────────\n· Heartbeat tracking\n· Timeout detection\n· Retry depth monitoring"]
        end

        subgraph integration_module["Integration Module"]
            PRCOLLECT["PR Collector\n─────────────\n· Receive PR refs from 3a\n· Track per-repo readiness\n· Assemble PR manifest"]
            PREVIEWTRIG["Preview Trigger\n─────────────\n· Detect all PRs ready\n· Trigger Harness\n· Track preview status"]
            MERGECOORD["Merge Coordinator\n─────────────\n· Topological sort repos\n· Dependency-ordered merge\n· Per-repo CI verification\n· Rollback on failure"]
        end

        subgraph escalation_module["Escalation Module"]
            ESCROUTER["Escalation Router\n─────────────\n· Classify escalation type\n· Route to Phase 2 / M / HITL\n· Track resolution status"]
            ESCNOTIFY["Notification Service\n─────────────\n· Alert Tech Lead / Dev\n· Channel routing (Slack, email)\n· SLA tracking (4h target)"]
        end

        subgraph queue_module["Queue Module"]
            FEATUREQ["Feature Queue\n─────────────\n· Jobs awaiting tech design\n· Priority ordering\n· Team assignment\n· Self-assign support"]
            MAINTQ["Maintenance Queue\n─────────────\n· Maintenance bundles\n  awaiting approval\n· Severity ordering"]
        end

        subgraph audit_module["Audit Module"]
            AUDITLOG["Audit Log\n─────────────\n· Append-only event store\n· Actor identity\n· Timestamp\n· Evidence / context"]
            METRICS["Operating Metrics\n─────────────\n· Autonomous completion rate\n· Escalation rate per phase\n· Defect escape rate\n· Cycle time\n· Rework rate"]
        end
    end

    %% Inter-module event flow
    JOBREG -->|"JobCreated event"| FEATUREQ
    JOBREG -->|"MaintenanceJobCreated"| MAINTQ
    GATEDECISION -->|"GateApproved event"| JOBSTATE
    JOBSTATE -->|"StateTransition event"| AUDITLOG
    JOBSTATE -->|"APPROVED→DISPATCHING"| DISPATCHER
    DISPATCHER -->|"JobDispatched event"| WSMONITOR
    WSMONITOR -->|"PRReported event"| PRCOLLECT
    PRCOLLECT -->|"AllPRsReady event"| PREVIEWTRIG
    PREVIEWTRIG -->|"PreviewPassed event"| MERGECOORD
    MERGECOORD -->|"AllMerged event"| JOBSTATE
    WSMONITOR -->|"EscalationRaised event"| ESCROUTER
    ESCROUTER -->|"EscalationRouted event"| AUDITLOG
    GATEDECISION -->|"GateDecision event"| AUDITLOG

    style JOBREG fill:#0e1420,stroke:#22d3ee,color:#d4d6de
    style JOBSTATE fill:#0e1420,stroke:#22d3ee,color:#d4d6de
    style JOBQUERY fill:#0e1420,stroke:#22d3ee,color:#d4d6de
    style GATEREG fill:#201c28,stroke:#a78bfa,color:#d4d6de
    style GATEDECISION fill:#201c28,stroke:#a78bfa,color:#d4d6de
    style GATEQUERY fill:#201c28,stroke:#a78bfa,color:#d4d6de
    style DISPATCHER fill:#0e1220,stroke:#3b82f6,color:#d4d6de
    style WSMONITOR fill:#0e1220,stroke:#3b82f6,color:#d4d6de
    style PRCOLLECT fill:#0e1610,stroke:#34d399,color:#d4d6de
    style PREVIEWTRIG fill:#0e1610,stroke:#34d399,color:#d4d6de
    style MERGECOORD fill:#0e1610,stroke:#34d399,color:#d4d6de
    style ESCROUTER fill:#1e1600,stroke:#f59e0b,color:#d4d6de
    style ESCNOTIFY fill:#1e1600,stroke:#f59e0b,color:#d4d6de
    style FEATUREQ fill:#16140e,stroke:#f59e0b,color:#d4d6de
    style MAINTQ fill:#0e1420,stroke:#38bdf8,color:#d4d6de
    style AUDITLOG fill:#111115,stroke:#888780,color:#d4d6de
    style METRICS fill:#111115,stroke:#888780,color:#d4d6de
```

### External Integration Points

```mermaid
graph LR
    subgraph orchestrator["Orchestrator"]
        REST["REST API\n· FeatureOutlook CRUD\n· Job queries\n· Gate decisions"]
        WS["WebSocket\n· Real-time job status\n· Execution progress"]
        WEBHOOK["Webhook Handler\n· GitHub PR events\n· CI status callbacks\n· Workspace heartbeats"]
        EVENT["Event Publisher\n· Dispatch events\n· Escalation alerts"]
    end

    FO["FeatureOutlook\n(VS Code + TUI)"] <-->|"REST"| REST
    FO <-->|"WebSocket"| WS

    GH["GitHub"] -->|"Webhooks"| WEBHOOK
    CI["Harness"] -->|"Callbacks"| WEBHOOK

    ADW["Agentic Delivery\nWorkspace"] -->|"Callbacks"| WEBHOOK
    EVENT -->|"Dispatch"| ADW
    EVENT -->|"Notifications"| NOTIFY["Slack / Email"]

    CTX["Context Server\n(Neo4j)"] <-->|"Cypher"| orchestrator

    style REST fill:#0e1420,stroke:#22d3ee,color:#d4d6de
    style WS fill:#0e1420,stroke:#22d3ee,color:#d4d6de
    style WEBHOOK fill:#0e1420,stroke:#22d3ee,color:#d4d6de
    style EVENT fill:#0e1420,stroke:#22d3ee,color:#d4d6de
```

---

## L4 — Code Level

### Job State Machine

The job lifecycle is the heart of the Orchestrator. Every job follows this state graph, with transitions guarded by preconditions.

```mermaid
stateDiagram-v2
    [*] --> SUBMITTED : feature.md received

    SUBMITTED --> QUEUED : Job registered + risk tier assigned

    QUEUED --> DESIGNING : Assigned in FeatureOutlook\n(self-assign or team assign)

    DESIGNING --> APPROVED : plan.md gate approved\n[Guard: all NFRs reviewed\n+ required approvers met\n+ risk tier constraints]

    APPROVED --> DISPATCHING : Orchestrator packages bundle

    DISPATCHING --> EXECUTING : Workspace provisioned\n+ heartbeat received

    EXECUTING --> INTEGRATING : All PRs reported\n[Guard: all expected repos\nhave PR refs]

    INTEGRATING --> MERGING : Preview + E2E passed\n[Guard: preview deployed\n+ E2E results: PASS]

    MERGING --> COMPLETE : All PRs merged to develop\n[Guard: all repo develop\nCI green]

    %% Escalation paths
    EXECUTING --> ESCALATED : Agent escalation raised
    INTEGRATING --> ESCALATED : Integration failure

    ESCALATED --> DESIGNING : Routed to Phase 2\n(re-design needed)
    ESCALATED --> DISPATCHING : Routed to HITL inline\n(quick fix applied)

    %% Maintenance path
    [*] --> SCAN_COMPLETE : Maintenance scan done
    SCAN_COMPLETE --> AWAITING_APPROVAL : Bundle registered
    AWAITING_APPROVAL --> APPROVED : Per-fix approval granted

    %% Failure
    EXECUTING --> FAILED : Max retries exhausted\n+ escalation unresolved
    INTEGRATING --> FAILED : Unresolvable integration failure
    MERGING --> FAILED : Merge conflict unresolvable

    FAILED --> [*]
    COMPLETE --> [*]
```

### State Machine Configuration (Spring State Machine)

```java
@Configuration
@EnableStateMachineFactory
public class JobStateMachineConfig
        extends EnumStateMachineConfigurerAdapter<JobState, JobEvent> {

    @Override
    public void configure(StateMachineStateConfigurer<JobState, JobEvent> states)
            throws Exception {
        states.withStates()
            .initial(JobState.SUBMITTED)
            .end(JobState.COMPLETE)
            .end(JobState.FAILED)
            .states(EnumSet.allOf(JobState.class));
    }

    @Override
    public void configure(StateMachineTransitionConfigurer<JobState, JobEvent> transitions)
            throws Exception {
        transitions
            // Feature path
            .withExternal()
                .source(SUBMITTED).target(QUEUED)
                .event(JOB_REGISTERED)
                .action(assignRiskTierAction())
                .and()
            .withExternal()
                .source(QUEUED).target(DESIGNING)
                .event(JOB_ASSIGNED)
                .and()
            .withExternal()
                .source(DESIGNING).target(APPROVED)
                .event(PLAN_APPROVED)
                .guard(allNfrsReviewedGuard())
                .guard(requiredApproversMetGuard())
                .guard(riskTierConstraintsGuard())
                .action(recordApprovalAction())
                .and()
            .withExternal()
                .source(APPROVED).target(DISPATCHING)
                .event(DISPATCH_TRIGGERED)
                .action(packageArtifactBundleAction())
                .and()
            .withExternal()
                .source(DISPATCHING).target(EXECUTING)
                .event(WORKSPACE_READY)
                .and()
            .withExternal()
                .source(EXECUTING).target(INTEGRATING)
                .event(ALL_PRS_READY)
                .guard(allExpectedReposHavePRsGuard())
                .action(assemblePRManifestAction())
                .and()
            .withExternal()
                .source(INTEGRATING).target(MERGING)
                .event(PREVIEW_PASSED)
                .guard(e2eResultsPassGuard())
                .and()
            .withExternal()
                .source(MERGING).target(COMPLETE)
                .event(ALL_MERGED)
                .guard(allDevelopCIGreenGuard())
                .action(recordCompletionAction())
                .and()

            // Escalation
            .withExternal()
                .source(EXECUTING).target(ESCALATED)
                .event(ESCALATION_RAISED)
                .action(routeEscalationAction())
                .and()
            .withExternal()
                .source(ESCALATED).target(DESIGNING)
                .event(REROUTED_TO_DESIGN)
                .and()
            .withExternal()
                .source(ESCALATED).target(DISPATCHING)
                .event(HITL_FIX_APPLIED)
                .action(repackageArtifactBundleAction())
                .and()

            // Maintenance path
            .withExternal()
                .source(SUBMITTED).target(SCAN_COMPLETE)
                .event(MAINTENANCE_SCAN_DONE)
                .and()
            .withExternal()
                .source(SCAN_COMPLETE).target(AWAITING_APPROVAL)
                .event(BUNDLE_REGISTERED)
                .and()
            .withExternal()
                .source(AWAITING_APPROVAL).target(APPROVED)
                .event(MAINTENANCE_APPROVED)
                .guard(perFixApprovalsCompleteGuard())
                .and()

            // Failure
            .withExternal()
                .source(EXECUTING).target(FAILED)
                .event(MAX_RETRIES_EXHAUSTED)
                .action(recordFailureAction());
    }
}
```

### Guard Implementations

Guards are precondition checks that must pass before a state transition is allowed.

```mermaid
classDiagram
    class Guard {
        <<interface>>
        +evaluate(context: StateContext) boolean
    }

    class AllNfrsReviewedGuard {
        +evaluate(context) boolean
        -checkNfrCategories() List~NfrCategory~
    }

    class RequiredApproversMetGuard {
        +evaluate(context) boolean
        -getRequiredRoles(riskTier) List~Role~
        -getActualApprovers(jobId) List~Approver~
    }

    class RiskTierConstraintsGuard {
        +evaluate(context) boolean
        -validateTierSpecificRules(tier, job) boolean
    }

    class AllExpectedReposHavePRsGuard {
        +evaluate(context) boolean
        -getExpectedRepos(planMd) List~String~
        -getReportedPRs(jobId) List~PRRef~
    }

    class E2eResultsPassGuard {
        +evaluate(context) boolean
        -getPreviewResult(jobId) PreviewResult
    }

    class AllDevelopCIGreenGuard {
        +evaluate(context) boolean
        -checkCIStatus(repo, sha) CIStatus
    }

    Guard <|.. AllNfrsReviewedGuard
    Guard <|.. RequiredApproversMetGuard
    Guard <|.. RiskTierConstraintsGuard
    Guard <|.. AllExpectedReposHavePRsGuard
    Guard <|.. E2eResultsPassGuard
    Guard <|.. AllDevelopCIGreenGuard
```

### Core Domain Model

```mermaid
classDiagram
    class Job {
        +UUID id
        +String featureId
        +JobSource source
        +JobState state
        +RiskTier riskTier
        +List~ArtifactRef~ artifacts
        +List~GateDecision~ gates
        +List~PRRef~ pullRequests
        +List~EscalationEvent~ escalations
        +Instant createdAt
        +Instant completedAt
        +String assignedTo
    }

    class JobSource {
        <<enumeration>>
        FEATURE
        MAINTENANCE
    }

    class JobState {
        <<enumeration>>
        SUBMITTED
        QUEUED
        DESIGNING
        APPROVED
        DISPATCHING
        EXECUTING
        INTEGRATING
        MERGING
        COMPLETE
        ESCALATED
        FAILED
        SCAN_COMPLETE
        AWAITING_APPROVAL
    }

    class RiskTier {
        <<enumeration>>
        LOW
        MEDIUM
        HIGH
    }

    class GateDecision {
        +UUID id
        +String gateId
        +GateOutcome outcome
        +String decidedBy
        +String role
        +Instant decidedAt
        +String evidence
        +Map~String,String~ metadata
    }

    class EscalationEvent {
        +UUID id
        +EscalationType type
        +String description
        +EscalationTarget target
        +EscalationStatus status
        +String resolvedBy
        +Instant raisedAt
        +Instant resolvedAt
    }

    class EscalationType {
        <<enumeration>>
        ARCHITECTURAL_CONFLICT
        AMBIGUOUS_CRITERIA
        CROSS_REPO_DEPENDENCY
        NO_SAFE_RETRY
        CONTRACT_MISMATCH
        E2E_FAILURE
        PRODUCT_AMBIGUITY
    }

    class EscalationTarget {
        <<enumeration>>
        PHASE_2_TECH_DESIGN
        PHASE_M_MAINTENANCE
        HITL_INLINE
    }

    class AuditEntry {
        +UUID id
        +UUID jobId
        +JobState fromState
        +JobState toState
        +String actor
        +String actorRole
        +Instant timestamp
        +String eventType
        +Map~String,Object~ context
    }

    Job --> JobSource
    Job --> JobState
    Job --> RiskTier
    Job "1" --> "*" GateDecision
    Job "1" --> "*" EscalationEvent
    EscalationEvent --> EscalationType
    EscalationEvent --> EscalationTarget
    Job ..> AuditEntry : every transition produces
```

### Modulith Module Boundaries

Spring Modulith enforces that modules communicate only through events, not direct method calls.

```mermaid
graph TB
    subgraph modules["Spring Modulith — Module Communication"]
        direction LR

        JOB["job\n─────\nJob aggregate\nState machine\nQuery service"]
        GATE["gate\n─────\nGate definitions\nDecision handling\nApprover validation"]
        DISPATCH["dispatch\n─────\nArtifact packaging\nWorkspace provisioning\nHeartbeat monitoring"]
        INTEGRATION["integration\n─────\nPR collection\nPreview trigger\nMerge coordination"]
        ESCALATION["escalation\n─────\nType classification\nTarget routing\nNotification"]
        QUEUE["queue\n─────\nFeature queue\nMaintenance queue\nPriority ordering"]
        AUDIT["audit\n─────\nEvent store\nMetrics aggregation\nCompliance reports"]
    end

    JOB ==>|"JobCreated"| QUEUE
    GATE ==>|"GateApproved"| JOB
    JOB ==>|"StateChanged\n(APPROVED→DISPATCHING)"| DISPATCH
    DISPATCH ==>|"PRReported"| INTEGRATION
    INTEGRATION ==>|"AllPRsReady"| JOB
    INTEGRATION ==>|"AllMerged"| JOB
    DISPATCH ==>|"EscalationRaised"| ESCALATION
    ESCALATION ==>|"EscalationRouted"| JOB

    JOB ==>|"every transition"| AUDIT
    GATE ==>|"every decision"| AUDIT
    ESCALATION ==>|"every routing"| AUDIT

    style JOB fill:#0e1420,stroke:#22d3ee,color:#d4d6de
    style GATE fill:#201c28,stroke:#a78bfa,color:#d4d6de
    style DISPATCH fill:#0e1220,stroke:#3b82f6,color:#d4d6de
    style INTEGRATION fill:#0e1610,stroke:#34d399,color:#d4d6de
    style ESCALATION fill:#1e1600,stroke:#f59e0b,color:#d4d6de
    style QUEUE fill:#16140e,stroke:#f59e0b,color:#d4d6de
    style AUDIT fill:#111115,stroke:#888780,color:#d4d6de
```

### REST API Surface (FeatureOutlook Integration)

```
# Queue
GET    /api/queue/features              → List feature jobs (filterable)
GET    /api/queue/maintenance            → List maintenance bundles
POST   /api/queue/features/{id}/assign   → Self-assign or team-assign

# Jobs
GET    /api/jobs/{id}                    → Job detail + current state
GET    /api/jobs/{id}/history            → State transition history
GET    /api/jobs/{id}/artifacts          → Artifact refs

# Gates
GET    /api/jobs/{id}/gates              → Pending gates for this job
POST   /api/jobs/{id}/gates/{gateId}     → Submit gate decision
                                           Body: { outcome, evidence, metadata }

# Escalations
GET    /api/escalations?status=open      → Open escalations
POST   /api/escalations/{id}/resolve     → Mark resolved

# Metrics
GET    /api/metrics/completion-rate      → Autonomous completion rate
GET    /api/metrics/escalation-rate      → Escalation rate by phase
GET    /api/metrics/cycle-time           → Idea → develop cycle time
```

### Persistence

```
┌─────────────────────────────────────────────┐
│  PostgreSQL                                  │
│                                              │
│  jobs               → Job aggregate root     │
│  gate_decisions      → Gate decisions         │
│  escalation_events   → Escalation log         │
│  audit_entries       → Append-only audit trail │
│  pr_refs             → PR manifest per job    │
│  state_machine_ctx   → SSM persistence        │
│                                              │
│  Redis                                       │
│  queue:features      → Feature job queue      │
│  queue:maintenance   → Maintenance queue      │
│  locks:job:{id}      → Distributed lock       │
│  ws:sessions         → WebSocket sessions     │
└─────────────────────────────────────────────┘
```

### Key Design Decisions

**Why single JAR (Modulith) instead of microservices?**
The Orchestrator's modules share the `Job` aggregate. Splitting them into microservices would require distributed transactions or event sourcing for basic operations like "approve gate and advance state." A Modulith keeps ACID transactions local while enforcing clean module boundaries through Spring's `@ApplicationModuleTest` and event-based communication.

**Why Spring State Machine (not a custom state enum)?**
A custom state enum with switch statements would work for simple lifecycles. But the Orchestrator's lifecycle has 13 states, multiple transition paths (feature vs. maintenance), guarded transitions (NFR review, risk tier), and side effects (dispatch, notification). Spring State Machine makes these declarative, testable, and auditable. Each transition is logged automatically.

**Why append-only audit trail?**
Governed agentic delivery requires accountability. If a defect escapes to production, the audit trail must show: who approved the plan.md, what risk tier was assigned, whether any escalations were overridden, and exactly when each state transition occurred. Append-only ensures this trail cannot be tampered with after the fact.

**Why Redis for queues (not PostgreSQL)?**
Queues are hot-path operations — FeatureOutlook polls them frequently, and self-assignment needs to be fast with proper locking. Redis provides atomic queue operations and distributed locks that PostgreSQL can handle but with higher latency. The queue state is derived from job state in PostgreSQL, so Redis is a cache — if it fails, queues rebuild from the database.

### Operating Metrics

The Orchestrator tracks five key metrics that measure the health of the governed delivery model:

| Metric | Target | Source |
|--------|--------|--------|
| **Autonomous Completion Rate** | > 70% | Jobs that complete Phase 3 without HITL escalation |
| **Escalation Rate** | Track per phase | Percentage of jobs escalated at each phase boundary |
| **Defect Escape Rate** | < 5% | Defects found post-merge to `develop` |
| **Cycle Time** | Trend downward | Time from curated feature.md to merged to `develop` |
| **Rework Rate** | Trend downward | Jobs that return to Phase 2 from Phase 3 |
