---
id: brain-ec-storage
type: brainstorm
aico_skills_version: "0.1.2"
epic: "ec-storage"
title: "Native Storage Architecture: AicoGraphStore, LanceDB Hybrid Search & Multi-Project Federation"
status: "Completed"
derives_from: ["ADR-0001", "ADR-0006", "ADR-0011"]
created_at: "2026-10-24T17:26:16.032Z"
---

# Brainstorm: Native Storage Architecture: AicoGraphStore, LanceDB Hybrid Search & Multi-Project Federation

## 1. Executive Summary & Problem Statement

AICO's core developer promise is deterministic, local-first Spec-Driven Development (SDD) backed by deep architectural intelligence: multi-hop lineage walks (`ADR -> Epic -> Feature -> Spec -> Requirement <- Task <- Code Seam`) and semantic code/specification discovery across multi-repository workspaces without cloud telemetry or external background daemons.

However, the storage subsystem currently suffers from four critical architectural discrepancies:
1. **Misleading Kùzu Legacy Branding & Stubbed Cypher Interface**: `crates/aico-storage/src/graph.rs` is implemented as a pure-Rust property graph (`RwLock<GraphSnapshot>`), yet retains the legacy name `KuzuGraphStore` and is branded as `"Kùzu Graph"` across UI host diagnostics, MCP tools, and documentation, despite containing zero C++ Kùzu bindings.
2. **Synchronous $O(N^2)$ Disk Contention in Graph Mutations**: Every single node or edge upsert in `crates/aico-storage/src/graph.rs` reads and rewrites the entire multi-megabyte `lineage_graph.json` file synchronously to disk, causing severe I/O bottlenecks during batch AST ingestion.
3. **Redundant Per-Request Graph Rescanning**: `GET /api/v1/graph` in `crates/aico-server/src/server.rs` walks the entire specs filesystem on every HTTP call, re-parsing markdown files rather than building once on startup, caching in memory, and incrementally invalidating only modified files via file `mtime` and `sha256` manifest tracking.
4. **Unpersisted In-Memory Vector Search Stub**: `crates/aico-storage/src/vector.rs` currently holds an ephemeral in-memory hash map with a byte-hash stub (`chunker.rs`), leaving vector and hybrid search unintegrated with the real, production-ready `lancedb` crate.

This epic establishes the unified **Native Storage Engine**:
- Rebranding and upgrading the pure-Rust property graph to **`AicoGraphStore`** with dual in-memory indexed adjacency lists (`incoming` and `outgoing`), batched snapshot persistence, and build-once change-invalidated caching (`mtime` / `sha256` manifest) plus AST symbol and call-graph ingestion.
- Integrating the official **`lancedb`** Rust crate (`https://docs.rs/lancedb/latest/lancedb/index.html`) for persistent columnar vector storage, BM25 inverted lexical indexing, and Reciprocal Rank Fusion (RRF) hybrid search across all workspace projects.
- Providing multi-project storage federation, dynamic cache switching upon active project changes, and live storage telemetry exposed through `/api/v1/stats` and WebUI Host Diagnostics (`"AICO Graph: Embedded OK"`, `"LanceDB: Embedded OK"`).

---

## 2. Core Decisions & Trade-Off Analysis

| Architectural Dimension | Option A: C++ Kùzu & In-Memory Vectors | Option B: SQLite + sqlite-vec | Option C: AicoGraphStore + Official LanceDB (Chosen) |
| :--- | :--- | :--- | :--- |
| **Toolchain & Build Friction** | Heavy C++ CMake / MSVC build dependencies; fragile cross-compilation. | Single C dependency, simple build. | **Zero C++ compiler dependencies**: pure-Rust graph + official `lancedb` crate. |
| **Graph Query Performance** | Fast columnar CSR, but high FFI boundary cost. | Recursive CTEs required for multi-hop graph walks; slow at scale. | **Sub-millisecond in-memory adjacency index**: $O(1)$ edge lookups and sub-10ms BFS/DFS lineage walks. |
| **Vector & Lexical Search** | Ephemeral, unpersisted in-memory stub. | Rudimentary vector indexing; no native columnar Lance format. | **Production-grade disk-backed columnar LanceDB**: IVF-PQ vector indexing + BM25 full-text + RRF. |
| **Caching & Invalidation** | Cold filesystem rescan on every request. | SQLite triggers and dirty tables. | **Build-once in-memory cache** with file `mtime` + `sha256` manifest validation and file watcher updates. |
| **Verdict** | **Rejected** (Toolchain fragility & unpersisted vectors) | **Rejected** (Weak graph performance) | **Chosen** (Ratified in [ADR-0011](../../adr/0011-aico-graph-store-and-lancedb-hybrid-storage.md)) |

---

## 3. Epic Decomposition (Child Features)

This epic decomposes into 4 sequential vertical tracer-bullet features:

| Feature Slug | Order | Name | Core Responsibilities |
| :--- | :--- | :--- | :--- |
| `ft-storage-aico-graph-and-incremental-cache` | 1 | AicoGraphStore Property Graph & Incremental Cache | Rename `KuzuGraphStore` $\to$ `AicoGraphStore`, remove Kùzu terminology, in-memory indexed adjacency lists, batched persistence, build-once `mtime`/`sha256` graph cache, AST symbol and call-graph ingestion. |
| `ft-storage-lancedb-hybrid-search` | 2 | LanceDB Persistent Vector & BM25 Hybrid Search Engine | Add `lancedb` crate to `crates/aico-storage`, disk-backed vector tables under `.aico/storage/vector/`, AST chunking, local dense embeddings (`bge-small-en-v1.5`), BM25 lexical indexing, and Reciprocal Rank Fusion (RRF) MCP tool `search_hybrid`. |
| `ft-storage-multi-project-federation-and-telemetry` | 3 | Multi-Project Storage Federation & Telemetry | Multi-project storage federation across `WorkspaceRegistry`, cross-project search scoping, dynamic cache switching on active project change, live storage telemetry in `/api/v1/stats` and WebUI Host Diagnostics (`"AICO Graph: Embedded OK"`, `"LanceDB: Embedded OK"`). |
| `ft-storage-workspace-crawler-and-reindexing` | 4 | Workspace Vector Crawler, Graph Rebuild & Settings Maintenance Controls | Canonicalize project ID to `*-specs` (`aico-specs`), workspace crawler for specs and codebase files, `POST /api/v1/storage/reindex`, `POST /api/v1/storage/rebuild-graph`, startup background indexing, and Settings Modal maintenance buttons. |

---

## 4. Cross-Feature Architectural Invariants

1. **Zero External Daemon Dependencies**: Both `AicoGraphStore` and `LanceDB` run in-process within the `aico` host runtime binary. No standalone background daemons, Docker containers, or JVM processes are permitted.
2. **Deterministic Manifest Invalidation**: Graph cache invalidation evaluates file `mtime` followed by SHA-256 content verification to prevent false cache hits or redundant full rebuilds.
3. **Workspace-Root Storage Topology**: All runtime databases live at `<workspace-root>/.aico/storage/` outside of tracked project Git repositories (`<project>-specs`), with a single unified `LanceVectorStore` at `<workspace-root>/.aico/storage/vector/` (partitioned logically via the `project_id` column) and per-project `AicoGraphStore` snapshots at `<workspace-root>/.aico/storage/graph/<project_id>/lineage_graph.json`. Epic and Feature assets remain versioned inside `<project>-specs`.
4. **Dual-Axis Verification Guarantee**: All storage endpoints and MCP tools must pass unit/integration tests with `@verifies` and `@implements` traceability tags before implementation is marked complete.
