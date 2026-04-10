# ADR-0002: KuzuDB over ChromaDB for Context Edge GraphRAG

- **Status:** Accepted
- **Date:** 2026-04-09
- **Decision Makers:** Edwin Cruz
- **Context Area:** Context Edge Server / Graph Intelligence Layer

---

## Context

The Context Edge is an ephemeral in-container sidecar that provides sub-millisecond graph intelligence to agents during code generation (stations 16-18, Phase 3a). It runs inside each DevContainer as a FastAPI-based MCP server, serving six tools to LangChain and OpenCode.

The Context Edge must:

1. **Store a tenant-scoped graph snapshot** — pulled from the central Context Server (Neo4j) at provision time. The data is inherently graph-structured: modules, dependencies, patterns, run history, and constitution rules connected by typed edges (`DEPENDS_ON`, `APPLIES_TO`, `EXECUTED`, `USES`, `DISCOVERED`).
2. **Serve multi-hop Cypher queries** — enrichment queries traverse relationships: "find patterns that apply to module X, its dependencies, and their prior run results." These are 2-3 hop graph traversals with property filtering.
3. **Run embedded (no server process)** — the sidecar loads the database as a library. No network calls, no separate container, no port conflicts.
4. **Buffer writes and flush to Neo4j** — during execution, writes accumulate locally. On teardown, the buffer flushes back to Context Server for ingestion into Neo4j.
5. **Degrade gracefully** — three modes: Full (Kuzu + Context Server), Stale (Kuzu + volume, no server), Null (no Kuzu, spec-only). The pipeline never breaks.
6. **Support Cypher** — the central store is Neo4j. Query language compatibility between edge and central minimizes translation overhead.
7. **Be embeddable and permissively licensed** — MIT or Apache 2.0, deployable in Docker.

We evaluated **KuzuDB** (embedded property graph database) and **ChromaDB** (embedded vector database) for this role.

---

## Decision

**Use KuzuDB** as the embedded graph engine for Context Edge.

Pin to the last MIT-licensed release (v0.11.3) and vendor in the DevContainer image. Monitor **LadybugDB** (the actively maintained community fork at v0.12.0) as the upgrade path when KuzuDB's archived status becomes a practical concern.

---

## Rationale

### The Core Issue: Graph Database vs. Vector Database

KuzuDB and ChromaDB are **different categories of database** solving fundamentally different problems. This is not a close trade-off — it is a category selection.

| Capability | KuzuDB | ChromaDB |
|---|---|---|
| **Data model** | Property graph (nodes + typed edges + properties) | Flat collections (documents + embeddings + metadata) |
| **Query language** | Cypher (Neo4j-compatible) | Python API / REST (no query language) |
| **Multi-hop traversal** | Native — `MATCH (a)-[:DEPENDS_ON*1..3]->(b)` | Impossible — no edge concept |
| **Relationship types** | First-class: `DEPENDS_ON`, `APPLIES_TO`, `EXECUTED`, `USES` | None — relationships must be flattened into metadata |
| **Schema enforcement** | Required — node/edge tables with types and primary keys | None — schemaless metadata |
| **Vector search (HNSW)** | Yes — native extension, built into v0.11.3 | Yes — core feature |
| **Full-text search** | Yes — BM25 extension, built into v0.11.3 | Partial — `where_document` filtering |
| **Graph algorithms** | Yes — PageRank, Louvain, shortest paths, K-Core | None |
| **Neo4j compatibility** | High — Cypher syntax, migration extension, same data model | None — no Cypher, no graph model, lossy import |

### Why Context Edge Needs a Graph Database

The Context Edge serves six MCP tools. Every read tool requires graph traversal:

**`context_query_enrichment`** (called by LangChain per task):
```cypher
-- Find patterns for a module, with confidence filtering
MATCH (p:Pattern)-[:APPLIES_TO]->(m:Module) 
WHERE m.name = $module AND p.confidence > 0.7
RETURN p.name, p.content, p.confidence

-- Retrieve dependency edges
MATCH (m:Module)-[:DEPENDS_ON]->(dep:Module) 
WHERE m.name = $module
RETURN dep.name, dep.repo

-- Prior run results for this module
MATCH (r:Run)-[:EXECUTED]->(m:Module) 
WHERE m.name = $module
RETURN r.status, r.timestamp ORDER BY r.timestamp DESC LIMIT 5
```

**`context_query_live`** (called by OpenCode via MCP):
```cypher
-- API surface and contracts
MATCH (m:Module)-[:USES]->(api:Module) 
WHERE m.name = $module
RETURN api.name, api.contract_spec
```

These are **inherently graph queries**. They follow typed edges between nodes, filter by properties, and return structured results. ChromaDB has no mechanism to express or execute any of these.

### What ChromaDB Would Require

To serve the same queries with ChromaDB, the Context Edge would need to:

1. **Flatten the graph into documents** — each module becomes a document with all its relationships encoded as metadata fields (`depends_on: ["mod_a", "mod_b"]`). This loses relationship types, edge properties, and traversal capability.

2. **Implement graph traversal in application code** — multi-hop queries become multiple ChromaDB calls stitched together with Python loops. A 3-hop dependency query becomes O(n) sequential calls instead of a single Cypher statement.

3. **Abandon Cypher** — all queries must be rewritten as Python-level metadata filters and post-processing. The shared query language between Context Edge (Kuzu) and Context Server (Neo4j) is lost.

4. **Lose schema enforcement** — ChromaDB's schemaless metadata cannot enforce that every Module has required fields or that DEPENDS_ON edges have valid source/target types. Schema drift becomes undetectable.

5. **Lose Neo4j snapshot fidelity** — the graph snapshot from Context Server cannot be imported into ChromaDB without destructive transformation. Nodes survive, edges are lost or denormalized.

This would effectively require rebuilding a graph engine on top of a vector store — the wrong abstraction for the problem.

### Where ChromaDB Excels (And Why It Doesn't Apply Here)

ChromaDB is an excellent choice for:

- **Semantic search RAG** — "find code chunks similar to this query" via embedding similarity
- **Document retrieval** — nearest-neighbor lookup in embedding space
- **Unstructured knowledge bases** — when relationships are implicit in text, not explicit in structure

The Context Edge does **not** currently use semantic search. All queries are deterministic graph traversals — following edges and filtering properties. The enrichment model is structural (dependency trees, pattern associations, run history), not semantic (embedding similarity).

If semantic search were added in the future, KuzuDB covers this too — v0.11.3 includes a native HNSW vector index extension:

```cypher
-- KuzuDB can do vector search natively
CALL CREATE_VECTOR_INDEX('Module', 'mod_emb_idx', 'embedding', metric := 'cosine');
CALL QUERY_VECTOR_INDEX('Module', 'mod_emb_idx', $query_vector, 10) 
  RETURN node, distance;
```

This means KuzuDB can serve as a **unified graph + vector engine**, eliminating the need for a separate vector store. A hybrid query — "find modules semantically similar to X, then traverse their dependencies" — is a single query path in KuzuDB, but would require two databases and application-level joins with ChromaDB.

### KuzuDB Fitness for Context Edge

KuzuDB maps directly to every Context Edge requirement:

| Requirement | How KuzuDB Satisfies It |
|---|---|
| Tenant-scoped graph snapshot | Bulk-load JSON from Context Server into typed node/edge tables |
| Multi-hop Cypher queries | Native Cypher with CSR adjacency lists — sub-millisecond traversal |
| Embedded (no server) | `import kuzu; db = kuzu.Database(path)` — zero infrastructure |
| Write buffer + flush | Write to Kuzu (query consistency) + append to buffer (eventual flush) |
| Graceful degradation | In-memory mode for transient, file-backed for stale, None for null |
| Neo4j compatibility | Same Cypher dialect, same property graph model, migration extension |
| Schema enforcement | `CREATE NODE TABLE`, `CREATE REL TABLE` — prevents drift |
| FastAPI integration | Async Python API (`kuzu.AsyncConnection`) designed for FastAPI |
| Single-file persistence | `.kuzu` file — trivially copyable, volume-mountable |
| Cold start | Sub-second library load, scoped snapshot bulk-load in seconds |

### Risk: KuzuDB Is Archived

KuzuDB was archived in October 2025 after the team was acquired by Apple. v0.11.3 is the final release.

**This is a known risk, shared with ADR-0001 (OpenCode CLI).** The mitigation pattern is the same:

1. **Vendor and pin v0.11.3** — the binary is self-contained, MIT-licensed, and includes all needed extensions (vector, FTS, algo, JSON).
2. **Containment** — KuzuDB runs inside an ephemeral DevContainer. Its blast radius is a single job. Container rebuild fixes any corruption.
3. **LadybugDB as upgrade path** — actively maintained community fork (v0.12.0 released), same API, same Cypher dialect, dropped CLA for community contribution. Led by Arun Sharma (ex-Facebook/Google). If KuzuDB v0.11.3 hits a blocking bug, LadybugDB is a drop-in replacement.
4. **Abstraction boundary** — the `ContextProvider` interface (`query()`, `write()`, `flush()`) abstracts the storage backend. Swapping KuzuDB for LadybugDB, or even a different embedded graph DB, is localized to `backend/kuzu_client.py`.
5. **Vela fork** — addresses the single-writer constraint for concurrent AI agent workloads. Available if concurrency becomes a bottleneck.

### Risk: Single-Writer Constraint

KuzuDB allows only one write transaction at a time. In the Context Edge, writes come from two sources: LangChain (task results, patterns) and the seeder (repo scanning at startup).

**Mitigation:**
- Writes are serialized through the FastAPI server — single-threaded write path by design.
- The seeder runs at startup before the MCP server accepts requests — no contention.
- If concurrent writes become necessary, the Vela fork adds multi-writer support.

---

## Alternatives Considered

### Alternative 1: ChromaDB (Vector Store)

Use ChromaDB as the Context Edge storage backend.

**Rejected because:**
- Cannot model graph relationships (edges, typed connections, multi-hop traversal)
- No Cypher support — loses query language compatibility with Neo4j
- Graph snapshot from Context Server cannot be imported with fidelity
- Would require building a graph engine in application code on top of a vector store
- Schemaless metadata cannot enforce the Context Edge's typed schema

**Would reconsider if:**
- Context Edge pivoted to purely semantic search (embedding similarity) with no structural queries
- ChromaDB added graph/relationship primitives (unlikely — not on their roadmap)

### Alternative 2: ChromaDB + KuzuDB Hybrid

Use KuzuDB for graph traversal and ChromaDB for semantic vector search, side by side.

**Deferred because:**
- Context Edge currently has no semantic search requirement — all queries are deterministic graph traversals
- KuzuDB v0.11.3 includes native HNSW vector search, covering the semantic dimension if needed
- Running two embedded databases increases memory footprint and operational complexity
- The hybrid pattern remains viable as a future enhancement if KuzuDB's vector search proves insufficient for advanced semantic retrieval

### Alternative 3: SQLite + Application-Level Graph

Use SQLite with adjacency tables and implement graph traversal in Python.

**Rejected because:**
- Requires reimplementing graph traversal (BFS/DFS, path finding, recursive CTEs)
- No Cypher — all queries must be hand-written SQL with recursive CTEs or application loops
- Loses the direct mapping to Neo4j's property graph model
- SQLite's recursive CTE performance degrades on deep traversals
- KuzuDB's columnar storage and CSR adjacency lists are purpose-built for this workload

### Alternative 4: Neo4j Embedded (Community Edition)

Run a Neo4j instance inside the DevContainer.

**Rejected because:**
- Neo4j is a server process, not an embeddable library — requires JVM, ~500MB+ footprint
- Cold start time: 10-30 seconds (JVM + database recovery) vs. sub-second for KuzuDB
- Resource overhead inappropriate for an ephemeral sidecar
- Community Edition licensing restrictions for embedded commercial use
- KuzuDB provides Cypher compatibility without the server overhead

### Alternative 5: LadybugDB (KuzuDB Fork)

Adopt LadybugDB directly instead of KuzuDB v0.11.3.

**Deferred because:**
- LadybugDB is only at v0.12.0 — early days for a fork
- API compatibility with KuzuDB is not yet fully validated in our stack
- The fork's long-term sustainability is unproven (single maintainer risk)
- KuzuDB v0.11.3 is stable and sufficient for current requirements
- Migration path is straightforward when/if LadybugDB proves stable — same API, same Cypher

---

## Consequences

### Positive

- **Native graph model** — dependency trees, module relationships, pattern associations, and run history are first-class citizens with typed edges
- **Cypher compatibility with Neo4j** — same query language across edge (Kuzu) and central (Neo4j), minimal translation in snapshot sync
- **Unified graph + vector engine** — if semantic search is needed, HNSW vector index is built in — no second database required
- **Sub-millisecond queries** — columnar storage + CSR adjacency lists deliver 50-300x speedup over Neo4j for the same queries
- **Zero infrastructure** — embedded library, no server process, no port management, no JVM
- **Single-file persistence** — trivial snapshot, volume mount, and stale-mode recovery
- **Schema enforcement** — typed node and edge tables prevent drift between Context Edge and Context Server

### Negative

- **Archived project** — no upstream fixes; must vendor and maintain the binary
- **Single-writer** — one write transaction at a time (mitigated by serialized write path)
- **No built-in Neo4j push** — flush-to-Neo4j requires export + re-import via Python driver (already implemented in `sync/push.py`)
- **Fork uncertainty** — LadybugDB's longevity is unproven as an upgrade path
- **Extension ecosystem frozen** — only the four bundled extensions (vector, FTS, algo, JSON) are available; no new extensions possible without forking

### Neutral

- **Monitor LadybugDB** — track stability and adoption; evaluate migration at v1.0 or if a blocking bug is found in KuzuDB v0.11.3
- **Hybrid option preserved** — if advanced semantic search is needed beyond KuzuDB's HNSW, ChromaDB can be added alongside (not instead of) KuzuDB
- **ContextProvider abstraction** — the interface isolates storage choice; backend swaps are localized to `kuzu_client.py`

---

## Comparison Summary

| Dimension | KuzuDB | ChromaDB | Verdict |
|---|---|---|---|
| **Data model** | Property graph (nodes + typed edges) | Flat collections (docs + embeddings) | KuzuDB — graph data needs a graph model |
| **Query language** | Cypher (Neo4j-compatible) | Python API only | KuzuDB — shared language with central store |
| **Multi-hop traversal** | Native, sub-millisecond | Impossible | KuzuDB — core requirement |
| **Vector search** | Yes (HNSW extension) | Yes (core feature) | Tie — both support it |
| **Neo4j compatibility** | High (same model, migration ext.) | None | KuzuDB — snapshot sync requires it |
| **Embedded mode** | Library, single file | Library, directory | Tie — both embed well |
| **Schema enforcement** | Required (typed tables) | None (schemaless) | KuzuDB — prevents drift |
| **License** | MIT | Apache 2.0 | Tie — both permissive |
| **Project status** | Archived (fork: LadybugDB) | Active | ChromaDB — but wrong category |
| **Memory footprint** | ~50MB binary + data | ~100-200MB Python + data | KuzuDB — lighter |
| **Python async** | Yes (thread-pool, FastAPI-ready) | Yes | Tie |
| **Graph algorithms** | PageRank, Louvain, shortest paths | None | KuzuDB |

---

## References

- [KuzuDB Repository (archived)](https://github.com/kuzudb/kuzu) — MIT License, v0.11.3 final
- [LadybugDB (active fork)](https://github.com/ladybugdb/ladybugdb) — v0.12.0
- [Vela KuzuDB Fork (multi-writer)](https://github.com/vela-engineering/kuzu) — concurrent writes
- [ChromaDB Repository](https://github.com/chroma-core/chroma) — Apache 2.0
- [GAD Context Edge Component](/docs/c4-architecture/components/context-edge-server/README.md)
- [GAD Context Server Component](/docs/c4-architecture/components/context-server/README.md)
- [ADR-0001: OpenCode CLI](/docs/architectural-decision-records/0001-opencode-cli-over-claude-code-cli.md) — shares archived-project mitigation pattern
