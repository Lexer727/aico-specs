---
id: plan-ft-storage-lancedb-hybrid-search
type: plan
aico_skills_version: "0.1.2"
epic: "ec-storage"
order: 2
title: "LanceDB Persistent Vector & BM25 Hybrid Search Engine Implementation Plan"
status: "Completed"
repositories: ["aico", "aico-specs"]
derives_from: ["spec-ft-storage-lancedb-hybrid-search"]
created_at: "2026-10-24T17:26:21.517Z"
---

# Implementation Plan: LanceDB Persistent Vector & BM25 Hybrid Search Engine

## 1. Technical Architecture & Decisions

- **Crate**: `crates/aico-storage` using the official `lancedb` crate (`cargo add lancedb`).
- **Disk Path**: Persisted as a single unified workspace-wide store at `<workspace-root>/.aico/storage/vector/`.
- **Apache Arrow Table**: Defined using `arrow-schema` with schema:
  - `id`: Utf8 (SHA-256 chunk hash)
  - `content`: Utf8
  - `file_path`: Utf8
  - `start_line`: UInt32
  - `end_line`: UInt32
  - `language`: Utf8
  - `vector`: FixedSizeList(384, Float32)
  - `doc_type`: Utf8 (`"code"`, `"spec"`, `"adr"`, `"doc"`)
  - `project_id`: Utf8
- **Embedding Generation**: In-process `fastembed` / local ONNX runtime with int8 quantized `bge-small-en-v1.5` generating 384-float vectors, with deterministic scalar fallback.
- **Lexical Indexing & BM25**: Extracted keyword tokens scored with Okapi BM25.
- **Hybrid Fusion**: Reciprocal Rank Fusion (RRF) with $k=60$, combining vector nearest-neighbors and BM25 rank.
- **MCP Exposure**: Wired into `aico-mcp` via `McpServer::search_hybrid`.

---

## 2. Task Graph (DAG)

### Phase 1: LanceDB Crate Integration & Columnar Arrow Table

- **TASK-STORAGE-002-1**: Integrate `lancedb` Crate & Implement Persistent Disk-Backed `LanceVectorStore`
  - **Status**: Done
  - **Addresses**: `REQ-001`, `REQ-002`, `REQ-010`
  - **Blocked by**: None (Can start immediately)
  - **What it delivers**:
    - Adds `lancedb` (and compatible `arrow` dependencies) to `crates/aico-storage/Cargo.toml`.
    - Implements `LanceVectorStore::open(&Path)` connecting to embedded LanceDB on disk.
    - Creates or opens the `chunks` Arrow table with canonical schema.
    - Provides `upsert_records` and `delete_by_path` methods.
  - **Verification Seam**: `crates/aico-storage/tests/storage_vector_test.rs::test_lancedb_disk_table_creation_and_upsert`

---

### Phase 2: Embedding Pipeline & Lexical Tokenization

- **TASK-STORAGE-002-2**: Implement 384-Dim Local Embedding Pipeline & BM25 Tokenizer
  - **Status**: Done
  - **Addresses**: `REQ-003`, `REQ-011`
  - **Blocked by**: `TASK-STORAGE-002-1`
  - **What it delivers**:
    - Implements `EmbeddingPipeline` producing 384-dimensional dense vectors.
    - Implements lexical tokenizer extracting code identifiers and markdown keywords for BM25.
    - Provides deterministic fallback vectorization when local ONNX weights are not present.
  - **Verification Seam**: `crates/aico-storage/tests/storage_vector_test.rs::test_embedding_pipeline_and_fallback`

---

### Phase 3: Reciprocal Rank Fusion (RRF) Hybrid Query Engine

- **TASK-STORAGE-002-3**: Implement `HybridQueryEngine` with Vector & BM25 Reciprocal Rank Fusion
  - **Status**: Done
  - **Addresses**: `REQ-004`, `REQ-008`, `REQ-009`
  - **Blocked by**: `TASK-STORAGE-002-2`
  - **What it delivers**:
    - Implements `HybridQueryEngine::search` executing vector approximate nearest neighbor query in LanceDB and BM25 lexical match.
    - Calculates reciprocal rank fusion (RRF) score $RRF(d) = \frac{0.6}{60 + r_{vec}} + \frac{0.4}{60 + r_{lex}}$.
    - Deduplicates results and returns ranked `Vec<SearchHit>`.
  - **Verification Seam**: `crates/aico-storage/tests/storage_hybrid_test.rs::test_hybrid_rrf_scoring_and_ranking`

---

### Phase 4: Differential Indexer & File Ingestion

- **TASK-STORAGE-002-4**: Implement `DifferentialIndexer` with SHA-256 Chunk Hashing
  - **Status**: Done
  - **Addresses**: `REQ-005`, `REQ-006`
  - **Blocked by**: `TASK-STORAGE-002-3`
  - **What it delivers**:
    - Implements AST scope-aware chunking in `CodeChunker`.
    - Hashes each chunk with SHA-256; only re-embeds and updates chunks whose hash has changed.
    - Removes deleted file records on file delete events.
  - **Verification Seam**: `crates/aico-storage/tests/storage_watcher_test.rs::test_differential_chunk_indexing`

---

### Phase 5: MCP Tool & Server Endpoint Integration

- **TASK-STORAGE-002-5**: Wire LanceDB Hybrid Search into MCP `search_hybrid` and Server API
  - **Status**: Done
  - **Addresses**: `REQ-007`
  - **Blocked by**: `TASK-STORAGE-002-4`
  - **What it delivers**:
    - Connects `HybridQueryEngine` into `crates/aico-mcp/src/graph_tools.rs` for `search_hybrid`.
    - Exposes `POST /api/v1/search/hybrid` on `crates/aico-server`.
    - Verifies JSON-RPC tool calls return structured, ranked context packets.
  - **Verification Seam**: `crates/aico-mcp/tests/mcp_graph_hybrid_test.rs::test_mcp_hybrid_search_tool`

---

## 3. Amendment Log

- **2026-10-24T17:26:21Z**: Initial task graph established under `ec-storage`.
