---
id: plan-ft-storage-aico-graph-and-incremental-cache
type: plan
aico_skills_version: "0.1.2"
epic: "ec-storage"
order: 1
title: "AicoGraphStore Property Graph & Incremental Change-Invalidated Cache Implementation Plan"
status: "Completed"
repositories: ["aico", "aico-specs"]
derives_from: ["spec-ft-storage-aico-graph-and-incremental-cache"]
created_at: "2026-10-24T17:26:21.143Z"
---

# Implementation Plan: AicoGraphStore Property Graph & Incremental Change-Invalidated Cache

## 1. Technical Architecture & Decisions

- **Graph Storage Engine**: Pure-Rust property graph in `crates/aico-storage/src/graph.rs` renamed from `KuzuGraphStore` to `AicoGraphStore`.
- **Adjacency Optimization**: Replace linear edge searches with dual adjacency maps:
  - `incoming: HashMap<String, HashSet<String>>` (mapping target node ID to set of edge IDs).
  - `outgoing: HashMap<String, HashSet<String>>` (mapping source node ID to set of edge IDs).
- **Snapshot Persistence**: Asynchronous/batched JSON serialization to `.aico/storage/graph/lineage_graph.json` with in-memory dirty flags to eliminate $O(N^2)$ single-upsert disk writes.
- **Cache Invalidation & Manifest**: In `crates/aico-server/src/server.rs`, introduce `GraphCacheManager` tracking a file manifest (`path -> (mtime, sha256, node_ids, edge_ids)`). If no files have changed, `GET /api/v1/graph` serves the cached graph instantly. If files change, only modified files are re-parsed.
- **AST Integration**: Wire `CodeSymbolParser` (`crates/aico-storage/src/parser.rs`) into `AicoGraphStore` to ingest functions, structs, calls, and in-band `@implements` / `@verifies` annotations.
- **UI & Telemetry**: Update `DashboardView.tsx` Host Diagnostics card to render `"AICO Graph: Embedded OK"`.

---

## 2. Task Graph (DAG)

### Phase 1: Renaming & Kùzu Nomenclature Purge

- **TASK-STORAGE-001-1**: Rename `KuzuGraphStore` to `AicoGraphStore` & Purge Legacy Kùzu Mentions
  - **Status**: Done
  - **Addresses**: `REQ-001`, `REQ-005`
  - **Blocked by**: None (Can start immediately)
  - **What it delivers**:
    - Renames struct `KuzuGraphStore` $\to$ `AicoGraphStore` in `crates/aico-storage/src/graph.rs`, `lib.rs`, `manager.rs`, `schema.rs`.
    - Updates MCP tool handlers in `crates/aico-mcp/src/graph_tools.rs` and `server.rs` to refer to `AicoGraphStore`.
    - Updates `crates/aico-server/src/server.rs` to use `AicoGraphStore`.
    - Updates `frontend/src/features/dashboard/DashboardView.tsx` host diagnostics to render `"AICO Graph: Embedded OK"`.
  - **Verification Seam**: `crates/aico-storage/tests/storage_graph_test.rs::test_aico_graph_store_naming_and_init`, `frontend/tests/DashboardView.test.tsx`

---

### Phase 2: Dual Adjacency Lists & Batched Persistence

- **TASK-STORAGE-001-2**: Implement Dual In-Memory Adjacency Indexing & Batched Persistence
  - **Status**: Done
  - **Addresses**: `REQ-002`, `REQ-003`, `REQ-004`, `REQ-010`, `REQ-011`, `REQ-012`
  - **Blocked by**: `TASK-STORAGE-001-1`
  - **What it delivers**:
    - Implements `incoming` and `outgoing` adjacency maps in `GraphSnapshot`.
    - Upgrades `query_lineage` to perform fast BFS/DFS traversals using the indexed adjacency sets without linear scanning.
    - Implements deferred/batched `flush()` method so individual insertions do not rewrite the entire JSON file.
    - Handles corrupted JSON files gracefully by initializing a clean graph and rebuilding.
  - **Verification Seam**: `crates/aico-storage/tests/storage_graph_test.rs::test_indexed_adjacency_and_fast_traversal`

---

### Phase 3: Manifest-Driven Build-Once & Change-Invalidated Cache

- **TASK-STORAGE-001-3**: Implement Graph Manifest & Incremental Invalidation on `GET /api/v1/graph`
  - **Status**: Done
  - **Addresses**: `REQ-006`, `REQ-007`, `REQ-008`
  - **Blocked by**: `TASK-STORAGE-001-2`
  - **What it delivers**:
    - Implements `GraphCacheManager` maintaining file `mtime` and SHA-256 hashes for all markdown files in the specs repository.
    - Evaluates the manifest on startup and during `GET /api/v1/graph`: returns cached graph instantly if unchanged.
    - Incrementally removes old nodes/edges and re-parses only modified files when changes occur.
  - **Verification Seam**: `crates/aico-server/tests/server_test.rs::test_graph_cache_manifest_and_incremental_invalidation`

---

### Phase 4: AST Code Symbol & Seam Edge Ingestion

- **TASK-STORAGE-001-4**: Wire `CodeSymbolParser` AST Ingestion into `AicoGraphStore`
  - **Status**: Done
  - **Addresses**: `REQ-009`
  - **Blocked by**: `TASK-STORAGE-001-3`
  - **What it delivers**:
    - Connects `CodeSymbolParser` with `AicoGraphStore::ingest_symbols`.
    - Inserts `CodeSymbol` nodes and extracts caller/callee edges, `@implements`, and `@verifies` linkages.
  - **Verification Seam**: `crates/aico-storage/tests/storage_ast_test.rs::test_symbol_ingestion_and_graph_linkage`

---

### Phase 5: Regression & Integration Verification

- **TASK-STORAGE-001-5**: End-to-End Lineage & UI Verification
  - **Status**: Done
  - **Addresses**: `REQ-001`, `REQ-002`, `REQ-005`, `REQ-006`
  - **Blocked by**: `TASK-STORAGE-001-4`
  - **What it delivers**:
    - Verifies all backend storage tests and server graph tests pass green.
    - Verifies Vitest tests for `DashboardView.test.tsx` and `VisualLineageCanvas.test.tsx` pass green.
  - **Verification Seam**: `crates/aico-server/tests/server_test.rs`, `frontend/tests/DashboardView.test.tsx`

---

## 3. Amendment Log

- **2026-10-24T17:26:21Z**: Initial task graph established under `ec-storage`.
