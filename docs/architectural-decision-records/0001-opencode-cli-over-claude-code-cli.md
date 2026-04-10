# ADR-0001: OpenCode CLI over Claude Code CLI for Execution Engine Code Generation

- **Status:** Accepted
- **Date:** 2026-04-09
- **Decision Makers:** Edwin Cruz
- **Context Area:** Execution Environment / Code Generation Layer

---

## Context

The Execution Environment is the worker inside our Governed Agentic Delivery pipeline. At stations 17-18 (Phase 3a — Agent Execution), `speckit.implement` drives a code generation CLI to produce code and tests per task. This CLI must:

1. **Be wrappable by LangChain** — spec-kit orchestrates agents via LangChain + LangSmith. The CLI must be invocable programmatically (subprocess or SDK) with structured output.
2. **Support MCP (Model Context Protocol)** — Context Edge runs as an MCP sidecar inside the DevContainer. The CLI must connect to it for live graph enrichment during code generation.
3. **Run headlessly in a DevContainer** — no TTY, no interactive prompts, fully autonomous.
4. **Allow tool restriction** — we need to control what the CLI can do (e.g., allow file writes but restrict network access).
5. **Be deployable in a Docker image** — single binary or simple install, no heavy runtime dependencies.
6. **Be permissively licensed** — we embed this in our platform; proprietary or restrictive licenses are problematic.

The LLM provider is **not** a fixed constraint — it would follow the CLI choice. OpenCode would mean GitHub Models (via Copilot); Claude Code would mean Anthropic. Both are viable providers. The question is which CLI + provider combination best serves the execution engine when wrapped by LangChain.

We evaluated two candidates: **OpenCode CLI** (`opencode-ai/opencode`) and **Claude Code CLI** (`@anthropic-ai/claude-code`).

---

## Decision

**Use OpenCode CLI** (the MIT-licensed archived version) as the code generation driver inside the Execution Environment, wrapped by LangChain via subprocess invocation.

Pin to the last MIT-licensed release and vendor the binary in our DevContainer image. Do **not** adopt the successor project (Crush/Charmbracelet) due to its non-MIT license. The LLM provider consequently becomes GitHub Models via the Copilot provider.

---

## Rationale

### Side-by-Side Evaluation

| Dimension | OpenCode CLI | Claude Code CLI |
|---|---|---|
| **LLM provider** | GitHub Models (Copilot) — zero incremental cost with subscription | Anthropic — $3-15/MTok (Sonnet), $5-25/MTok (Opus) |
| **LangChain wrapping** | `opencode -p "..." -f json -q` — subprocess with JSON output | `-p - --output-format json` — subprocess with JSON. Also has first-party Python/TS Agent SDK with streaming, hooks, subagents |
| **MCP support** | First-class via `mcp-go` — stdio + SSE transports | First-class — stdio + HTTP + SSE. Can also *serve* as MCP server |
| **Headless execution** | `-p` flag — all permissions auto-approved | `-p` flag — requires `--dangerously-skip-permissions` or explicit `--allowedTools` |
| **Tool restriction** | None in non-interactive mode — auto-approves everything | Fine-grained: `--allowedTools "Read,Edit,Bash(git *)"`, `--disallowedTools`, `--permission-mode` |
| **DevContainer install** | Single Go binary (~30MB) | Node.js package (npm install) |
| **License** | MIT (archived version) | Proprietary — "All rights reserved", Anthropic Commercial ToS |
| **Project status** | Archived (Sep 2025) — no updates | Actively maintained by Anthropic |
| **Multi-provider** | 10+ providers (Anthropic, OpenAI, Gemini, Copilot, Bedrock, Groq, OpenRouter, etc.) | Anthropic only (via Direct, Bedrock, Vertex, Foundry) |
| **Streaming output** | Not available in non-interactive mode | `--output-format stream-json` — real-time NDJSON |
| **SDK/library mode** | None — subprocess only | Python + TypeScript SDKs with hooks, session mgmt, subagent orchestration |
| **Model quality access** | Claude Sonnet 4, GPT-4.1, Gemini 2.5 Pro, O4-mini (via Copilot) | Claude Opus 4, Sonnet 4, Haiku 4.5 (direct access, latest models first) |
| **Context management** | Auto-compact at 95% context window | Auto-compact + prompt caching (90%+ cache reads) |
| **Cost at scale** | ~$0/job (Copilot subscription) | ~$0.50-5.00/job (token-based, varies by model + task size) |

### Why OpenCode Wins: License + Cost + Flexibility

The decision rests on three reinforcing advantages:

**1. MIT License vs. Proprietary**

OpenCode is MIT-licensed — we can vendor, fork, modify, and redistribute freely. Claude Code is proprietary ("All rights reserved", Anthropic Commercial ToS). Embedding a proprietary CLI inside our execution engine creates legal exposure: we cannot patch it, we cannot redistribute it without Anthropic's terms, and we cannot guarantee availability if terms change.

For a component running inside every job's DevContainer, the license is load-bearing infrastructure.

**2. Cost Structure at Scale**

Our execution engine processes many jobs, each involving multiple `speckit.implement` invocations. The cost models diverge significantly:

| Scenario | OpenCode + GitHub Models | Claude Code + Anthropic |
|---|---|---|
| Per-job inference | $0 incremental (Copilot subscription) | $0.50-5.00 (Sonnet), $2-15 (Opus) |
| 100 jobs/month | ~$0 | ~$50-500 (Sonnet), $200-1500 (Opus) |
| 1000 jobs/month | ~$0 | ~$500-5000 (Sonnet), $2000-15000 (Opus) |

GitHub Copilot's flat-rate model means code generation cost does not scale with usage. This is a structural advantage for a platform that aims to increase autonomous throughput over time.

**3. Provider Flexibility as Insurance**

OpenCode supports 10+ LLM providers. If GitHub Models degrades, we switch to OpenRouter, direct Anthropic, or local models — all via config change. Claude Code locks us to Anthropic's ecosystem. In a platform where the CLI is abstracted behind spec-kit, provider flexibility at the CLI level compounds the abstraction boundary's value.

### Where Claude Code Is Genuinely Stronger

These are real trade-offs, not dismissed:

- **Agent SDK quality**: Claude Code's Python/TypeScript SDK (`claude-agent-sdk`) offers streaming, hooks (`PreToolUse`, `PostToolUse`, `Stop`), subagent orchestration, and session management. This is a materially better integration surface than subprocess JSON parsing. If we chose Claude Code, our LangChain wrapper would be thinner and more capable.

- **Tool restriction granularity**: Claude Code's `--allowedTools "Read,Edit,Bash(git *)"` with prefix matching gives fine-grained control over what the agent can do. OpenCode auto-approves everything in `-p` mode. For a security-conscious execution engine, this matters.

- **Streaming output**: `--output-format stream-json` enables real-time progress visibility. Our LangChain wrapper could surface incremental updates to LangSmith traces. OpenCode blocks until completion.

- **Active maintenance**: Claude Code receives continuous updates, security patches, and new features. OpenCode is frozen in time.

- **Model access timing**: Anthropic ships new Claude models to Claude Code first. Via GitHub Models/Copilot, we get new models when GitHub adds them — potentially weeks later.

- **MCP server capability**: Claude Code can expose its own tools via `claude mcp serve`, enabling bidirectional MCP architectures. OpenCode is MCP client only.

### Why These Strengths Don't Tip the Balance

The Agent SDK advantage is significant but **mitigable** — subprocess JSON parsing is a well-understood pattern (koehler8/builder runs 8 agents this way in production). The integration is coarser but functional.

Tool restriction is handled at a different layer — the **DevContainer itself** is the security boundary (network policy, filesystem mounts, credential scoping). Container-level sandboxing is more robust than CLI-level allowlists anyway, as CLI-level restrictions have known bypass vectors.

Streaming is a nice-to-have for observability but not essential — LangSmith traces capture inputs/outputs regardless, and job-level progress is tracked by the Orchestrator via heartbeats.

Active maintenance is the strongest argument for Claude Code, but it's offset by the archived-project mitigations below, and by the fact that a proprietary dependency can change terms at any time.

### Risk: OpenCode Is Archived

OpenCode was archived in September 2025 when its creator joined the Charm team. The successor project, **Crush** (`charmbracelet/crush`), has ~22,700 stars and is actively maintained but uses a **non-MIT license** (NOASSERTION/custom).

**Mitigation strategy:**

1. **Vendor and pin**: Fork the last MIT release into our build pipeline. The binary is self-contained Go — no runtime dependencies to rot.
2. **Containment**: OpenCode runs inside an ephemeral DevContainer. Its blast radius is limited to a single job's lifecycle. If it breaks, the job escalates to a human — this is already a first-class path in our architecture.
3. **Abstraction boundary**: spec-kit wraps OpenCode behind its `implement` agent interface. If we need to swap to Crush, Claude Code, a direct LangChain LLM integration, or another CLI in the future, the change is localized to the spec-kit implement agent — no upstream pipeline changes needed.
4. **Monitor Crush license**: If Charm adopts a permissive license, we can migrate. The tool and MCP interfaces are compatible.

### Risk: No Tool Restriction in Non-Interactive Mode

OpenCode auto-approves all tool permissions when running with `-p`. Unlike Claude Code's `--allowedTools`, there is no way to restrict what tools the agent can use via configuration.

**Mitigation:**

1. **DevContainer sandboxing**: The container itself is the security boundary. Network access, filesystem mounts, and credentials are controlled at the Docker level, not the CLI level.
2. **Ephemeral by design**: Each job gets a fresh container. Any damage is contained and discarded on teardown.
3. **spec-kit governance**: The constitution and checklist agents validate outputs before they leave the container. Dangerous operations are caught at the governance layer, not the CLI layer.

---

## Alternatives Considered

### Alternative 1: Claude Code CLI with Anthropic API

Use Claude Code as the code generation CLI, with Anthropic as the LLM provider.

**Rejected because:**
- Proprietary license creates legal exposure for an embedded platform component
- Per-token cost model scales poorly as autonomous throughput increases
- Single-provider lock-in (Anthropic only) reduces resilience
- The Agent SDK advantage, while real, does not outweigh license + cost + flexibility

**Would reconsider if:**
- Anthropic releases Claude Code under a permissive license
- Claude Code adds multi-provider support (beyond Anthropic model variants)
- Our job volume stays low enough that per-token cost is immaterial

### Alternative 2: Direct LangChain LLM Integration (No CLI Wrapper)

Skip the CLI entirely. Use LangChain's native `ChatOpenAI` or `ChatAnthropic` providers with custom tool implementations for file I/O, bash execution, and MCP queries.

**Rejected because:**
- Requires reimplementing the entire agentic coding loop: file discovery, context management, edit application, LSP integration, auto-compaction
- OpenCode has solved these problems with tested tool implementations (`view`, `edit`, `write`, `patch`, `glob`, `grep`, `bash`)
- Significantly increases our development and maintenance surface
- However, this remains a viable **fallback** if OpenCode proves too unstable in production

### Alternative 3: Crush (Charmbracelet Fork)

Adopt the actively maintained successor to OpenCode.

**Deferred because:**
- License is listed as NOASSERTION — needs legal review before commercial embedding
- Functional capabilities are similar to OpenCode (same codebase lineage)
- If/when the license is clarified as permissive, this becomes the preferred upgrade path

### Alternative 4: Aider CLI

Open-source AI coding assistant with multi-model support.

**Not evaluated in depth because:**
- Python-based — heavier DevContainer footprint than a Go binary
- Less mature MCP support at time of evaluation
- Could be a future candidate if OpenCode proves insufficient

---

## Consequences

### Positive

- **Zero incremental LLM cost** — GitHub Models via Copilot subscription covers all code generation inference
- **MIT license** — no legal constraints on vendoring, modification, or redistribution
- **Provider flexibility** — 10+ LLM providers available via config change if GitHub Models degrades
- **MCP-native** — Context Edge integration works out of the box via `.opencode.json` config
- **Single binary** — trivial DevContainer installation, no Node.js/Python runtime needed for the CLI itself

### Negative

- **Archived project** — no upstream security patches, bug fixes, or feature development
- **No SDK** — coarser-grained integration (subprocess only) compared to Claude Code's Agent SDK
- **No tool restriction** — must rely on container-level sandboxing rather than CLI-level permissions
- **No streaming in non-interactive mode** — LangChain wrapper cannot show incremental progress
- **Fork maintenance burden** — we own the vendored binary going forward
- **Model access lag** — new models arrive via GitHub/Copilot on their schedule, not day-one

### Neutral

- **Monitoring required** — track Crush license changes; evaluate migration when/if permissive
- **spec-kit abstraction** — the implement agent interface isolates the CLI choice; future swaps are localized
- **LLM provider follows CLI** — choosing OpenCode means GitHub Models; this is a trade-off, not a constraint

---

## Integration Pattern

```
spec-kit (Python/LangChain)
  └── speckit.implement agent
        └── subprocess: opencode -p "{enriched_prompt}" -f json -q
              ├── .opencode.json (model: copilot.claude-sonnet-4)
              ├── mcpServers.context-edge (stdio → Context Edge sidecar)
              └── tools: view, edit, write, patch, bash, glob, grep
                    └── LLM: GitHub Models (via Copilot provider)
```

```json
// .opencode.json in DevContainer
{
  "providers": {
    "copilot": { "disabled": false }
  },
  "agents": {
    "coder": {
      "model": "copilot.claude-sonnet-4",
      "maxTokens": 5000
    }
  },
  "mcpServers": {
    "context-edge": {
      "type": "stdio",
      "command": "/usr/local/bin/context-edge-mcp"
    }
  }
}
```

---

## References

- [OpenCode CLI Repository (archived)](https://github.com/opencode-ai/opencode) — MIT License
- [Crush (successor)](https://github.com/charmbracelet/crush) — NOASSERTION License
- [Claude Code CLI](https://github.com/anthropics/claude-code) — Proprietary
- [Claude Code License](https://github.com/anthropics/claude-code/blob/main/LICENSE.md) — "All rights reserved"
- [Claude Agent SDK (Python)](https://github.com/anthropics/claude-agent-sdk-python)
- [GAD Execution Environment Component](/docs/c4-architecture/components/execution-environment/README.md)
- [GAD Context Edge Component](/docs/c4-architecture/components/context-edge-server/README.md)
