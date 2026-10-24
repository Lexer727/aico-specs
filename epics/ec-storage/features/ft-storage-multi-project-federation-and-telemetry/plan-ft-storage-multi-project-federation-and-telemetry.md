---
id: plan-ft-storage-multi-project-federation-and-telemetry
type: plan
aico_skills_version: "0.1.2"
epic: "ec-storage"
order: 3
title: "Multi-Project Storage Federation, Cache Invalidation & Telemetry Implementation Plan"
status: "Completed"
repositories: ["aico", "aico-specs"]
derives_from: ["spec-ft-storage-multi-project-federation-and-telemetry"]
created_at: "2026-10-24T17:26:21.874Z"
---

# Implementation Plan: Multi-Project Storage Federation, Cache Invalidation & Telemetry

## 1. Technical Architecture & Decisions

- **Storage Federation Manager**: Implemented in `crates/aico-storage/src/manager.rs` as `StorageFederationManager`, wrapping an `Arc<RwLock<HashMap<String, ProjectStorage>>>`.
- **Project Isolation**: Each registered project resolves its storage root to `<specs_path>/.aico/storage/` containing its dedicated `lineage_graph.json` and LanceDB directory.
- **Dynamic Swapping**: When `POST /api/v1/projects/switch` is received, `StorageFederationManager` switches the active pointer with zero file contention.
- **Federated Search**: `HybridQueryEngine::search_federated` aggregates results from all registered project stores in parallel using `tokio::join!`, normalizing and sorting via global RRF.
- **Telemetry & UI**:
  - Exposes `GET /api/v1/storage/stats` with node count, edge count, vector count, and directory size.
  - Updates `frontend/src/features/dashboard/DashboardView.tsx` Host Diagnostics to display live metrics.

---

## 2. Task Graph (DAG)

### Phase 1: Storage Federation Manager & Isolation

- **TASK-STORAGE-003-1**: Implement `StorageFederationManager` & Per-Project Handle Pooling
  - **Status**: Done
  - **Addresses**: `REQ-001`, `REQ-002`, `REQ-007`
  - **Blocked by**: None (Can start immediately)
  - **What it delivers**:
    - Implements `StorageFederationManager` maintaining a map of project storage contexts.
    - Ensures each project maintains its own isolated `.aico/storage/` folder.
    - Proves concurrency and non-blocking multi-project access.
  - **Verification Seam**: `crates/aico-storage/tests/storage_connection_test.rs::test_federation_handle_pooling_and_isolation`

---

### Phase 2: Dynamic Project Switching & Handle Swap

- **TASK-STORAGE-003-2**: Implement Dynamic Active Project Storage Switching
  - **Status**: Done
  - **Addresses**: `REQ-005`, `REQ-008`, `REQ-009`, `REQ-010`
  - **Blocked by**: `TASK-STORAGE-003-1`
  - **What it delivers**:
    - Wires active project switcher in `server.rs` to hot-swap active `StorageFederationManager` handles within 50ms.
    - Verifies existing queries complete cleanly without deadlocks.
    - Returns proper 404 error if an invalid project is requested.
  - **Verification Seam**: `crates/aico-server/tests/server_test.rs::test_dynamic_project_storage_switching`

---

### Phase 3: Cross-Project Federated Hybrid Search

- **TASK-STORAGE-003-3**: Implement Cross-Project Federated Search (`scope: "all"`)
  - **Status**: Done
  - **Addresses**: `REQ-006`
  - **Blocked by**: `TASK-STORAGE-003-1`
  - **What it delivers**:
    - Extends `HybridQueryEngine` to support federated search across all registered projects.
    - Fuses results with global Reciprocal Rank Fusion, preserving project IDs.
  - **Verification Seam**: `crates/aico-storage/tests/storage_hybrid_test.rs::test_federated_hybrid_search_across_projects`

---

### Phase 4: Storage Telemetry API & WebUI Host Diagnostics

- **TASK-STORAGE-003-4**: Implement `GET /api/v1/storage/stats` & Dashboard Diagnostics UI Integration
  - **Status**: Done
  - **Addresses**: `REQ-003`, `REQ-004`
  - **Blocked by**: `TASK-STORAGE-003-2`
  - **What it delivers**:
    - Adds `GET /api/v1/storage/stats` returning node, edge, and vector counts.
    - Updates `DashboardView.tsx` Host Diagnostics to display live counts: `"AICO Graph: Embedded OK (<count> nodes)"` and `"LanceDB: Embedded OK (<count> vectors)"`.
  - **Verification Seam**: `crates/aico-server/tests/server_test.rs::test_storage_stats_endpoint`, `frontend/tests/DashboardView.test.tsx`

---

### Phase 5: Regression & Multi-Project Integration Verification

- **TASK-STORAGE-003-5**: End-to-End Multi-Project Integration Verification
  - **Status**: Done
  - **Addresses**: `REQ-001`, `REQ-003`, `REQ-004`, `REQ-005`, `REQ-006`
  - **Blocked by**: `TASK-STORAGE-003-3`, `TASK-STORAGE-003-4`
  - **What it delivers**:
    - Verifies all storage tests and server tests pass across multi-project fixtures.
    - Verifies Vitest tests for `DashboardView.test.tsx` pass green.
  - **Verification Seam**: `crates/aico-server/tests/server_test.rs`, `frontend/tests/DashboardView.test.tsx`

---

## 3. Amendment Log

- **2026-10-24T17:26:21Z**: Initial task graph established under `ec-storage`.
