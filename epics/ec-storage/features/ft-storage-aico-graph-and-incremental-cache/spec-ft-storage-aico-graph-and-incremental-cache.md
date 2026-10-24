---
id: spec-ft-storage-aico-graph-and-incremental-cache
type: spec
aico_skills_version: "0.1.2"
epic: "ec-storage"
order: 1
title: "AicoGraphStore Property Graph & Incremental Change-Invalidated Cache"
status: "Completed"
derives_from: ["brain-ec-storage", "ADR-0011"]
created_at: "2026-10-24T17:26:21.143Z"
---

# Feature Specification: AicoGraphStore Property Graph & Incremental Change-Invalidated Cache

## 1. Problem Statement & User Value

AICO's visual lineage canvas and MCP graph tools (`query_lineage_graph`, `find_dangling_requirements`) require sub-10ms graph traversals across all project artifacts (`ADR -> Epic -> Feature -> Spec -> Requirement <- Task <- Code Seam`). However, the existing implementation exhibits three major defects:
1. **Misleading Kùzu Nomenclature**: The graph store is named `KuzuGraphStore` and labeled `"Kùzu Graph"` in UI diagnostics and MCP tools despite being a pure-Rust property graph without C++ Kùzu dependencies.
2. **Synchronous $O(N^2)$ Disk Write Amplification**: Each individual node or edge insertion serializes and rewrites the entire `lineage_graph.json` file to disk synchronously, creating severe I/O bottlenecks.
3. **Cold Filesystem Rescans on Every HTTP Request**: `GET /api/v1/graph` re-walks the entire filesystem and re-parses every markdown file on every page reload, causing unnecessary CPU cycles and latency instead of caching in memory and invalidating only changed files.

This specification defines the upgraded **`AicoGraphStore`**: a pure-Rust property graph featuring dual in-memory indexed adjacency lists, batched snapshot persistence, and a build-once change-invalidated cache evaluated via file `mtime` and SHA-256 manifests.

---

## In-Scope

- Renaming `KuzuGraphStore` $\to$ `AicoGraphStore` across `crates/aico-storage`, `crates/aico-mcp`, `crates/aico-server`, and WebUI.
- Updating WebUI Host Diagnostics on `DashboardView.tsx` to display `"AICO Graph: Embedded OK"`.
- Dual in-memory indexed adjacency maps (`incoming` and `outgoing`) providing $O(1)$ neighbor lookup and sub-millisecond traversal.
- Batched asynchronous or explicit snapshot flushes to `.aico/storage/graph/lineage_graph.json`.
- Build-once in-memory graph caching with file `mtime` and SHA-256 manifest invalidation on `GET /api/v1/graph`.
- Ingestion of Tree-sitter extracted AST code symbols and call-graph edges into `AicoGraphStore`.

---

## 2. User Stories

1. **US-01** (Priority: P1): As a developer opening the Home Dashboard or Epic Hub, I want the lineage graph to load in under 10ms from an in-memory cache without rescanning the disk on every page reload.
   - *Independent Test*: Query `GET /api/v1/graph` twice consecutively; verify the second request serves from memory without filesystem I/O and latency is under 5ms.
2. **US-02** (Priority: P1): As a developer editing an SDD specification or code file, I want only the modified file re-indexed in the graph, so that saving a file updates the graph incrementally in under 20ms.
   - *Independent Test*: Touch a single specification file and call `GET /api/v1/graph`; verify only that file is re-parsed and its updated requirements appear in the graph.
3. **US-03** (Priority: P1): As a developer inspecting system diagnostics in the UI or using MCP tools, I want clear and accurate naming ("AICO Graph") rather than legacy "Kùzu" mentions.
   - *Independent Test*: Mount `DashboardView.tsx` and query MCP `tools/list`; verify host diagnostics and tool descriptions display "AICO Graph".
4. **US-04** (Priority: P2): As an agent executing deep-dive queries, I want multi-hop traversals across 10,000+ nodes to complete in under 5ms using indexed in-memory adjacency lists.
   - *Independent Test*: Execute a 4-hop BFS traversal on a 1,000-node graph fixture; verify traversal completes in under 2ms.

---

## 3. Functional Requirements (EARS)

### Ubiquitous Requirements
- **REQ-001**: The `AicoGraphStore` shall replace `KuzuGraphStore` across `crates/aico-storage`, `crates/aico-mcp`, `crates/aico-server`, and WebUI components.
- **REQ-002**: The `AicoGraphStore` shall maintain an in-memory property graph with indexed incoming and outgoing adjacency maps (`HashMap<String, HashSet<EdgeId>>`), providing $O(1)$ neighbor lookup and sub-millisecond BFS/DFS traversal.
- **REQ-003**: The `AicoGraphStore` shall evaluate Cypher-style `MATCH (n)-[r]->(m)` expressions and multi-hop lineage queries (`query_lineage`).
- **REQ-004**: The `AicoGraphStore` shall persist snapshots to `.aico/storage/graph/lineage_graph.json` upon explicit flush or batch completion, avoiding per-upsert synchronous disk writes.
- **REQ-005**: The WebUI Host Diagnostics on `DashboardView.tsx` shall display `"AICO Graph: Embedded OK"` instead of `"Kùzu Graph: Embedded OK"`.

### Event-Driven Requirements
- **REQ-006**: When `GET /api/v1/graph` is received or on server boot, the `LineageCacheManager` shall check the graph manifest against filesystem `mtime` and SHA-256 hashes.
- **REQ-007**: When all file timestamps and hashes in the manifest match the filesystem, the `LineageCacheManager` shall return the in-memory cached graph immediately without re-parsing markdown files.
- **REQ-008**: When a specification, brainstorm, ADR, or task plan file has a modified `mtime` or altered SHA-256 hash, the `LineageCacheManager` shall re-parse only the modified file, update the graph nodes and edges, and refresh the manifest.
- **REQ-009**: When `CodeSymbolParser` extracts AST symbols and call sites from codebase files, `AicoGraphStore` shall upsert `CodeSymbol` nodes and create `[:CALLS]`, `[:IMPORTS]`, `[:IMPLEMENTS]`, and `[:VERIFIES]` edges.

### State-Driven Requirements
- **REQ-010**: While multi-hop lineage queries are executing, `AicoGraphStore` shall traverse up to 6 hops in under 10 milliseconds for graphs containing up to 50,000 nodes.
- **REQ-011**: While batch insertions are in progress, `AicoGraphStore` shall hold in-memory write locks and defer disk persistence until the batch operation completes.

### Unwanted Behavior & Error Handling
- **REQ-012**: If the cached `lineage_graph.json` or manifest file is corrupted or unparseable, then `AicoGraphStore` shall log a warning, rebuild the full graph from the markdown files, and rewrite the cache file cleanly.

---

## 4. Designated Public Test Seams

### Primary Seams:
1. **AicoGraphStore Adjacency & Traversal Seam (`crates/aico-storage/tests/storage_graph_test.rs`)**:
   - Signature: `AicoGraphStore::query_lineage(&self, start_id: &str, hops: usize) -> Result<GraphSnapshot, GraphError>`
   - Verifies: $O(1)$ adjacency lookups, multi-hop BFS traversal, and batch persistence (`REQ-001`, `REQ-002`, `REQ-003`, `REQ-004`).
2. **Lineage Cache & Manifest Invalidation Seam (`crates/aico-server/tests/server_test.rs`)**:
   - Signature: `GET /api/v1/graph`
   - Verifies: Build-once caching, timestamp/hash evaluation, and incremental re-indexing (`REQ-006`, `REQ-007`, `REQ-008`).
3. **AST Symbol & Edge Ingestion Seam (`crates/aico-storage/tests/storage_ast_test.rs`)**:
   - Signature: `AicoGraphStore::ingest_code_symbols(&self, symbols: &[CodeSymbol]) -> Result<(), StorageError>`
   - Verifies: `CodeSymbol` nodes and call/verification edges (`REQ-009`).
4. **UI Host Diagnostics Seam (`frontend/tests/DashboardView.test.tsx`)**:
   - Signature: `<DashboardView ... />`
   - Verifies: Display of `"AICO Graph: Embedded OK"` (`REQ-005`).

---

## Out-of-Scope

- Distributed cluster graph databases (Neo4j, Memgraph).
- Arbitrary Cypher subqueries with complex aggregation expressions (`WITH`, `UNWIND`, `OPTIONAL MATCH`).
- External file synchronization services.

---

## 5. Open Clarifications

None. Ratified in [ADR-0011](../../../../adr/0011-aico-graph-store-and-lancedb-hybrid-storage.md).
