---
id: plan-ft-git-backend-status
type: plan
aico_skills_version: "0.1.2"
epic: "ec-git"
order: 1
title: "Multi-Repo Manifest Discovery & Git Status Inspection Engine Plan"
status: "Completed"
repositories: ["aico", "aico-specs"]
derives_from: ["spec-ft-git-backend-status"]
created_at: "2026-10-07T00:30:00Z"
---

# Implementation Plan: Multi-Repo Manifest Discovery & Git Status Inspection Engine

## 1. Technical Architecture & Decisions

- **Manifest Schema Extension**: Extend `ProjectManifest` in `crates/aico-core/src/manifest.rs` with an optional `repositories: Option<Vec<ProjectRepository>>` field. When omitted, default to two repositories: `codebase` (from `codebase_path`) and `specs` (from the specs directory).
- **Asynchronous Tokio Git Runner**: Implement `GitStatusScanner` in `crates/aico-server/src/git.rs` using `tokio::process::Command`. Execute `git status --porcelain=v2 --branch` and `git log -1 --format=%H|%s|%an|%cI` without C library dependencies.
- **Robust Error Handling**: If a directory does not exist or is not a git repository, return a structured `GitRepoStatus` with `is_available: false` and the error message, ensuring other repositories continue reporting accurately.
- **REST Telemetry Endpoint**: Expose `GET /api/v1/git/status` via Axum in `crates/aico-server/src/server.rs`.

---

## 2. Task Graph (DAG)

### Phase 1: Core Manifest Extension

- **TASK-GIT-001-1**: Multi-Repo Manifest Parsing in `aico-core`
  - **Status**: Done
  - **Addresses**: REQ-001, REQ-002
  - **Blocked by**: None (Can start immediately)
  - **What it delivers**:
    - `ProjectRepository` struct in `crates/aico-core/src/manifest.rs` with `id`, `name`, `path`, and `repo_type`.
    - Manifest parsing supporting `repositories` array with backward-compatible fallback to `codebase_path` and specs root.
  - **Verification Seam**: `crates/aico-core/tests/manifest_test.rs::test_manifest_multi_repo_parsing`

### Phase 2: Asynchronous Git Engine

- **TASK-GIT-001-2**: Asynchronous Git Status Inspection Engine in `aico-server`
  - **Status**: Done
  - **Addresses**: REQ-003, REQ-004, REQ-006
  - **Blocked by**: TASK-GIT-001-1
  - **What it delivers**:
    - `GitRepoStatus` and `GitStatusScanner` in `crates/aico-server/src/git.rs` executing async `git status --porcelain=v2 --branch` via Tokio.
    - Parsers for branch, upstream tracking remote, ahead/behind counts, working tree cleanliness, and latest commit info.
    - Graceful error handling (`is_available: false`) on missing paths or git failures.
  - **Verification Seam**: `crates/aico-server/tests/git_scanner_test.rs::test_git_status_inspection`

### Phase 3: REST API Integration

- **TASK-GIT-001-3**: Git Status REST API Endpoint in `aico-server`
  - **Status**: Done
  - **Addresses**: REQ-005
  - **Blocked by**: TASK-GIT-001-2
  - **What it delivers**:
    - Route `GET /api/v1/git/status` in `crates/aico-server/src/server.rs` returning status list for all active repositories.
    - Integration test asserting valid JSON payload containing repository status objects.
  - **Verification Seam**: `crates/aico-server/tests/server_test.rs::test_git_status_endpoint`
