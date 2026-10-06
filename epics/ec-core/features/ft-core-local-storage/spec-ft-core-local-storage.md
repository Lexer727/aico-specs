---
id: spec-ft-core-local-storage
type: spec
aico_skills_version: "0.1.2"
epic: "ec-core"
order: 3
title: "Dual Embedded Storage & AST Ingestion Pipeline"
status: "Completed"
derives_from: ["ADR-0001", "ADR-0006"]
created_at: "2026-10-05T19:15:00Z"
---

# Feature Specification: Dual Embedded Storage & AST Ingestion Pipeline


---

## 1. Problem Statement & User Value

Traditional AI coding assistants rely on shallow string grep or send entire source files to external vector databases, causing high latency, high cloud costs, and index drift. Furthermore, understanding architectural lineage requires multi-hop graph walks (tracing functions back through specs to brainstorms) that vector stores cannot perform.

This specification defines AICO's dual embedded storage topology: an in-process **LanceDB** engine for hybrid lexical/dense search and an in-process **Kùzu** engine for openCypher graph traversals, fed by an incremental background **Tree-sitter AST watcher** that updates indexes on file save without requiring background host daemons.

---

## In-Scope
- In-process Kùzu embedded graph database for artifact lineage and AST dependency relationships.
- In-process LanceDB vector database with BM25 + dense vector hybrid search and RRF ranking.
- AST symbol chunking and automated file change ingestion via Tree-sitter.
- Zero background daemon processes required for local storage execution.

---

## 2. User Stories

1. **US-01** (Priority: P1): As a developer, I want all vector embeddings generated locally on my CPU using ONNX Runtime, so that my code remains completely private and search requires zero API keys.
   - *Independent Test*: Ingest a source file and verify 384-dimensional embeddings are generated via `bge-small-en-v1.5` in under 10ms without internet access.
2. **US-02** (Priority: P1): As a developer, I want multi-hop graph walks in Kùzu to return in under 10ms, so that navigating callers and blast-radius dependencies feels instantaneous.
   - *Independent Test*: Execute a 3-hop openCypher query `MATCH (c:CodeSymbol)-[:CALLS*1..3]->(target)` on a 1,000-node graph and measure traversal latency.
3. **US-03** (Priority: P1): As a developer, I want saving a file to automatically update AST symbols, graph edges, and search chunks, so that indexes never drift out of sync.
   - *Independent Test*: Modify and save a Rust/TypeScript file, and verify new functions appear in LanceDB and Kùzu within 100ms.
4. **US-04** (Priority: P2): As a user updating AICO, I want database schema migrations executed automatically on startup, so that my local graph and vector caches upgrade without data loss.
   - *Independent Test*: Boot the binary against a database with an older `_schema_version` and verify migration scripts apply successfully.

---

## 3. Functional Requirements (EARS)

### Ubiquitous Requirements
- **REQ-001**: The LanceDBStore shall maintain an embedded columnar table indexed with disk-backed IVF-PQ approximate nearest neighbor vectors and BM25 inverted lexical indexes.
- **REQ-002**: The KuzuGraphStore shall execute openCypher queries over an in-process columnar Compressed Sparse Row (CSR) property graph storage layer.
- **REQ-003**: The EmbeddingPipeline shall generate 384-dimensional vector embeddings using an int8-quantized `bge-small-en-v1.5` ONNX model executing on CPU with AVX2 or NEON acceleration.
- **REQ-004**: The SchemaMigrator shall execute idempotent migration scripts on boot before the storage engines accept read or write queries.

### Event-Driven Requirements
- **REQ-005**: When a source file (`*.ts`, `*.rs`, `*.py`, `*.md`) is saved, the TreeSitterWatcher shall extract AST symbols, function signatures, import statements, and call sites within 15 milliseconds.
- **REQ-006**: When AST extraction completes, the KuzuGraphStore shall merge `CodeSymbol`, `Module`, `Test`, and `Requirement` nodes and create corresponding `[:CALLS]`, `[:IMPORTS]`, `[:IMPLEMENTS]`, and `[:VERIFIES]` edges.
- **REQ-007**: When markdown documentation or code docstrings are parsed, the LanceDBStore shall chunk content by semantic AST boundaries and upsert text embeddings into the hybrid index.
- **REQ-008**: When a hybrid search query is issued, the LanceDBStore shall calculate Reciprocal Rank Fusion (RRF) scores combining BM25 lexical rank and cosine vector similarity.

### State-Driven Requirements
- **REQ-009**: While the TreeSitterWatcher is active, the FileSystemMonitor shall debounce rapid save events occurring within 100 milliseconds to prevent redundant indexing cycles.
- **REQ-010**: While multi-hop Cypher traversals are executing, the KuzuGraphStore shall complete 3-to-6 hop paths in under 10 milliseconds for graphs containing up to 100,000 nodes.

### Unwanted Behavior & Error Handling
- **REQ-011**: If a source file contains syntax errors that prevent full AST parsing, then the TreeSitterWatcher shall ingest available partial syntax nodes without throwing an unhandled exception.
- **REQ-012**: If the local database storage is locked by another running process, then the HostRuntime shall emit an actionable locking error and abort startup cleanly.

### Optional Feature Requirements
- **REQ-013**: Where hardware AVX2 acceleration is unavailable, the EmbeddingPipeline shall execute using standard scalar CPU execution kernels.
- **REQ-014**: Where the `--reindex` CLI flag is provided, the StorageEngine shall rebuild all LanceDB vector tables and Kùzu graph databases from the source files.

---

## 4. Designated Public Test Seams

### Primary Seams:
1. **Search Engine Interface (`aico_core::storage::SearchEngine`)**:
   - Signature: `pub async fn hybrid_search(&self, query: &str, scope: SearchScope, limit: usize) -> Result<Vec<SearchHit>, StorageError>`
   - Verifies BM25 + dense vector RRF ranking across specs, code, and brainstorms.
2. **Graph Engine Interface (`aico_core::storage::GraphEngine`)**:
   - Signature: `pub fn execute_cypher(&self, query: &str, params: Value) -> Result<QueryResult, GraphError>`
   - Executes openCypher traversal queries against in-process Kùzu.
3. **AST Ingestion Seam (`aico_core::watcher::AstIngestionService`)**:
   - Signature: `pub fn ingest_file(&self, path: &Path, content: &str) -> Result<IngestionStats, IngestionError>`
   - Tests Tree-sitter parsing and graph edge generation against sample fixture files.

---

## Out-of-Scope

- External hosted vector database sync (Pinecone, Qdrant, Weaviate).
- External JVM property graph engines (Neo4j).
- Cloud-only remote embedding APIs as a hard requirement (local ONNX is the primary offline default).

---

## 6. Open Clarifications

None. Ratified in [ADR-0001](../../../../adr/0001-host-runtime-and-network-topology.md) and [ADR-0006](../../../../adr/0006-packaging-aico-specs-and-version-tagging.md).
