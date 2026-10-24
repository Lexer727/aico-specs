---
id: plan-ft-storage-workspace-crawler-and-reindexing
type: plan
aico_skills_version: "0.1.2"
epic: "ec-storage"
order: 4
title: "Workspace Vector Crawler, Graph Rebuild & Settings Maintenance Controls - Implementation Plan"
status: "Completed"
derives_from: ["spec-ft-storage-workspace-crawler-and-reindexing"]
repositories: ["aico", "aico-specs"]
created_at: "2026-10-24T21:57:00.000Z"
---

# Implementation Plan: Workspace Vector Crawler, Graph Rebuild & Settings Maintenance Controls

## 1. Overview & Strategy

This plan delivers automated workspace indexing, graph cache invalidation, and Settings Modal maintenance actions through 4 vertical tracer-bullet tasks:
1. Canonicalize project ID in `aico-core` to eliminate duplicate `.aico/storage/graph/aico` directories (`TASK-STORAGE-004-1`).
2. Build `WorkspaceCrawler` and expose `POST /api/v1/storage/reindex` (`TASK-STORAGE-004-2`).
3. Implement `POST /api/v1/storage/rebuild-graph` and wire startup crawl in `main.rs` (`TASK-STORAGE-004-3`).
4. Add interactive Rebuild Graph and Reindex Vector controls in the WebUI Settings Modal with status feedback badges (`TASK-STORAGE-004-4`).

---

## 2. Task Graph (DAG)

### Phase 1: Canonical Project ID Discovery & Cleanup

- **TASK-STORAGE-004-1**: Canonicalize Project ID in `ProjectManifest` & `RootScanner`
  - **Status**: Done
  - **Addresses**: `REQ-001`
  - **Blocked by**: None (Can start immediately)
  - **What it delivers**:
    - Updates `ProjectManifest` in `crates/aico-core/src/manifest.rs` to retain `pub id: Option<String>`.
    - Updates `RootScanner::scan` in `crates/aico-core/src/scanner.rs` to set `DiscoveredProject.id` to `manifest.id` or folder name (`*-specs`).
    - Cleans up stale `.aico/storage/graph/aico/` directory.
  - **Verification Seam**: `crates/aico-core/tests/scanner_test.rs`

---

### Phase 2: Workspace Vector Crawler & Reindex Endpoint

- **TASK-STORAGE-004-2**: Implement Workspace Vector Crawler & `POST /api/v1/storage/reindex`
  - **Status**: Done
  - **Addresses**: `REQ-002`, `REQ-003`, `REQ-005`, `REQ-008`
  - **Blocked by**: `TASK-STORAGE-004-1`
  - **What it delivers**:
    - Implements `WorkspaceCrawler::crawl_and_index` traversing project specs and codebase files into `DifferentialIndexer` and `LanceVectorStore`.
    - Adds Axum route `POST /api/v1/storage/reindex` returning indexing statistics and broadcasting `storage_updated` over WebSockets.
  - **Verification Seam**: `crates/aico-server/tests/server_test.rs`

---

### Phase 3: Force Graph Rebuild & Startup Background Indexing

- **TASK-STORAGE-004-3**: Implement `POST /api/v1/storage/rebuild-graph` & Startup Crawl
  - **Status**: Done
  - **Addresses**: `REQ-004`, `REQ-006`, `REQ-007`
  - **Blocked by**: `TASK-STORAGE-004-2`
  - **What it delivers**:
    - Adds `force_rebuild` to `GraphCacheManager` and Axum route `POST /api/v1/storage/rebuild-graph` broadcasting `graph_updated`.
    - Spawns non-blocking background indexing task on active project during server startup in `main.rs`.
  - **Verification Seam**: `crates/aico-server/tests/server_test.rs`

---

### Phase 4: Frontend Settings Modal Maintenance Controls

- **TASK-STORAGE-004-4**: Frontend Settings Modal Maintenance Buttons & Live Feedback
  - **Status**: Done
  - **Addresses**: `REQ-009`, `REQ-010`
  - **Blocked by**: `TASK-STORAGE-004-3`
  - **What it delivers**:
    - Adds "Rebuild Lineage Graph" and "Reindex Vector Embeddings" buttons to `frontend/src/components/settings/SettingsModal.tsx`.
    - Displays loading spinners and result badges upon completion.
    - Adds Vitest component tests in `frontend/tests/SettingsModal.test.tsx`.
  - **Verification Seam**: `frontend/tests/SettingsModal.test.tsx`

---

### Phase 5: Bulk Batch Ingestion, Post-Crawl Indexing & Local-Only LanceDB

- **TASK-STORAGE-004-5**: Optimize LanceDB Ingestion with Bulk Batching and Local-Only Dependencies
  - **Status**: Done
  - **Addresses**: `REQ-003`, `REQ-011`
  - **Blocked by**: `TASK-STORAGE-004-4`
  - **What it delivers**:
    - Removes `features = ["remote"]` from `lancedb` in `crates/aico-storage/Cargo.toml`.
    - Moves `create_index` out of `upsert_records` into `create_vector_index()`.
    - Refactors `WorkspaceCrawler::crawl_and_index` to buffer chunks and upsert in bulk Arrow batches (500 chunks per batch), calling `create_vector_index()` once at the end.
  - **Verification Seam**: `crates/aico-storage/tests/storage_connection_test.rs`
