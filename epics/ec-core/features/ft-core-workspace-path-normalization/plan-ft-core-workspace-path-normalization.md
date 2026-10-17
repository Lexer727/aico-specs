---
id: plan-ft-core-workspace-path-normalization
type: plan
aico_skills_version: "0.1.2"
epic: "ec-core"
order: 14
title: "Workspace Path Normalization & Architecture Topology View Plan"
status: "Completed"
repositories: ["aico", "aico-specs"]
derives_from: ["spec-ft-core-workspace-path-normalization"]
created_at: "2026-10-17T18:25:00Z"
---

# Implementation Plan: Workspace Path Normalization & Architecture Topology View

## 1. Technical Architecture & Decisions

- **Canonical Workspace Path Representation (`REQ-001`, `REQ-002`)**:
  - In `crates/aico-server/src/server.rs`, `scan_codebase_files` computes the path relative to the workspace root directory (e.g. `d:/aico`).
  - When scanning `d:/aico/codebase/aico`, a file at `d:/aico/codebase/aico/crates/aico-server/src/server.rs` produces:
    - `path`: `codebase/aico/crates/aico-server/src/server.rs`
    - `relative_path`: `crates/aico-server/src/server.rs`
    - `name`: `server.rs`
  - In `get_raw_file_handler`, the handler first checks if `query.path` exists directly or relative to `root_dir` (`d:/aico`):
    - `let root_rel = root_dir.join(&query.path);`
    - If `root_rel.exists() && root_rel.is_file()`, stream its contents.
    - Fallback checks against `target_path` (active specs root) and `codebase_path` (repository root) ensure full backward compatibility.

- **Frontend Navigation & Tab Deduplication (`REQ-003`, `REQ-004`, `REQ-005`)**:
  - In `ShellLayout.tsx`, change `c.relative_path || c.path` to `c.path`, passing `codebase/aico/...`.
  - In `FeatureViewer.tsx` and `EpicHubView.tsx`, ensure `f.path` passes `codebase/aico/...`.
  - In `App.tsx`, introduce `normalizeWorkspacePath(file: string)`:
    - Strips host drive root prefixes (`^[a-zA-Z]:[/\\]aico[/\\]?`, `^/?[a-zA-Z]:`, etc.).
    - Normalizes backslashes to forward slashes.
    - If a path starts with `crates/` or similar codebase-relative path, prefixes with `codebase/aico/`.
    - Guarantees that opening from sidebar or feature view yields the identical tab ID `codebase/aico/crates/aico-server/src/server.rs`.

- **SettingsModal Workspace Architecture & Topology Display (`REQ-006`)**:
  - In `SettingsModal.tsx`, insert a dedicated "Workspace Architecture & Topology" card:
    - Workspace Root: `d:/aico`
    - Specifications Root: `d:/aico/*-specs/` (e.g. `d:/aico/aico-specs`)
    - Codebase Repositories: `d:/aico/codebase/*/` (e.g. `d:/aico/codebase/aico`)
    - Description: Explains that `d:/aico` is the root container decoupling specs from multi-repo codebases.

---

## 2. Vertical Task Directed Acyclic Graph (DAG)

```
TASK-CORE-014-1 (Backend Path Normalization & /api/v1/raw Workspace Root Resolution)
       │
       ├──────────────────────────────┐
       ▼                              ▼
TASK-CORE-014-2 (Frontend Path &   TASK-CORE-014-3 (SettingsModal Workspace
Tab ID Uniformity across Views)     Architecture & Topology View)
       │                              │
       └──────────────┬───────────────┘
                      ▼
TASK-CORE-014-4 (End-to-End Seam Verification & Regression Suite)
```

---

### Phase 1: Backend Path Normalization & Raw File Resolution

- **TASK-CORE-014-1**: Backend Workspace Root Path Generation and Multi-Prefix `/api/v1/raw` Resolution
  - **Status**: Done
  - **Addresses**: REQ-001, REQ-002
  - **Blocked by**: None
  - **Verification Seam**: `crates/aico-server/tests/code_scanner_test.rs`
  - **What it delivers**:
    - Updates `scan_codebase_files` in `crates/aico-server/src/server.rs` to compute `path` relative to `root_dir` (`codebase/aico/...`).
    - Updates `get_raw_file_handler` in `crates/aico-server/src/server.rs` to resolve relative to `root_dir` (`d:/aico`).
    - Adds test in `crates/aico-server/tests/code_scanner_test.rs` asserting `path` starts with `codebase/` and `/api/v1/raw?path=codebase/...` streams file content.

---

### Phase 2: Frontend Navigation & Tab Deduplication

- **TASK-CORE-014-2**: Frontend Tab ID Uniformity and Path Alignment across Views
  - **Status**: Done
  - **Addresses**: REQ-003, REQ-004, REQ-005
  - **Blocked by**: TASK-CORE-014-1
  - **Verification Seam**: `frontend/tests/CodeViewer.test.tsx`, `frontend/tests/ShellLayoutSearch.test.tsx`, `frontend/tests/FeatureViewer.test.tsx`
  - **What it delivers**:
    - Updates `ShellLayout.tsx` to display and pass `c.path` (`codebase/aico/...`).
    - Updates `FeatureViewer.tsx` and `EpicHubView.tsx` to pass `f.path`.
    - Updates `App.tsx` `handleOpenCode` with path normalization function to guarantee single tab per file.
    - Updates frontend tests to verify unified workspace paths.

---

### Phase 3: SettingsModal Workspace Topology

- **TASK-CORE-014-3**: SettingsModal Workspace Architecture & Multi-Project Topology Display
  - **Status**: Done
  - **Addresses**: REQ-006
  - **Blocked by**: None
  - **Verification Seam**: `frontend/tests/SettingsModal.test.tsx`
  - **What it delivers**:
    - Adds a "Workspace Architecture & Topology" card to `SettingsModal.tsx` documenting `d:/aico` as root workspace hosting `*-specs/` and `codebase/*/`.
    - Updates `frontend/tests/SettingsModal.test.tsx` to verify presence of the card and root path text.

---

### Phase 4: Verification & Regression

- **TASK-CORE-014-4**: End-to-End Seam Verification & Regression Suite
  - **Status**: Done
  - **Addresses**: REQ-001, REQ-002, REQ-003, REQ-004, REQ-005, REQ-006
  - **Blocked by**: TASK-CORE-014-2, TASK-CORE-014-3
  - **Verification Seam**: `verify-artifacts.mjs`, `cargo test`, `npm test`
  - **What it delivers**:
    - Runs `verify-artifacts.mjs` ensuring 100% specification and plan validity.
    - Runs all Cargo unit and integration tests.
    - Runs all Vitest frontend suites.
