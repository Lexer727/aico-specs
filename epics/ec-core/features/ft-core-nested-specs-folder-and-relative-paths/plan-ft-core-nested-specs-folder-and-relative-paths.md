---
id: plan-ft-core-nested-specs-folder-and-relative-paths
type: plan
aico_skills_version: "0.1.2"
epic: "ec-core"
order: 22
title: "Nested Specs Directory Structure and Relative Path Normalization Implementation Plan"
status: "Completed"
repositories: ["aico"]
derives_from: ["spec-ft-core-nested-specs-folder-and-relative-paths"]
created_at: "2026-10-30T18:20:00.000Z"
---

# Implementation Plan: Nested Specs Directory Structure and Relative Path Normalization

## 1. Technical Architecture & Decisions

- **Target repository**: `aico`.
- **Primary Specifications Directory**:
  - All project specifications will be housed inside `<workspace-root>/specs/<slug>-specs/`.
  - `RootScanner::scan()` will first scan `<workspace-root>/specs/` for entries containing `.aico-project.json`. It will also scan `<workspace-root>` as a fallback so that any unmigrated repositories remain discovered without hard breakage.
- **Root Resolution for Nested Specs**:
  - `get_workspace_root()` in `crates/aico-server/src/server.rs` and `StorageFederationManager` in `crates/aico-storage/src/manager.rs` will check if `specs_path.parent()` ends in `specs`. If so, they will step up one more directory level to return `<workspace-root>` (`d:/aico`).
- **Relative Path Trajectory**:
  - In `.aico-project.json`, codebase repository paths will be normalized to `../../codebase/<id>`.
  - `ProjectManifest::resolve_path()` in `crates/aico-core/src/manifest.rs` will resolve relative to `specs_dir`. To ensure resilient backward-compatibility, if a relative path does not exist, it will check if stepping up one additional or fewer directory levels yields an existing codebase.
- **Scaffolding & Creation**:
  - `CodebaseWizard::scaffold_project_specs()` in `crates/aico-core/src/wizard.rs` and `handle_create_project` in `crates/aico-server/src/server.rs` will create target directories at `<workspace-root>/specs/<slug>-specs/` and serialize codebase paths as `../../codebase/<id>`.
  - `resolve_clone_target_dir()` in `crates/aico-server/src/git.rs` will return `<workspace-root>/specs/<repo_id>` when `repo_type == "specs"`.
- **Frontend Alignment**:
  - In `ProjectConfigModal.tsx`:
    - Folder preview updated to `specs/<project-name>-specs`.
    - Codebase repository read-only preview updated to `../../codebase/<id>`.
    - Submission payload uses `../../codebase/${cleanId}`.
    - Codebase folder browser text updated to `../../codebase/`.
  - In `SettingsModal.tsx`:
    - Architecture & Topology section updated to display `d:/aico/specs/*-specs/`.
- **Verification Engine & Workspace Migration**:
  - Update `aico-cli.mjs` candidate manifest lookup to include `specs/aico-specs/.aico-project.json`.
  - Move physical workspace directory `d:/aico/aico-specs` into `d:/aico/specs/aico-specs` and update its `.aico-project.json` repository paths to `../../codebase/<id>`.

---

## 2. Task Graph (DAG)

- **TASK-CORE-022-1**: Implement Nested `specs/` Discovery and Root Resolution
  - **Status**: Done
  - **Addresses**: `REQ-001`, `REQ-002`, `REQ-003`
  - **Blocked by**: None
  - **What it delivers**:
    - Updates `RootScanner::scan()` to scan `specs/`, and updates `get_workspace_root()` and `StorageFederationManager` to resolve workspace root properly for nested specs.
  - **Verification Seam**: `crates/aico-core/tests/scanner_test.rs`

- **TASK-CORE-022-2**: Update Manifest Relative Path Resolution and Scaffolding
  - **Status**: Done
  - **Addresses**: `REQ-004`, `REQ-005`
  - **Blocked by**: `TASK-CORE-022-1`
  - **What it delivers**:
    - Updates `ProjectManifest::resolve_path()` to handle `../../codebase/<id>`, and updates `CodebaseWizard`, `handle_create_project`, and `resolve_clone_target_dir` to scaffold and clone under `specs/`.
  - **Verification Seam**: `crates/aico-core/tests/manifest_test.rs`, `crates/aico-server/tests/project_api_test.rs`

- **TASK-CORE-022-3**: Update Unified Verification CLI and Script Lookups
  - **Status**: Done
  - **Addresses**: `REQ-008`
  - **Blocked by**: None
  - **What it delivers**:
    - Updates `aico-cli.mjs` to resolve manifests and repo dirs under `specs/`.
  - **Verification Seam**: `scripts/verify-artifacts.mjs`

- **TASK-CORE-022-4**: Update Frontend Add/Edit Project Modal and Settings Topology UI
  - **Status**: Done
  - **Addresses**: `REQ-006`, `REQ-007`
  - **Blocked by**: `TASK-CORE-022-2`
  - **What it delivers**:
    - Updates `ProjectConfigModal.tsx` previews and submissions to `specs/` and `../../codebase/`, and updates `SettingsModal.tsx` topology documentation.
  - **Verification Seam**: `frontend/tests/ProjectSwitcher.test.tsx`, `frontend/tests/SettingsModal.test.tsx`

- **TASK-CORE-022-5**: Relocate Workspace `aico-specs` to `specs/aico-specs` and Update Manifest
  - **Status**: Done
  - **Addresses**: `REQ-001`, `REQ-004`
  - **Blocked by**: `TASK-CORE-022-1`, `TASK-CORE-022-2`, `TASK-CORE-022-3`
  - **What it delivers**:
    - Moves `d:/aico/aico-specs` into `d:/aico/specs/aico-specs` and normalizes `.aico-project.json` repository paths.
  - **Verification Seam**: `crates/aico-core/tests/scanner_test.rs`

- **TASK-CORE-022-6**: Comprehensive Integration and Regression Verification across Seams
  - **Status**: Done
  - **Addresses**: `REQ-001`, `REQ-002`, `REQ-003`, `REQ-004`, `REQ-005`, `REQ-006`, `REQ-007`, `REQ-008`
  - **Blocked by**: `TASK-CORE-022-4`, `TASK-CORE-022-5`
  - **What it delivers**:
    - Full end-to-end test execution across backend, frontend, and verification scripts ensuring zero regressions.
  - **Verification Seam**: `crates/aico-server/tests/project_api_test.rs`, `frontend/tests/ProjectSwitcher.test.tsx`, `scripts/verify-artifacts.mjs`
