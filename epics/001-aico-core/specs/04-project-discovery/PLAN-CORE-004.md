# Implementation Plan: Project Discovery & Workspace Manifests

- **ID**: PLAN-CORE-004
- **Spec Reference**: [SPEC-CORE-004.md](file:///d:/aico/aico-specs/epics/001-aico-core/specs/04-project-discovery/SPEC-CORE-004.md)
- **Status**: Ready

---

## 1. Technical Architecture & Decisions

- **Crate**: `crates/aico-core` (or sub-module `aico_core::discovery`).
- **Core Dependencies**:
  - `walkdir`: Fast recursive directory traversal for root discovery.
  - `serde` / `serde_json`: Deserialization and validation of `.aico-project.json` manifests.
  - `notify`: Filesystem watcher detecting newly created `*-specs` or codebase directories.
  - `axum`: REST endpoints mounted under `/api/v1/projects`.
- **Discovery Strategy**: On startup, scan the configured AICO root (e.g., `d:/aico/`). Look for directories ending in `-specs` containing `.aico-project.json`. Validate project ID uniqueness and resolve codebase links immediately into memory.

---

## 2. Task Graph (DAG)

### Phase 1: Manifest Schema & Path Resolution (Unblocked Frontier)

- [x] **TASK-CORE-004-1**: Project Manifest Schema Validator & Path Normalizer
  - **Addresses**: `REQ-002`, `REQ-003`, `REQ-007`
  - **Blocked by**: None (Can start immediately)
  - **What it delivers**:
    - Defines Rust data models: `ProjectManifest`, `CodebaseLink`, `ProjectMetadata`.
    - Parses `.aico-project.json` and verifies required fields (`id`, `name`, `codebase`).
    - Implements bidirectional path resolution for relative (`../codebase/foo`) and absolute (`D:/dev/foo`) paths.
    - Validates project ID format (kebab-case) and checks for collision errors.
  - **Verification Seam**: `tests/project_manifest_test.rs::test_manifest_parsing_and_resolution`

---

### Phase 2: Root Directory Scanner & Workspace Registry

- [x] **TASK-CORE-004-2**: Root Directory Scanner & In-Memory Registry
  - **Addresses**: `REQ-001`, `REQ-005`
  - **Blocked by**: `TASK-CORE-004-1`
  - **What it delivers**:
    - Implements `RootScanner::scan(root_path)` traversing immediate subdirectories to discover `*-specs` repos.
    - Builds `WorkspaceRegistry` storing active projects, linked codebases, and git status.
    - Exposes registry methods: `get_active_project()`, `set_active_project(id)`, `list_projects()`.
  - **Verification Seam**: `tests/project_scanner_test.rs::test_root_scanner_and_registry`

---

### Phase 3: REST API Endpoints

- [x] **TASK-CORE-004-3**: Project Management REST API
  - **Addresses**: `REQ-008`
  - **Blocked by**: `TASK-CORE-004-2`, `TASK-CORE-001-1` (Axum Server)
  - **What it delivers**:
    - Implements `GET /api/v1/projects` returning array of registered projects with metadata.
    - Implements `POST /api/v1/projects/active` switching active workspace.
    - Implements `POST /api/v1/projects` creating a new specs repository and generating `.aico-project.json`.
  - **Verification Seam**: `tests/project_api_test.rs::test_project_rest_endpoints`

---

### Phase 4: Dynamic Hot Discovery & Git Clone Helper

- [ ] **TASK-CORE-004-4**: Dynamic Hot Discovery Filesystem Watcher
  - **Addresses**: `REQ-006`
  - **Blocked by**: `TASK-CORE-004-2`
  - **What it delivers**:
    - Watches root directory using `notify` for new folder creations or renamed directories.
    - Hot-registers valid specs directories into `WorkspaceRegistry` without server restart.
    - Broadcasts `project_registry_updated` event over WebSocket.
  - **Verification Seam**: `tests/project_watcher_test.rs::test_hot_discovery_watcher`

- [ ] **TASK-CORE-004-5**: Missing Codebase Detection & Initialization Wizard
  - **Addresses**: `REQ-004`
  - **Blocked by**: `TASK-CORE-004-2`
  - **What it delivers**:
    - Detects when `.aico-project.json` points to a non-existent codebase path.
    - Flags project state as `CodebaseMissing`.
    - Exposes API and WebSocket command to trigger git clone from remote URL or initialize an empty git repo.
  - **Verification Seam**: `tests/project_missing_test.rs::test_missing_codebase_handling`

---

## 3. Initial Active Frontier

- `TASK-CORE-004-1`: Project Manifest Schema Validator & Path Normalizer (Ready for `/aico-test-scaffolding`).
