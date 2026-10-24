---
id: adr-0011
type: adr
aico_skills_version: "0.1.2"
title: "AicoGraphStore Property Graph & LanceDB Hybrid Storage Architecture"
status: "Completed"
derives_from: ["ADR-0001", "ADR-0006"]
created_at: "2026-10-24T17:26:12.349Z"
---

# ADR-0011: AicoGraphStore Property Graph & LanceDB Hybrid Storage Architecture

## Context & Problem Statement

Early architecture definitions (ADR-0001, ADR-0004, ADR-0006, and `spec-ft-core-local-storage`) envisioned embedding two database engines into the local AICO host runtime:
1. **Kùzu**: A C++ columnar property graph database engine accessed via FFI/C-bindings.
2. **LanceDB**: An embedded columnar vector engine for semantic code and specification retrieval.

In practice, integrating native C++ Kùzu introduced significant packaging overhead, fragile C++ build toolchains on Windows/macOS/Linux, and linker friction. In response, `crates/aico-storage/src/graph.rs` was implemented as a pure-Rust property graph (`RwLock<GraphSnapshot>`), while retaining the legacy name `KuzuGraphStore` and superficial openCypher query stubs. However:
- The graph store was still branded as `"Kùzu Graph"` across UI diagnostics, MCP tool definitions, and documentation.
- The pure-Rust graph rewrote the entire multi-megabyte `lineage_graph.json` snapshot to disk on every single node/edge upsert, causing $O(N^2)$ I/O contention during AST ingestion.
- The lineage graph was completely rescanned from markdown files on every HTTP request (`GET /api/v1/graph`) rather than being built once, cached in memory, and incrementally updated on file changes (`mtime` / `sha256` manifest).
- Meanwhile, vector storage in `crates/aico-storage/src/vector.rs` remained an unpersisted in-memory hash map stub rather than utilizing the real, official `lancedb` crate.

AICO requires a coherent, robust, production-grade storage architecture that scales across all projects without external C++ toolchains.

## Decision Drivers

- **Zero External C++ Toolchain Dependencies**: Maintain single-binary, pure-Rust compilation without requiring MSVC C++ CMake toolchains or dynamic C library bindings for graph processing.
- **Sub-Millisecond Graph Traversals**: High-performance in-memory adjacency list traversals for multi-hop lineage (`ADR -> Epic -> Feature -> Spec -> Plan -> Task -> Code Seams`).
- **Build-Once & Change-Invalidated Graph Cache**: Lineage graph constructed on startup or first request, cached in memory, and incrementally refreshed only when spec/code files change via `mtime` and `sha256` manifest tracking.
- **Scalable Persistent Vector & Hybrid Search**: Production-grade disk-backed columnar storage with vector indexing (IVF-PQ) and BM25 inverted full-text search, capable of scaling to hundreds of thousands of code and spec chunks across all projects.
- **Architectural & Semantic Purity**: Eliminate misleading "Kùzu" references across code, UI diagnostics, MCP tools, and specification artifacts.

## Considered Options

1. **Option A (Chosen)**:
   - **Graph**: Adopt **`AicoGraphStore`** as the canonical pure-Rust property graph. Upgrade it with indexed in-memory adjacency maps (`incoming` and `outgoing`), atomic snapshot persistence, and build-once change-invalidated caching based on file `mtime` and `sha256` manifests. Completely deprecate and remove all "Kùzu" branding and artifacts.
   - **Vector & Hybrid Search**: Adopt the official **`lancedb`** Rust crate (`https://docs.rs/lancedb/latest/lancedb/index.html`) as an embedded disk-backed vector database in `crates/aico-storage`. Store vector embeddings and BM25 lexical tokens under `.aico/storage/vector/` with Reciprocal Rank Fusion (RRF) scoring for multi-project hybrid discovery.
2. **Option B**:
   - Link the official C++ Kùzu engine via `kuzu` crate and maintain pure-Rust in-memory vector indexing.
   - *Rejected*: Compiling C++ Kùzu brings heavy CMake/MSVC toolchain prerequisites, complex cross-compilation hurdles, and no substantial performance benefit over pure Rust for project graph sizes (~1,000–50,000 nodes).
3. **Option C**:
   - Replace both graph and vector engines with SQLite (`rusqlite` + `sqlite-vec`).
   - *Rejected*: Relational tables require recursive CTEs for multi-hop graph walks, and `sqlite-vec` lacks native columnar batch filtering, IVF-PQ indexing, and native Lance file format compatibility for fast multi-modal storage.

## Decision Outcome

Chosen option: **Option A**.

### 1. Pure-Rust `AicoGraphStore` (Per-Project at Workspace Root)
- Rename `KuzuGraphStore` $\to$ `AicoGraphStore` across `crates/aico-storage`, `crates/aico-mcp`, `crates/aico-server`, and the frontend (`VisualLineageCanvas`, Host Diagnostics).
- Maintain node and edge indexing via dual adjacency maps (`HashMap<String, HashSet<EdgeId>>` for incoming and outgoing edges), ensuring $O(1)$ neighbor lookups and sub-millisecond BFS/DFS traversals.
- Persist per-project snapshots at `<workspace-root>/.aico/storage/graph/<project_id>/lineage_graph.json` asynchronously or on batch completion, eliminating $O(N^2)$ synchronous disk rewrites and keeping `<project>-specs` Git repositories completely clean of runtime database files.
- Implement build-once caching: on startup or initial request, `AicoGraphStore` builds the lineage graph from the filesystem, records file `mtime` and `sha256` hashes in a manifest, and only re-indexes modified files on subsequent requests or file watcher events.

### 2. Single Unified Workspace-Wide `LanceDB` Vector & Hybrid Search
- Add `lancedb` to `crates/aico-storage/Cargo.toml`.
- Maintain a **single unified LanceDB instance** at `<workspace-root>/.aico/storage/vector/` shared across all registered workspace projects, with a `project_id` column in the `chunks` Arrow schema.
- Support both single-project searches (filtered by `project_id`) and workspace-wide searches (`scope: "all"`) against the single columnar table without multi-database fan-out.
- Epic and Feature assets remain stored inside `<project>-specs/epics/<epic>/assets/` and `<project>-specs/.../features/<feature>/assets/` so they are versioned and pushed to Git alongside their specifications.
- Chunk code symbols and SDD requirements via `chunker.rs`, generate 384-dimensional dense embeddings (`bge-small-en-v1.5`), and index lexical keywords for BM25 search.
- Provide a unified `search_hybrid` interface that calculates Reciprocal Rank Fusion (RRF) scores across dense vector distance and lexical BM25 rank.

### 3. Artifact & UI Alignment
- Update all historical specifications, ADRs, and `GLOSSARY.md` to reference `AicoGraphStore` and embedded `LanceDB`.
- Update frontend diagnostics in `DashboardView.tsx` from `"Kùzu Graph: Embedded OK"` to `"AICO Graph: Embedded OK"`.
- Update MCP tool `query_lineage_graph` documentation to reflect `AicoGraphStore`.

### Positive Consequences
- True zero-C++ pure-Rust compilation on Windows, Linux, and macOS.
- Instantaneous graph traversals and sub-10ms UI lineage rendering with zero redundant disk rescans.
- Persistent, scalable vector and hybrid search backed by the high-performance Lance columnar format.
- Clear, consistent domain terminology across all layers of the codebase and documentation.

### Negative Consequences / Trade-offs
- Custom Cypher query parser is limited to supported lineage expressions (`MATCH (n)-[r]->(m) WHERE ... RETURN ...`); complex openCypher subqueries must use specialized BFS/DFS lineage queries instead.
