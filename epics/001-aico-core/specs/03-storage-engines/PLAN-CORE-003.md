---
id: PLAN-CORE-003
type: plan
aico_version: "0.1.1"
epic: "001-aico-core"
title: "Dual Embedded Storage & AST Ingestion Pipeline"
status: Completed
derives_from: ["SPEC-CORE-003"]
created_at: "2026-10-05T20:00:00Z"
---

# Implementation Plan: Dual Embedded Storage & AST Ingestion Pipeline


---

## 1. Technical Architecture & Decisions

- **Crate**: `crates/aico-storage` (part of the AICO Rust Cargo workspace).
- **Core Dependencies**:
  - `lancedb`: Local-first embedded columnar vector database for semantic search.
  - `kuzu`: Embedded in-process Cypher property graph database for relational and graph queries.
  - `tree-sitter` & language parsers (`tree-sitter-rust`, `tree-sitter-typescript`, `tree-sitter-python`): Deterministic AST extraction.
  - `fastembed` / local ONNX runtime: Local-first text embedding generation (e.g., `all-MiniLM-L6-v2`) without cloud API dependency.
  - `notify`: Cross-platform filesystem event watcher for incremental re-indexing.
- **Storage Layout**: All data stored locally in `.aico/storage/` (project-scoped or root-scoped) with zero background daemon processes.

---

## 2. Task Graph (DAG)

### Phase 1: In-Process Storage Drivers & Concurrency (Unblocked Frontier)

- **TASK-CORE-003-1**: Embedded LanceDB & Kùzu In-Process Connection Managers
  - **Status**: Done
  - **Addresses**: `REQ-001`, `REQ-002`, `REQ-003`, `REQ-009`
  - **Blocked by**: None (Can start immediately)
  - **What it delivers**:
    - Implements `LanceVectorStore` wrapping embedded LanceDB under `.aico/storage/vector/`.
    - Implements `KuzuGraphStore` wrapping embedded Kùzu under `.aico/storage/graph/`.
    - Implements thread-safe connection pooling and concurrent read/write locks (`tokio::sync::RwLock`).
    - Verifies pure in-process operation without background daemon or socket dependencies.
  - **Verification Seam**: `tests/storage_connection_test.rs::test_embedded_databases_init_and_concurrency`

---

### Phase 2: Schema Migrations & Traceability Graph

- **TASK-CORE-003-2**: Graph Schema Initializer, Versioning & Traceability Schema
  - **Status**: Done
  - **Addresses**: `REQ-006`, `REQ-008`
  - **Blocked by**: `TASK-CORE-003-1`
  - **What it delivers**:
    - Defines Kùzu node tables: `Project`, `Epic`, `Spec`, `Requirement`, `Task`, `Symbol`, `File`, `AuditEvent`.
    - Defines Kùzu relationship tables: `BELONGS_TO`, `CONTAINS_SPEC`, `HAS_REQUIREMENT`, `ADDRESSES`, `BLOCKS`, `CALLS`, `DEFINES_SYMBOL`.
    - Implements migration engine verifying `aico_version` frontmatter compatibility.
  - **Verification Seam**: `tests/storage_schema_test.rs::test_kuzu_schema_migration_and_traceability`

---

### Phase 3: Tree-sitter AST Parsing & Symbol Call Graphs

- **TASK-CORE-003-3**: Tree-sitter AST Parser & Symbol Ingestion Pipeline
  - **Status**: Done
  - **Addresses**: `REQ-005`
  - **Blocked by**: `TASK-CORE-003-2`
  - **What it delivers**:
    - Implements `CodeSymbolParser` using Tree-sitter for Rust, TypeScript, and Python.
    - Extracts symbols (functions, structs, traits, interfaces, classes) with line ranges and doc comments.
    - Extracts call graph edges (`caller -> callee`) and import dependencies.
    - Batches symbol and relation writes into Kùzu property graph.
  - **Verification Seam**: `tests/storage_ast_test.rs::test_treesitter_symbol_and_callgraph_ingestion`

---

### Phase 4: Semantic Chunking & Vector Ingestion

- **TASK-CORE-003-4**: Semantic Chunker & Local LanceDB Embedding Pipeline
  - **Status**: Done
  - **Addresses**: `REQ-004`
  - **Blocked by**: `TASK-CORE-003-2`
  - **What it delivers**:
    - Implements `CodeChunker` using AST boundary-aware chunking (preserving function and class scopes).
    - Embeds text and code chunks using in-process `fastembed`.
    - Inserts vectorized records into LanceDB with metadata (`file_path`, `start_line`, `end_line`, `content_hash`).
  - **Verification Seam**: `tests/storage_vector_test.rs::test_semantic_chunking_and_vector_search`

---

### Phase 5: Incremental Watcher & Hybrid Search Engine

- **TASK-CORE-003-5**: Incremental File Watcher & Differential Indexer
  - **Status**: Done
  - **Addresses**: `REQ-010`
  - **Blocked by**: `TASK-CORE-003-3`, `TASK-CORE-003-4`
  - **What it delivers**:
    - Implements `StorageWatcher` using `notify` to listen for file create/modify/delete events.
    - Computes content SHA-256 hashes to skip unchanged files.
    - Updates only modified nodes/edges in Kùzu and vectors in LanceDB incrementally.
  - **Verification Seam**: `tests/storage_watcher_test.rs::test_incremental_file_reindexing`

---

- **TASK-CORE-003-6**: Unified Hybrid Query Engine (Vector + Graph)
  - **Status**: Done
  - **Addresses**: `REQ-007`
  - **Blocked by**: `TASK-CORE-003-3`, `TASK-CORE-003-4`
  - **What it delivers**:
    - Implements `HybridQueryEngine::search(query_text, filter_params)`.
    - Combines top-K vector matches from LanceDB with graph neighborhood traversal (callers, callees, specs) from Kùzu.
    - Returns ranked, deduplicated context packets for agent prompt injection.
  - **Verification Seam**: `tests/storage_hybrid_test.rs::test_hybrid_vector_and_graph_query`

---

## 3. Initial Active Frontier

- `TASK-CORE-003-1`: Embedded LanceDB & Kùzu In-Process Connection Managers (Ready for `/aico-test-scaffolding`).
