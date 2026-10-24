---
id: spec-ft-storage-workspace-crawler-and-reindexing
type: spec
aico_skills_version: "0.1.2"
epic: "ec-storage"
order: 4
title: "Workspace Vector Crawler, Graph Rebuild & Settings Maintenance Controls"
status: "Completed"
derives_from: ["brain-ec-storage", "ADR-0011"]
repositories: ["aico", "aico-specs"]
created_at: "2026-10-24T21:57:00.000Z"
---

# Feature Specification: Workspace Vector Crawler, Graph Rebuild & Settings Maintenance Controls

## 1. Problem Statement & User Value

While `ec-storage` establishes embedded storage engines (`AicoGraphStore` and `LanceVectorStore`), developers currently encounter two operational friction points:
1. **Empty Vector Store on Startup**: The persistent `LanceVectorStore` directory at `<workspace-root>/.aico/storage/vector/` remains empty because `aico-server` does not run an automatic background crawler or startup indexing job that ingests codebase and specification files into LanceDB.
2. **Missing Maintenance Controls**: Developers editing specifications or code outside the UI have no explicit way to force an immediate graph rebuild or vector re-index without restarting the server or deleting disk files.
3. **Project ID Inconsistency**: In `crates/aico-core/src/scanner.rs`, `RootScanner` previously set project ID to `manifest.name` (`"aico"`) instead of `"aico-specs"`, creating an extraneous empty folder `.aico/storage/graph/aico` alongside `.aico/storage/graph/aico-specs`.

This feature provides:
- Canonical project ID resolution so that all projects consistently use their specs folder name (`*-specs`, e.g. `aico-specs`), eliminating redundant graph directories.
- An automatic, non-blocking workspace vector crawler and indexer on server startup.
- `POST /api/v1/storage/reindex` and `POST /api/v1/storage/rebuild-graph` endpoints for on-demand maintenance.
- Interactive "Rebuild Lineage Graph" and "Reindex Vector Embeddings" controls in the WebUI Settings Modal with real-time status feedback.

---

## 2. In-Scope & Out-of-Scope Boundaries

### In-Scope
- Canonical project ID resolution in `ProjectManifest` and `RootScanner` matching the specs folder name.
- Automatic non-blocking workspace vector crawler on startup indexing codebase and specs into LanceDB with bulk Arrow batching.
- Post-crawl vector index generation via `create_vector_index()` without per-append KMeans retraining.
- Pure local-first LanceDB dependency without remote cloud SaaS features.
- `POST /api/v1/storage/reindex` and `POST /api/v1/storage/rebuild-graph` Axum endpoints.
- WebSocket broadcast events (`graph_updated`, `storage_updated`) upon maintenance completion.
- Interactive Rebuild Lineage Graph and Reindex Vector Embeddings action buttons with spinners and result badges in `SettingsModal.tsx`.

### Out-of-Scope
- Cloud or remote vector database synchronization (strictly local-first LanceDB).
- External database processes or standalone daemon services.
- Automatic Git commit or push upon indexing.

---

## 3. User Stories

1. **US-01** (Priority: P1): As a developer opening the AICO workspace, I want my codebase and specifications to be automatically indexed into LanceDB in the background, so that hybrid and vector search tools are immediately operational without manual commands.
   - *Independent Test*: Start `aico-server` against a workspace; observe `.aico/storage/vector/` contains indexed chunks and `GET /api/v1/storage/stats` returns `vector_chunks_count > 0`.
2. **US-02** (Priority: P1): As a developer in the WebUI Settings Modal, I want a "Rebuild Lineage Graph" button, so that I can force a clean refresh of the property graph and snapshot file at any time.
   - *Independent Test*: Click "Rebuild Lineage Graph" in Settings; verify `POST /api/v1/storage/rebuild-graph` is invoked, returns 200 with node/edge counts, and the graph updates.
3. **US-03** (Priority: P1): As a developer in the WebUI Settings Modal, I want a "Reindex Vector Embeddings" button, so that I can trigger an on-demand re-chunking and re-embedding pass across all project files.
   - *Independent Test*: Click "Reindex Vector Embeddings" in Settings; verify `POST /api/v1/storage/reindex` is invoked, returns 200 with chunk statistics, and displays live confirmation.
4. **US-04** (Priority: P2): As a developer inspecting `.aico/storage/graph/`, I want graph snapshots to be strictly organized by canonical specs folder ID (e.g. `aico-specs`), so that no duplicate or orphaned project directories are created.
   - *Independent Test*: Inspect `.aico/storage/graph/`; verify only `aico-specs` exists and no empty `aico` directory is generated.

---

## 4. Functional Requirements (EARS)

### Ubiquitous Requirements
- **REQ-001**: The `ProjectManifest` and `RootScanner` shall resolve `DiscoveredProject.id` from `manifest.id` or the specifications folder name (e.g. `aico-specs`), ensuring all storage operations reference the canonical `*-specs` identifier.
- **REQ-002**: The `WorkspaceCrawler` shall traverse the active project's specifications directory (`epics/`, `features/`, `adr/`, `glossary.md`) and codebase directory (`.rs`, `.ts`, `.tsx`, `.js`, `.py`, ignoring `.git`, `node_modules`, `target`, and `dist`).
- **REQ-003**: The `WorkspaceCrawler` shall chunk traversed files, buffer them into bulk batches (up to 500 records per Arrow batch), upsert them into the unified `LanceVectorStore` at `<workspace-root>/.aico/storage/vector/` tagged with `project_id`, and invoke `create_vector_index` once upon crawl completion, maintaining sub-second ingestion throughput.

### Event-Driven Requirements
- **REQ-004**: When the server finishes binding during startup, the system shall spawn a non-blocking background task executing `WorkspaceCrawler::crawl_and_index` for the active project without delaying HTTP service readiness.
- **REQ-005**: When a client sends a `POST /api/v1/storage/reindex` request, the server shall execute `WorkspaceCrawler::crawl_and_index` and return a JSON payload with `status: "ok"`, `project_id`, `files_scanned`, `chunks_upserted`, and `total_chunks`.
- **REQ-006**: When a client sends a `POST /api/v1/storage/rebuild-graph` request, the server shall clear the `GraphCacheManager` in-memory cache and file manifest, rebuild `lineage_graph.json` under `<workspace-root>/.aico/storage/graph/<project_id>/`, and return `status: "ok"` with `nodes_count` and `edges_count`.
- **REQ-007**: When `POST /api/v1/storage/rebuild-graph` completes, the server shall broadcast a `graph_updated` event with the active `project_id` across the WebSocket hub.
- **REQ-008**: When `POST /api/v1/storage/reindex` completes, the server shall broadcast a `storage_updated` event with the active `project_id` across the WebSocket hub.

### State-Driven Requirements
- **REQ-009**: While the Settings Modal is open in the WebUI, the Embedded Storage Engines section shall render a "Rebuild Lineage Graph" button (`data-testid="rebuild-graph-btn"`) and a "Reindex Vector Embeddings" button (`data-testid="reindex-vectors-btn"`).
- **REQ-010**: While either maintenance operation is executing, the corresponding button shall display an active loading spinner and disabled state, updating upon completion with a badge displaying the operation results.

### Unwanted Behavior & Constraints
- **REQ-011**: The `crates/aico-storage` crate shall depend on `lancedb` strictly as a local embedded library without `remote` cloud features, guaranteeing offline, daemonless local-first storage.

---

## 5. Designated Public Test Seams

### Primary Seams:
1. **Discovery Project ID Seam (`crates/aico-core/tests/scanner_test.rs`)**:
   - Signature: `RootScanner::scan(root: &Path) -> Result<Vec<DiscoveredProject>, ManifestError>`
   - Verifies: `DiscoveredProject.id` equals `aico-specs` / `alpha-specs` (`REQ-001`).
2. **Workspace Vector Indexing API Seam (`crates/aico-server/tests/server_test.rs`)**:
   - Signature: `POST /api/v1/storage/reindex`
   - Verifies: Traversal, chunking into LanceDB table, and response schema (`REQ-002`, `REQ-003`, `REQ-005`, `REQ-008`).
3. **Force Graph Rebuild API Seam (`crates/aico-server/tests/server_test.rs`)**:
   - Signature: `POST /api/v1/storage/rebuild-graph`
   - Verifies: Memory cache invalidation, disk snapshot refresh, and websocket notification (`REQ-006`, `REQ-007`).
4. **Settings Modal Maintenance UI Seam (`frontend/tests/SettingsModal.test.tsx`)**:
   - Component: `<SettingsModal isOpen={true} onClose={...} />`
   - Verifies: Button rendering, loading states, and API triggers (`REQ-009`, `REQ-010`).

---

## 6. Traceability Matrix

| Requirement | Description | Implementing Module | Verification Seam |
| :--- | :--- | :--- | :--- |
| `REQ-001` | Canonical project ID in manifest & scanner | `aico-core::scanner`, `aico-core::manifest` | `scanner_test.rs` |
| `REQ-002` | Workspace crawler file traversal | `aico-server::crawler` | `server_test.rs` |
| `REQ-003` | LanceDB chunk upsert with `project_id` | `aico-server::crawler` | `server_test.rs` |
| `REQ-004` | Non-blocking startup crawl task | `aico-server::main` | `server_test.rs` |
| `REQ-005` | `POST /api/v1/storage/reindex` endpoint | `aico-server::server` | `server_test.rs` |
| `REQ-006` | `POST /api/v1/storage/rebuild-graph` endpoint | `aico-server::server` | `server_test.rs` |
| `REQ-007` | WebSocket `graph_updated` broadcast | `aico-server::server` | `server_test.rs` |
| `REQ-008` | WebSocket `storage_updated` broadcast | `aico-server::server` | `server_test.rs` |
| `REQ-009` | Settings Modal maintenance buttons | `frontend/src/.../SettingsModal.tsx` | `SettingsModal.test.tsx` |
| `REQ-010` | Loading spinners & result badges in Settings | `frontend/src/.../SettingsModal.tsx` | `SettingsModal.test.tsx` |
| `REQ-011` | Pure local-first LanceDB dependency | `aico-storage::Cargo.toml` | `storage_connection_test.rs` |
