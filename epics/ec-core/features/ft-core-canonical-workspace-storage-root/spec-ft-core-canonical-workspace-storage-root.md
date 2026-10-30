---
id: spec-ft-core-canonical-workspace-storage-root
type: spec
aico_skills_version: "0.1.2"
epic: "ec-core"
order: 23
title: "Canonical Workspace Storage Root and Terminal CWD Isolation"
status: "Completed"
derives_from: ["spec-ft-core-nested-specs-folder-and-relative-paths", "ADR-0004", "ADR-0008"]
created_at: "2026-10-30T18:55:00.000Z"
---

# Feature Specification: Canonical Workspace Storage Root and Terminal CWD Isolation

## 1. Problem Statement & User Value

Following the introduction of the nested `specs/` directory (`<workspace-root>/specs/<slug>-specs/`), project specification repositories reside two filesystem directory levels below the workspace root. While `get_workspace_root()` and `StorageFederationManager` were updated to resolve grandparent directories, `LineageGraphCache::build_or_get()` in `crates/aico-server/src/server.rs` and `resolve_default_cwd()` in `crates/aico-server/src/terminal.rs` retained the legacy single-hop assumption `specs_path.parent().unwrap_or(specs_path)`.

Consequently:
1. `LineageGraphCache` created a redundant, rogue storage directory at `<workspace-root>/specs/.aico/storage/graph/<project_id>/lineage_graph.json`, violating the core architectural invariant that all platform storage (vector embeddings, lineage graphs, cache) must reside exclusively under `<workspace-root>/.aico/`.
2. New terminal sessions spawned within `<workspace-root>/specs/` instead of the canonical workspace root `<workspace-root>/` (`d:/aico`).

This feature enforces canonical workspace root resolution across all backend subsystems, binds `StorageFederationManager` with the top-level workspace root at server bootstrap, eliminates rogue storage folders under `specs/`, and locks down the behavior with automated regression tests.

---

## In-Scope

- Implementing `resolve_workspace_root_from_specs_path(specs_path: &Path) -> PathBuf` in `crates/aico-server/src/server.rs` with `AICO_ROOT` environment override support and two-level grandparent resolution for nested `specs/` directories.
- Refactoring `LineageGraphCache::build_or_get()` to use `resolve_workspace_root_from_specs_path()`, guaranteeing snapshots are written to `<workspace-root>/.aico/storage/graph/<project_id>/lineage_graph.json`.
- Refactoring `resolve_default_cwd()` in `crates/aico-server/src/terminal.rs` to use canonical workspace root resolution.
- Initializing `StorageFederationManager` with the explicit `workspace_root` during `aico-server` bootstrap in `crates/aico-server/src/main.rs`.
- Pruning the stray `d:/aico/specs/.aico` directory from the workspace.
- Adding regression tests asserting that `specs/.aico` is never created and that `lineage_graph.json` resides at the top-level workspace root.

---

## Out-of-Scope

- Modifying vector storage indexing schemas in `crates/aico-storage/src/vector.rs`.
- Changing frontend UI components (already updated to reflect `specs/` and `../../codebase/` paths).

---

## 2. User Stories

1. **US-01** (Priority: P1): As a developer inspecting my workspace directory, I want all `.aico/` platform data (lineage graphs, vector databases) to reside exclusively at the workspace root (`d:/aico/.aico/`), so that `specs/` contains only specification repositories.
   - *Independent Test*: Trigger lineage graph generation for a project located in `specs/<project>-specs`; verify `lineage_graph.json` is written to `<root>/.aico/storage/graph/<project_id>/` and no `.aico` directory is created in `specs/`.
2. **US-02** (Priority: P1): As a developer opening an interactive terminal tab, I want the default working directory to be the top-level workspace root (`d:/aico`), so that I can immediately run Git and Cargo commands across the workspace.
   - *Independent Test*: Request default shell working directory for an active project located in `specs/`; verify the returned path is the top-level workspace root (`d:/aico`).

---

## 3. Requirements (EARS Syntax)

- **REQ-001**: The system shall provide a canonical root resolution function `resolve_workspace_root_from_specs_path(specs_path: &Path) -> PathBuf` that returns `AICO_ROOT` if set, the grandparent directory if `specs_path.parent()` has folder name `specs`, or the parent directory otherwise.
- **REQ-002**: When persisting a lineage graph snapshot, `LineageGraphCache` shall write `lineage_graph.json` exclusively into `<workspace_root>/.aico/storage/graph/<project_id>/`, where `<workspace_root>` is resolved via `resolve_workspace_root_from_specs_path()`.
- **REQ-003**: When resolving the default working directory for a terminal session without an explicit directory parameter, the terminal bridge shall return the workspace root resolved via `resolve_workspace_root_from_specs_path(&active.specs_path)`.
- **REQ-004**: When the host server starts up in `crates/aico-server/src/main.rs`, the server shall instantiate `StorageFederationManager::with_workspace_root(&root_dir)` and bind it to `AppState`.
- **REQ-005**: The system shall ensure that no `.aico` folder exists or is generated inside `<workspace-root>/specs/`.

---

## 4. Designated Public Test Seams

### Automated Regression Tests
- Seam: `crates/aico-server/tests/server_test.rs`
  - Assert that `LineageGraphCache::build_or_get()` writes to `<temp_dir>/.aico/storage/graph/<project_id>/lineage_graph.json`.
  - Assert that `<temp_dir>/specs/.aico` does not exist after lineage graph generation.
- Seam: `crates/aico-server/tests/conpty_bridge_test.rs`
  - Assert that `resolve_default_cwd()` returns `<temp_dir>` when `specs_path` is `<temp_dir>/specs/<project_id>`.
