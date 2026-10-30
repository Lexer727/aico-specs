---
id: plan-ft-core-canonical-workspace-storage-root
type: plan
aico_skills_version: "0.1.2"
epic: "ec-core"
order: 23
title: "Canonical Workspace Storage Root and Terminal CWD Isolation Implementation Plan"
status: "Completed"
repositories: ["aico"]
derives_from: ["spec-ft-core-canonical-workspace-storage-root"]
created_at: "2026-10-30T18:55:00.000Z"
---

# Implementation Plan: Canonical Workspace Storage Root and Terminal CWD Isolation

## 1. Technical Architecture & Decisions

- **Target repository**: `aico`.
- **Root Resolution Centralization**:
  - Implement `resolve_workspace_root_from_specs_path(specs_path: &Path) -> PathBuf` in `crates/aico-server/src/server.rs`.
  - Logic:
    1. Check `AICO_ROOT` environment variable.
    2. Inspect `specs_path.parent()`. If its file name is `"specs"`, return `parent.parent().unwrap_or(parent)`.
    3. Otherwise return `parent` (or `specs_path` if parent is empty).
  - Update `get_workspace_root(registry: &WorkspaceRegistry)` to delegate to `resolve_workspace_root_from_specs_path(&active.specs_path)`.
- **Lineage Graph Storage Isolation**:
  - In `LineageGraphCache::build_or_get()`, replace `specs_path.parent().unwrap_or(specs_path)` with `resolve_workspace_root_from_specs_path(specs_path)`.
  - Guarantees `lineage_graph.json` is saved to `<workspace_root>/.aico/storage/graph/<project_id>/lineage_graph.json`.
- **Terminal Working Directory Alignment**:
  - In `crates/aico-server/src/terminal.rs:528-536`, replace `active.specs_path.parent()` with `resolve_workspace_root_from_specs_path(&active.specs_path)`.
  - Ensures default terminal directory is the top-level workspace root (`d:/aico`).
- **Server Bootstrap Storage Federation Binding**:
  - In `crates/aico-server/src/main.rs:60`, instantiate `storage_federation` as `Arc::new(aico_storage::StorageFederationManager::with_workspace_root(&root_dir))` instead of `StorageFederationManager::new()`.
- **Cleanup & Verification**:
  - Prune the stray directory `d:/aico/specs/.aico`.
  - Add regression tests in `crates/aico-server/tests/server_test.rs` and `crates/aico-server/tests/conpty_bridge_test.rs`.

---

## 2. Task Graph (DAG)

- **TASK-CORE-023-1**: Implement Canonical Root Helper and Lineage Graph Isolation
  - **Status**: Done
  - **Addresses**: `REQ-001`, `REQ-002`
  - **Blocked by**: None
  - **What it delivers**:
    - Defines `resolve_workspace_root_from_specs_path()` in `crates/aico-server/src/server.rs`, refactors `get_workspace_root()`, and updates `LineageGraphCache::build_or_get()` to write to the canonical root `.aico/storage/graph/`.
  - **Verification Seam**: `crates/aico-server/tests/server_test.rs`

- **TASK-CORE-023-2**: Update Terminal Default Working Directory Resolution
  - **Status**: Done
  - **Addresses**: `REQ-003`
  - **Blocked by**: `TASK-CORE-023-1`
  - **What it delivers**:
    - Updates `resolve_default_cwd()` in `crates/aico-server/src/terminal.rs` to resolve the top-level workspace root for nested specs repositories.
  - **Verification Seam**: `crates/aico-server/tests/conpty_bridge_test.rs`

- **TASK-CORE-023-3**: Explicit Storage Federation Root Injection at Server Bootstrap
  - **Status**: Done
  - **Addresses**: `REQ-004`
  - **Blocked by**: None
  - **What it delivers**:
    - Updates `crates/aico-server/src/main.rs` to instantiate `StorageFederationManager::with_workspace_root(&root_dir)` at startup.
  - **Verification Seam**: `crates/aico-server/tests/server_test.rs`

- **TASK-CORE-023-4**: Prune Stray `specs/.aico` Directory and Add Regression Tests
  - **Status**: Done
  - **Addresses**: `REQ-005`
  - **Blocked by**: `TASK-CORE-023-1`, `TASK-CORE-023-2`, `TASK-CORE-023-3`
  - **What it delivers**:
    - Deletes `d:/aico/specs/.aico` and adds regression assertions verifying that `specs/.aico` is never created and all tests pass cleanly.
  - **Verification Seam**: `crates/aico-server/tests/server_test.rs`
