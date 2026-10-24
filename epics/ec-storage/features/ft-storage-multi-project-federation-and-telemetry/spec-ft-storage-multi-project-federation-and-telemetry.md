---
id: spec-ft-storage-multi-project-federation-and-telemetry
type: spec
aico_skills_version: "0.1.2"
epic: "ec-storage"
order: 3
title: "Multi-Project Storage Federation, Dynamic Cache Switching & Storage Telemetry"
status: "Completed"
derives_from: ["brain-ec-storage", "ADR-0011"]
created_at: "2026-10-24T17:26:21.874Z"
---

# Feature Specification: Multi-Project Storage Federation, Dynamic Cache Switching & Storage Telemetry

## 1. Problem Statement & User Value

Developers often manage multiple decoupled software projects within a single AICO root workspace (e.g. `aico`, `ecommerce-service`, `mobile-app`). Each project possesses its own independent specs repository (`<project>-specs`) and codebase directory. 

Currently, storage handles in `crates/aico-storage/src/manager.rs` are initialized once on startup for the default workspace, creating two operational limitations:
1. **No Dynamic Project Storage Swapping**: Switching active projects in the WebUI does not hot-swap the underlying `AicoGraphStore` and `LanceVectorStore` instances, causing graph queries and searches to target the wrong project or requiring a server restart.
2. **Missing Storage Observability & Diagnostics**: Neither the WebUI Host Diagnostics nor MCP tools expose live metrics on graph node counts, edge counts, indexed vector record counts, or cache hit rates, making it difficult to verify whether indexing has completed.

This specification implements **Multi-Project Storage Federation**:
- A thread-safe `StorageFederationManager` that dynamically pools, lazily loads, and swaps `AicoGraphStore` and `LanceVectorStore` handles per project.
- Cross-project federated search scoping (`scope: "active"` or `scope: "all"`).
- Real-time storage telemetry exposed via `GET /api/v1/storage/stats` and integrated into the WebUI Host Diagnostics card (`"AICO Graph: Embedded OK (<N> nodes)"`, `"LanceDB: Embedded OK (<N> vectors)"`).

---

## In-Scope

- Thread-safe `StorageFederationManager` in `crates/aico-storage` maintaining per-project storage instances.
- Seamless project handle switching on `POST /api/v1/projects/switch` and WebSocket switch events within 50ms without file locking.
- Cross-project federated search option in `search_hybrid` (`scope: "active" | "all"`).
- Storage telemetry endpoint `GET /api/v1/storage/stats` returning node count, edge count, vector count, disk size, and cache status.
- Integration into `DashboardView.tsx` Host Diagnostics displaying real-time node and vector metrics.

---

## 2. User Stories

1. **US-01** (Priority: P1): As a developer switching between projects in the UI switcher, I want the active lineage graph and vector store to swap instantly to the selected project, so that I view and query the correct project data.
   - *Independent Test*: Dispatch an active project switch from Project A to Project B; verify `GET /api/v1/graph` and `GET /api/v1/storage/stats` return Project B's nodes and vectors in under 50ms.
2. **US-02** (Priority: P1): As an agent searching for reusable library functions across my entire workspace, I want to query `search_hybrid` with `scope: "all"`, so that I discover relevant code across all registered projects.
   - *Independent Test*: Call `search_hybrid` with `scope: "all"` and verify returned search hits contain results from multiple projects with `project_id` tags.
3. **US-03** (Priority: P2): As a developer inspecting the Host Diagnostics panel on the Home Dashboard, I want to see live node and vector counts, so that I can immediately verify my project's storage health.
   - *Independent Test*: Mount `DashboardView.tsx`; verify Host Diagnostics renders `"AICO Graph: Embedded OK (<count> nodes)"` and `"LanceDB: Embedded OK (<count> vectors)"`.

---

## 3. Functional Requirements (EARS)

### Ubiquitous Requirements
- **REQ-001**: The `StorageFederationManager` shall maintain a thread-safe registry of `(AicoGraphStore, LanceVectorStore)` handles indexed by `project_id`.
- **REQ-002**: The `StorageFederationManager` shall isolate disk storage for each project to `<specs-dir>/.aico/storage/`, preventing cross-project file contention.
- **REQ-003**: The server shall expose `GET /api/v1/storage/stats` returning a JSON telemetry payload containing `project_id`, `graph_nodes_count`, `graph_edges_count`, `vector_chunks_count`, `cache_state`, and `storage_dir_bytes`.
- **REQ-004**: The `DashboardView.tsx` Host Diagnostics card shall render live metrics: `"AICO Graph: Embedded OK (<count> nodes)"` and `"LanceDB: Embedded OK (<count> vectors)"`.

### Event-Driven Requirements
- **REQ-005**: When an active project switch event is received via HTTP `POST /api/v1/projects/switch` or WebSocket message, the `StorageFederationManager` shall activate the target project's storage handles within 50 milliseconds.
- **REQ-006**: When `search_hybrid` is invoked with `scope: "all"`, the `HybridQueryEngine` shall execute parallel searches across all registered project vector tables and fuse the results with global Reciprocal Rank Fusion.
- **REQ-007**: When a new project is created or registered via `ProjectRegistry`, the `StorageFederationManager` shall provision an empty storage directory and initialize baseline storage handles.

### State-Driven Requirements
- **REQ-008**: While switching active projects, existing in-flight read operations on other projects shall continue without blocking or deadlocking.
- **REQ-009**: While a project is inactive, the `StorageFederationManager` shall evict its in-memory graph cache if idle memory thresholds are reached, reloading from disk on subsequent access.

### Unwanted Behavior & Error Handling
- **REQ-010**: If an active project switch targets a non-existent project ID, then the server shall return HTTP 404 with `StorageError::ProjectNotFound` and retain the previously active project handles.

---

## 4. Designated Public Test Seams

### Primary Seams:
1. **Storage Federation & Handle Switch Seam (`crates/aico-storage/tests/storage_connection_test.rs`)**:
   - Signature: `StorageFederationManager::switch_project(&self, project_id: &str) -> Result<ProjectStorageContext, StorageError>`
   - Verifies: Dynamic handle swap, isolation, and concurrency (`REQ-001`, `REQ-002`, `REQ-005`, `REQ-008`).
2. **Federated Multi-Project Search Seam (`crates/aico-storage/tests/storage_hybrid_test.rs`)**:
   - Signature: `HybridQueryEngine::search_federated(query: &str, scope: SearchScope) -> Result<Vec<SearchHit>, StorageError>`
   - Verifies: Cross-project parallel search and global RRF ranking (`REQ-006`).
3. **Storage Telemetry API Seam (`crates/aico-server/tests/server_test.rs`)**:
   - Signature: `GET /api/v1/storage/stats`
   - Verifies: Real-time node, edge, and vector count reporting (`REQ-003`, `REQ-010`).
4. **Dashboard Diagnostics UI Seam (`frontend/tests/DashboardView.test.tsx`)**:
   - Signature: `<DashboardView ... />`
   - Verifies: Rendering live graph node and vector counts in Host Diagnostics (`REQ-004`).

---

## Out-of-Scope

- Distributed cluster coordination across multiple host machines.
- Cloud synchronization of vector databases.

---

## 5. Open Clarifications

None. Ratified in [ADR-0011](../../../../adr/0011-aico-graph-store-and-lancedb-hybrid-storage.md).
