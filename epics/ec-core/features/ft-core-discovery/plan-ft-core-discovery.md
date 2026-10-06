---
id: plan-ft-core-discovery
type: plan
aico_version: "0.1.2"
epic: "ec-core"
order: 4
title: "Project Discovery & Workspace Manifests"
status: Completed
derives_from: ["spec-ft-core-discovery"]
created_at: "2026-10-05T20:00:00Z"
---

# Implementation Plan: Project Discovery & Workspace Manifests

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

- **TASK-CORE-004-1**: Project Manifest Schema Validator & Path Normalizer
  - **Status**: Done
  - **Addresses**: `REQ-002`, `REQ-003`, `REQ-007`
  - **Blocked by**: None (Can start immediately)
  - **What it delivers**:
    - Defines Rust data models: `ProjectManifest`, `CodebaseLink`, `ProjectMetadata`.
    - Parses `.aico-project.json` and verifies required fields (`id`, `name`, `codebase`).
    - Implements bidirectional path resolution for relative (`../codebase/foo`) and absolute (`D:/dev/foo`) paths.
    - Validates project ID format (kebab-case) and checks for collision errors.
  - **Execution Waves**:
    - **Wave 1 (Scaffolding)**: Define Rust data models `ProjectManifest`, `CodebaseLink` and failing deserialization seam tests.
    - **Wave 2 (Implementation)**: Serde parser and bidirectional path normalization logic turning tests green.
    - **Wave 3 (Verification)**: Collision checks, malformed JSON tests, and path normalization assertions.
  - **Verification Seam**: `tests/project_manifest_test.rs::test_manifest_parsing_and_resolution`

---

- **TASK-CORE-004-2**: Root Directory Scanner & In-Memory Registry
  - **Status**: Done
  - **Addresses**: `REQ-001`, `REQ-005`
  - **Blocked by**: `TASK-CORE-004-1`
  - **What it delivers**:
    - Implements `RootScanner::scan(root_path)` traversing immediate subdirectories to discover `*-specs` repos.
    - Builds `WorkspaceRegistry` storing active projects, linked codebases, and git status.
    - Exposes registry methods: `get_active_project()`, `set_active_project(id)`, `list_projects()`.
  - **Execution Waves**:
    - **Wave 1 (Scaffolding)**: `RootScanner` trait interface and test fixtures with dummy `-specs` folders.
    - **Wave 2 (Implementation)**: Recursive traversal and `WorkspaceRegistry` in-memory state.
    - **Wave 3 (Verification)**: Git status detection and registry query operations verification.
  - **Verification Seam**: `tests/project_scanner_test.rs::test_root_scanner_and_registry`

---

- **TASK-CORE-004-3**: Project Management REST API
  - **Status**: Done
  - **Addresses**: `REQ-008`
  - **Blocked by**: `TASK-CORE-004-2`, `TASK-CORE-001-1` (Axum Server)
  - **What it delivers**:
    - Implements `GET /api/v1/projects` returning array of registered projects with metadata.
    - Implements `POST /api/v1/projects/active` switching active workspace.
    - Implements `POST /api/v1/projects` creating a new specs repository and generating `.aico-project.json`.
  - **Execution Waves**:
    - **Wave 1 (Scaffolding)**: Axum router routes and failing HTTP response contract tests.
    - **Wave 2 (Implementation)**: Handlers for `GET /api/v1/projects` and `POST /api/v1/projects/active`.
    - **Wave 3 (Verification)**: Project creation endpoint integration and HTTP error handling tests.
  - **Verification Seam**: `tests/project_api_test.rs::test_project_rest_endpoints`

---

- **TASK-CORE-004-4**: Dynamic Hot Discovery Filesystem Watcher
  - **Status**: Done
  - **Addresses**: `REQ-006`
  - **Blocked by**: `TASK-CORE-004-2`
  - **What it delivers**:
    - Watches root directory using `notify` for new folder creations or renamed directories.
    - Hot-registers valid specs directories into `WorkspaceRegistry` without server restart.
    - Broadcasts `project_registry_updated` event over WebSocket.
  - **Execution Waves**:
    - **Wave 1 (Scaffolding)**: Notify filesystem watcher stub and mock channel listener.
    - **Wave 2 (Implementation)**: Hot-registration watcher and WebSocket event broadcaster.
    - **Wave 3 (Verification)**: Real filesystem directory rename and deletion event handling tests.
  - **Verification Seam**: `tests/project_watcher_test.rs::test_hot_discovery_watcher`

---

- **TASK-CORE-004-5**: Missing Codebase Detection & Initialization Wizard
  - **Status**: Done
  - **Addresses**: `REQ-004`
  - **Blocked by**: `TASK-CORE-004-2`
  - **What it delivers**:
    - Detects when `.aico-project.json` points to a non-existent codebase path.
    - Flags project state as `CodebaseMissing`.
    - Exposes API and WebSocket command to trigger git clone from remote URL or initialize an empty git repo.
  - **Execution Waves**:
    - **Wave 1 (Scaffolding)**: Missing codebase detector interfaces and error status mocks.
    - **Wave 2 (Implementation)**: `CodebaseMissing` state flagging and Git clone initialization wizard logic.
    - **Wave 3 (Verification)**: Integration seam testing handling broken paths and invalid repo URLs.
  - **Verification Seam**: `tests/project_missing_test.rs::test_missing_codebase_handling`

---

## 3. Active Frontier

- None remaining (All tasks completed and verified green).
