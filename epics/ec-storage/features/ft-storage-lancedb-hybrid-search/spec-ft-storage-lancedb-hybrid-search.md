---
id: spec-ft-storage-lancedb-hybrid-search
type: spec
aico_skills_version: "0.1.2"
epic: "ec-storage"
order: 2
title: "LanceDB Persistent Vector & BM25 Hybrid Search Engine"
status: "Completed"
derives_from: ["brain-ec-storage", "ADR-0011"]
created_at: "2026-10-24T17:26:21.517Z"
---

# Feature Specification: LanceDB Persistent Vector & BM25 Hybrid Search Engine

## 1. Problem Statement & User Value

AI agents require rapid, precise retrieval of code symbols, architectural specifications, ADRs, and documentation chunks across multi-repository workspaces. Pure lexical string grep misses semantic synonyms and intent, while pure vector search often misses exact identifiers, function signatures, or requirement IDs (e.g. `REQ-042`).

Currently, `crates/aico-storage/src/vector.rs` is an unpersisted in-memory `HashMap<String, VectorRecord>` with a byte-hash stub that resets on process termination and lacks true disk-backed columnar indexing.

This specification integrates the official **`lancedb`** Rust crate into `crates/aico-storage`:
1. Establishing persistent, disk-backed columnar vector tables under `.aico/storage/vector/`.
2. Chunking code symbols and markdown documents with semantic AST boundaries.
3. Generating 384-dimensional dense embeddings (`bge-small-en-v1.5`) alongside inverted lexical BM25 tokens.
4. Executing hybrid queries that fuse vector cosine similarity and BM25 lexical rank using Reciprocal Rank Fusion (RRF), surfaced through the MCP tool `search_hybrid`.

---

## In-Scope

- Adding the official `lancedb` crate to `crates/aico-storage/Cargo.toml`.
- Initializing an embedded disk-backed LanceDB connection at `.aico/storage/vector/`.
- Arrow schema definition: `(id, content, file_path, start_line, end_line, language, vector, doc_type, project_id)`.
- Semantic AST chunking via `CodeChunker` for Rust, TypeScript, Python, and Markdown.
- Dense vector generation (384-dimensional embeddings) and lexical keyword extraction.
- Reciprocal Rank Fusion (RRF) algorithm combining vector proximity and BM25 score.
- Incremental differential indexing via SHA-256 chunk hashing.
- Exposing hybrid search through MCP tool `search_hybrid` and API endpoint `POST /api/v1/search/hybrid`.

---

## 2. User Stories

1. **US-01** (Priority: P1): As an agent performing a code search, I want hybrid search results fusing keyword hits and semantic meaning, so that I find relevant functions even when wording differs.
   - *Independent Test*: Query `search_hybrid` for "user authentication and session token verification" and verify both semantic matches and functions tagged with `auth_token` return in the top 5 results.
2. **US-02** (Priority: P1): As a developer restarting the AICO server, I want vector indexes to persist on disk, so that the entire codebase does not require re-indexing on every boot.
   - *Independent Test*: Ingest 50 code chunks into LanceDB, restart the server process, and query `search_hybrid`; verify results return immediately from the persistent table.
3. **US-03** (Priority: P1): As an agent calling the MCP tool `search_hybrid`, I want exact requirement IDs and function symbols to match reliably with high rank.
   - *Independent Test*: Call MCP tool `search_hybrid` with query `"REQ-002"` and verify the document declaring `REQ-002` appears as rank 1 via BM25 boost.
4. **US-04** (Priority: P2): As a developer saving a file, I want only changed functions to be re-embedded, so that saving a file does not trigger costly CPU re-embedding of unchanged chunks.
   - *Independent Test*: Save a file with 10 functions where 1 was modified; verify only 1 chunk is updated in the LanceDB table.

---

## 3. Functional Requirements (EARS)

### Ubiquitous Requirements
- **REQ-001**: The `LanceVectorStore` shall manage an embedded columnar table using the official `lancedb` crate persisted on disk at `<specs-root>/.aico/storage/vector/`.
- **REQ-002**: The `LanceVectorStore` shall define an Apache Arrow table schema containing fields `id` (string), `content` (string), `file_path` (string), `start_line` (uint32), `end_line` (uint32), `language` (string), `vector` (fixed-size 384-float list), `doc_type` (string), and `project_id` (string).
- **REQ-003**: The `EmbeddingPipeline` shall produce 384-dimensional dense vectors using a local int8-quantized `bge-small-en-v1.5` model or deterministic fallback vectorizer executing locally on CPU.
- **REQ-004**: The `HybridQueryEngine` shall calculate Reciprocal Rank Fusion (RRF) scores combining dense vector cosine distance rank and BM25 lexical rank according to the formula:
  $$RRF(d) = \frac{w_{vec}}{k + r_{vec}(d)} + \frac{w_{lex}}{k + r_{lex}(d)}$$
  where $k=60$, $w_{vec}=0.6$, and $w_{lex}=0.4$.

### Event-Driven Requirements
- **REQ-005**: When `DifferentialIndexer` detects a modified or newly created source file, the system shall chunk the file by AST scope boundaries, compute SHA-256 hashes per chunk, and upsert only new or changed chunks into LanceDB.
- **REQ-006**: When a source file is deleted, the `DifferentialIndexer` shall delete all associated chunks from the LanceDB table where `file_path` matches.
- **REQ-007**: When an agent invokes the MCP tool `search_hybrid`, the MCP server shall query `HybridQueryEngine` and return the top-K deduplicated results formatted with file paths, line ranges, and relevance scores.

### State-Driven Requirements
- **REQ-008**: While the LanceDB table contains over 1,000 vectors, the `LanceVectorStore` shall maintain an IVF-PQ approximate nearest neighbor index for sub-15ms vector retrieval.
- **REQ-009**: While executing hybrid queries, the `HybridQueryEngine` shall complete queries across up to 50,000 indexed chunks in under 20 milliseconds.

### Unwanted Behavior & Error Handling
- **REQ-010**: If the disk-backed LanceDB table directory does not exist on startup, then `LanceVectorStore` shall create the directory recursively and initialize an empty table with the canonical Arrow schema.
- **REQ-011**: If an embedding generation fails for an individual chunk due to encoding anomalies, then the `EmbeddingPipeline` shall log a warning, generate a fallback vector, and proceed without crashing the indexer.

---

## 4. Designated Public Test Seams

### Primary Seams:
1. **LanceDB Connection & Arrow Table Seam (`crates/aico-storage/tests/storage_vector_test.rs`)**:
   - Signature: `LanceVectorStore::open(path: &Path) -> Result<LanceVectorStore, StorageError>`
   - Verifies: Arrow schema creation, disk persistence, and vector insertion (`REQ-001`, `REQ-002`, `REQ-010`).
2. **Dense Vector & Embedding Seam (`crates/aico-storage/tests/storage_vector_test.rs`)**:
   - Signature: `EmbeddingPipeline::embed_text(text: &str) -> Result<Vec<f32>, StorageError>`
   - Verifies: 384-dimensional vector output and deterministic fallback (`REQ-003`, `REQ-011`).
3. **Hybrid Search & RRF Ranking Seam (`crates/aico-storage/tests/storage_hybrid_test.rs`)**:
   - Signature: `HybridQueryEngine::search(query: &str, limit: usize) -> Result<Vec<SearchHit>, StorageError>`
   - Verifies: Combined dense vector and BM25 RRF scoring (`REQ-004`, `REQ-008`, `REQ-009`).
4. **Differential Indexing Seam (`crates/aico-storage/tests/storage_watcher_test.rs`)**:
   - Signature: `DifferentialIndexer::index_file(path: &Path, content: &str) -> Result<IndexReport, StorageError>`
   - Verifies: SHA-256 chunk deduplication and selective upsert (`REQ-005`, `REQ-006`).
5. **MCP Hybrid Search Tool Seam (`crates/aico-mcp/tests/mcp_graph_hybrid_test.rs`)**:
   - Signature: MCP Tool `search_hybrid`
   - Verifies: Typed JSON-RPC parameters, execution against LanceDB, and formatted output (`REQ-007`).

---

## Out-of-Scope

- Cloud-hosted remote vector databases (Pinecone, Qdrant, Milvus).
- Proprietary remote embedding APIs (OpenAI `text-embedding-ada-002` as a mandatory dependency).
- Multi-GPU distributed vector clustering.

---

## 5. Open Clarifications

None. Ratified in [ADR-0011](../../../../adr/0011-aico-graph-store-and-lancedb-hybrid-storage.md).
