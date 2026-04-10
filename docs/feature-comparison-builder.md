# Feature Comparison: Governed Agentic Delivery (GAD) vs. Builder

> **Date:** 2026-04-09
> **GAD Version:** v1.1 (March 2026)
> **Builder Version:** koehler8/builder @ 2026-04-09 (MIT License)

---

## Executive Summary

Both systems orchestrate AI agents to deliver software from intent to merged code, with human governance at key gates. They diverge sharply in **scale target** and **infrastructure weight**:

| | **Governed Agentic Delivery** | **Builder** |
|---|---|---|
| **Target** | Team/enterprise — multi-product, multi-repo, multi-role | Solo builder — one person, many projects |
| **Infrastructure** | Server-side platform (Spring Boot, PostgreSQL, Neo4j, Redis) | CLI tool (Node.js, GitHub Projects as state store) |
| **Agent runtime** | DevContainer + spec-kit + LangChain + OpenCode | Claude Code CLI subprocesses |
| **State management** | Orchestrator service + Spring State Machine | GitHub Projects V2 (GraphQL) |
| **Philosophy** | "Humans govern meaning, agents execute within approved scope" | "One builder. Many projects. AI agents do the work — you make the decisions." |

---

## Detailed Feature Comparison

### 1. Delivery Pipeline / Workflow

| Feature | GAD | Builder |
|---|---|---|
| **Pipeline phases** | Phase 1 (upstream) -> Phase 2 (tech design) -> Dispatch -> Phase 3a (execution) -> Phase 3b (integration & ship) + Phase M (maintenance) | Linear: intent -> refine -> validate -> plan -> test -> build -> accept -> done |
| **Number of stations** | 28 stations across 3 lanes (Human, System, Agent) | 8 states in a single pipeline |
| **Parallel lanes** | Yes — Human, System, and Agent lanes run concurrently | No — single sequential pipeline |
| **Cross-repo coordination** | Native — multi-repo PRs collected, preview environment deployed, dependency-ordered merge | Single-repo per intent — no cross-repo orchestration |
| **Artifact chain** | feature.md -> acceptance-criteria.md -> architecture.md -> ux.md -> qa-tests.md -> plan.md -> tasks.json -> PRs | 1-INTENT.md -> 2-REFINED.md -> 3-VALIDATION.md -> 4-BUILD-PLAN.md -> 5-TEST-PLAN.md -> 6-ACCEPT.md |
| **Maintenance pipeline** | Dedicated Phase M — scheduled scanning, triage, maintenance bundles | Not present |
| **Product registration** | Formal one-time onboarding (workspace provisioning, graph seeding, constitution setup) | `builder connect <repo>` — lightweight project board linking |

### 2. Agent Architecture

| Feature | GAD | Builder |
|---|---|---|
| **Agent framework** | spec-kit + LangChain + LangSmith (Python), 6 overridable agents | Claude Code CLI subprocesses, 8 specialized agents |
| **Agent types** | constitution, analyze, tasks, checklist, implement, taskstoissues + Task Agent, Execution Agent, Test Agent, Validation Agent, Review Agent | PM Assignment, PM Refinement, Validator, Planner, Tester, Builder, Acceptor, Resolver |
| **Agent isolation** | DevContainer (Docker) per job — full sandboxed environment | Git worktrees per stage — filesystem-level isolation |
| **Agent model selection** | GitHub Models (primary), OpenRouter (fallback), local fallbacks | Configurable per agent role — split between Sonnet and Opus |
| **Agent observability** | LangSmith traces + audit trail in PostgreSQL | Usage JSONL (cost, tokens, duration per run) + `builder quota` |
| **Merge conflict resolution** | Orchestrator's Merge Coordinator handles dependency-ordered merging | Dedicated Resolver agent — attempts rebase, then AI-assisted conflict resolution |
| **Rework cycles** | Three-outcome gate: Continue / Auto-Refine (max 3) / Escalate | Accept -> Test -> Build -> Accept cycle (max 3 before human escalation) |
| **Product ideation** | Not present | `builder dream` — AI-powered product ideation with web research |

### 3. Governance & Human Oversight

| Feature | GAD | Builder |
|---|---|---|
| **Risk tiers** | Low / Medium / High — scored by Orchestrator (repo criticality, data surface), overridable by Tech Lead | Not present — uniform governance per mode |
| **Autonomy modes** | Risk-tier-driven: Low = autonomous, Medium = elevated review, High = sign-off at every gate | Three modes: Normal (standard gates), Freak (all gates), Yolo (full auto with degradation) |
| **Gate decisions** | Approve / Auto-Refine / Escalate | Approve / Decline / Refine / Skip / Nix / Quit |
| **Escalation routing** | 7 classified types routed to Phase 2, Phase M, or HITL inline — 4-hour SLA target | Implicit — after 3 failed rework cycles, falls back to human |
| **Escalation types** | ARCHITECTURAL_CONFLICT, AMBIGUOUS_CRITERIA, CROSS_REPO_DEPENDENCY, NO_SAFE_RETRY, CONTRACT_MISMATCH, E2E_FAILURE, PRODUCT_AMBIGUITY | Not classified — generic escalation |
| **NFR review** | Formal 5-category checklist: Security, Observability, Performance, Accessibility, Data/Privacy | Not present — quality assessed by Acceptor agent |
| **Multi-role governance** | PM, Tech Lead, Architect, UX Designer, QA Engineer, Developer — each with distinct responsibilities | Single role: "the builder" makes all decisions |
| **Audit trail** | Append-only database with actor, timestamp, evidence, event type, context | GitHub issue comments + transitions.jsonl |
| **Documentation enforcement** | Constitution rules enforced by spec-kit agents | Builder agent instructed to update README/CLAUDE.md; Acceptor verifies |

### 4. Context & Knowledge Management

| Feature | GAD | Builder |
|---|---|---|
| **Knowledge architecture** | Three-tier: Neo4j (fleet graph) -> PostgreSQL (project registry) -> Kuzu (per-container edge) | Flat: GitHub Project metadata + AI-synthesized README summaries in config |
| **Cross-repo intelligence** | Neo4j graph with modules, dependencies, patterns, run history — tenant-scoped | Not present — each repo is independent |
| **Per-job context** | Context Edge (Kuzu embedded graph) as MCP sidecar — fast local queries, buffered writes | Agent prompts include issue context + previous agent output + builder comments |
| **MCP integration** | Native — Context Edge serves as MCP server for OpenCode enrichment | Not present |
| **Context propagation** | Graph snapshots exported per job, enrichment queries during execution | Artifact chain — each agent reads prior stage artifacts |
| **Multi-tenancy** | PostgreSQL RLS + Neo4j tenant-scoped queries | Not applicable — single-user tool |

### 5. Infrastructure & Tech Stack

| Feature | GAD | Builder |
|---|---|---|
| **Orchestration** | Spring Boot 3.x + Spring Modulith (8 modules) + Spring State Machine | Node.js CLI with Commander |
| **State persistence** | PostgreSQL (jobs, audit) + Redis (queues, locks) | GitHub Projects V2 (GraphQL API) |
| **Graph database** | Neo4j (fleet) + Kuzu (per-container) | None |
| **Execution runtime** | DevContainer (Docker) with Python, Node, Java build tools | Local machine — git worktrees for isolation |
| **LLM integration** | spec-kit -> LangChain -> OpenCode CLI -> GitHub Models | Direct Claude Code CLI subprocess spawning |
| **Dependencies** | Spring Boot, PostgreSQL, Redis, Neo4j, Docker, FastAPI, LangChain, LangSmith | 2 npm packages (commander, marked) + gh CLI + claude CLI |
| **CI/CD integration** | Harness.io — preview environment provisioning, E2E pipeline | Relies on target repo's existing CI; checks via `gh pr checks` |
| **Database** | PostgreSQL + Neo4j + Redis + Kuzu | None — GitHub is the database |

### 6. Testing & Quality

| Feature | GAD | Builder |
|---|---|---|
| **Test approach** | spec-kit agents generate tests as part of implementation; quality gates validate | TDD — Tester agent writes tests first, Builder implements to pass them |
| **Quality gates** | Three-outcome per station: Continue / Auto-Refine (max 3) / Escalate | Binary per gate: Approve / Decline (with rework cycle) |
| **Preview environments** | Ephemeral per-PR-set — full-stack cross-repo deployment via Harness | Not present |
| **E2E testing** | Cross-repo E2E in preview environment (contracts, integration, perf) | Not present — per-repo CI only |
| **Security scanning** | Phase M — Qodana, Snyk, SonarQube integration | Not present |

### 7. Developer Experience

| Feature | GAD | Builder |
|---|---|---|
| **Primary interface** | VS Code Extension (FeatureOutlook) + TUI (Ink) + CLI | CLI only (`builder` command, 18 subcommands) |
| **Work discovery** | Queue view in FeatureOutlook — self-assignment from queue | `builder act` — Marshaler agent prioritizes across all projects |
| **Process trigger** | Orchestrator-driven — always-on service processes automatically | Explicit `builder process` — human controls when agents run |
| **Status visibility** | Real-time via WebSocket in FeatureOutlook | `builder status` — dashboard of all items by state |
| **Multi-project management** | Product-scoped workspaces with constitution + overrides | Central main queue + per-project boards |
| **Metrics** | Operating metrics: Autonomous Completion Rate, Escalation Rate, Defect Escape Rate, Cycle Time, Rework Rate | Pipeline metrics: cycle time, lead time, velocity, throughput, dwell times, gate wait, rework rate, cost per story point, estimation accuracy |
| **Cost tracking** | LangSmith traces | Per-run JSONL: cost, tokens, duration — `builder quota` |

### 8. Deployment & Operations

| Feature | GAD | Builder |
|---|---|---|
| **Deployment model** | Server-side platform — always-on services (Orchestrator, Context Server, PDM) | Local CLI tool — `npm link` install |
| **Scaling** | Multi-tenant, multi-team, multi-product | Single user, multi-project |
| **Process model** | Always-on services + ephemeral containers per job | On-demand CLI runs with process locking (mutex) |
| **Configuration** | Product-scoped: constitution, agent overrides, workspace file, .devcontainer | Global: `~/.builder/config.json` with agent model assignments |
| **Plugin system** | 4 optional plugins: Task Queue, PR Composer, Quality Runner, Context Edge — each with fallback defaults | Not present |
| **Concurrency control** | Redis locks + Orchestrator queue management | File-based process.lock with stale detection (30 min / dead PID) |

---

## Unique to GAD (Not in Builder)

- **Cross-repo orchestration** — multi-repo PR coordination, dependency-ordered merge
- **Preview environments** — ephemeral full-stack deployment per feature
- **Graph-based context** — Neo4j fleet intelligence + Kuzu per-container edge
- **Risk tier system** — automated scoring with human override
- **Multi-role governance** — PM, Tech Lead, Architect, UX, QA, Developer
- **NFR checklists** — formal non-functional requirement review
- **Escalation classification** — 7 typed escalation categories with routing
- **MCP protocol** — context enrichment via Model Context Protocol
- **Maintenance pipeline** — Phase M for security scanning and dependency management
- **Constitution system** — governance rules baked into agent behavior
- **DevContainer isolation** — full Docker-based execution sandboxing
- **Product registration** — formal onboarding with workspace provisioning
- **Plugin architecture** — overridable components with fallback defaults
- **VS Code extension** — rich IDE integration for review workflows

## Unique to Builder (Not in GAD)

- **Dream command** — AI-powered product ideation with web research
- **Yolo mode** — full automation with graceful degradation (auto-answers questions, falls back on failure)
- **Story point estimation** — Validator assigns Fibonacci points, tracked via labels
- **Marshaler prioritization** — AI-powered cross-project work prioritization
- **Merge conflict resolver** — dedicated agent for conflict resolution
- **Cost-per-story-point metric** — AI cost efficiency tracking
- **Estimation accuracy metric** — story points vs. actual cycle time correlation
- **Zero-infrastructure design** — GitHub Projects as database, no servers to deploy
- **TDD-first approach** — tests written before implementation by separate agent
- **Background process spawning** — `builder add` and `builder act` auto-spawn processing
- **Per-agent model configuration** — granular model assignment per agent role

---

## Architecture Philosophy Comparison

| Dimension | GAD | Builder |
|---|---|---|
| **Complexity** | High — distributed platform with 5+ services | Low — single CLI binary + GitHub |
| **Setup cost** | Significant — requires PostgreSQL, Redis, Neo4j, Docker, Harness | Minimal — `npm link` + `gh auth` + `claude` CLI |
| **Operational burden** | Always-on services to maintain | Zero — runs on demand |
| **Team readiness** | Designed for team adoption with role-based workflows | Designed for individual power users |
| **Customizability** | Deep — constitutions, agent overrides, plugins, per-product config | Moderate — agent model selection, execution modes |
| **Auditability** | Enterprise-grade — append-only audit trail, typed escalations, SLAs | Pragmatic — GitHub comments + local JSONL logs |
| **Cross-repo** | First-class concern — graph intelligence, coordinated PRs, preview envs | Not addressed — single-repo per intent |

---

*Generated 2026-04-09 — source: GAD v1.1 architecture docs + koehler8/builder repository analysis*
