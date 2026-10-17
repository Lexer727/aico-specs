---
id: plan-ft-core-multirepo-code-path-normalization
type: plan
aico_skills_version: "0.1.2"
epic: "ec-core"
order: 16
title: "Multi-Repository Code Path Normalization & Root-Agnostic Resolution Plan"
status: "Completed"
repositories: ["aico", "aico-specs"]
derives_from: ["spec-ft-core-multirepo-code-path-normalization"]
created_at: "2026-10-17T19:35:00Z"
---

# Implementation Plan: Multi-Repository Code Path Normalization & Root-Agnostic Resolution

## 1. Technical Architecture & Decisions

- **Multi-Repository Path Standard (`REQ-001`, `REQ-002`)**:
  - In `crates/aico-server/src/server.rs`, `scan_codebase_files` computes paths relative to `codebase_root` (or `<repo>/<rel-path>`).
  - For a file at `<root>/codebase/aico/crates/aico-server/src/server.rs`, both `path` and `relative_path` become `aico/crates/aico-server/src/server.rs`.
  - For a file at `<root>/codebase/aico-standalone/src/main.rs`, both `path` and `relative_path` become `aico-standalone/src/main.rs`.
  - In `get_raw_file_handler`, the handler dynamically checks `resolve_workspace_root().join("codebase").join(&query.path)` without hardcoding drive letters or root directory names.

- **OmniSearch Multi-Repo Title Output (`REQ-003`)**:
  - In `crates/aico-server/src/server.rs`:
    - In `search_handler`, code search hits set `title`, `id`, and `path` to `file.path` (e.g. `aico/crates/aico-server/src/server.rs`).
  - In `frontend/src/utils/searchEngine.ts`:
    - Code index items use `title: filePath` (`aico/crates/...`).
  - In `frontend/src/components/search/OmniSearchModal.tsx`:
    - Code hits render `item.title` (`aico/crates/...`) in font-mono style.

- **Dynamic Root-Agnostic Normalization & View Consistency (`REQ-004`, `REQ-005`)**:
  - In `frontend/src/utils/pathUtils.ts`:
    ```ts
    export function normalizeWorkspacePath(file: string): string {
      if (!file) return file;
      let normalized = file.replace(/\\/g, '/');
      const codebaseIdx = normalized.indexOf('/codebase/');
      if (codebaseIdx !== -1) {
        normalized = normalized.slice(codebaseIdx + '/codebase/'.length);
      } else if (normalized.startsWith('codebase/')) {
        normalized = normalized.slice('codebase/'.length);
      }
      const specsMatch = normalized.match(/\/([^/]+-specs\/.*)$/);
      if (specsMatch) {
        normalized = specsMatch[1];
      }
      return normalized.replace(/^\/+/, '');
    }
    ```
  - Eliminates hardcoded `d:/aico` assumptions and works for arbitrary install directories.
  - Updates `mockCodeSnippets` in `CodeViewer.tsx` to key on `aico/crates/aico-server/src/server.rs`.
  - `ShellLayout`, `FeatureViewer`, `EpicHubView`, and `CodeViewer` align uniformly on `<repo>/<subpath>`.

---

## 2. Vertical Task Directed Acyclic Graph (DAG)

```
TASK-CORE-016-1 (Backend Scanner & Dynamic Raw API Multi-Repo Path Normalization)
       │
       ├──────────────────────────────┐
       ▼                              ▼
TASK-CORE-016-2 (OmniSearch Multi-Repo    TASK-CORE-016-3 (Root-Agnostic Dynamic pathUtils
Title Output & searchEngine Indexing)     Normalization & Unified View Alignment)
       │                              │
       └──────────────┬───────────────┘
                      ▼
TASK-CORE-016-4 (End-to-End Seam Verification & Regression Suite)
```

---

### Phase 1: Backend Scanner & Dynamic Raw API Multi-Repo Path Normalization

- **TASK-CORE-016-1**: Backend Scanner & Dynamic Raw File Streaming with Multi-Repo Path Emission
  - **Status**: Done
  - **Addresses**: REQ-001, REQ-002, REQ-003
  - **Blocked by**: None
  - **Verification Seam**: `crates/aico-server/tests/code_scanner_test.rs`, `crates/aico-server/tests/search_test.rs`
  - **What it delivers**:
    - Updates `scan_codebase_files` in `crates/aico-server/src/server.rs` to compute `<repo>/<rel-path>` (e.g. `aico/crates/...`).
    - Updates `get_raw_file_handler` in `crates/aico-server/src/server.rs` to dynamically resolve `<repo>/<rel-path>` against `root_dir.join("codebase")`.
    - Updates `code_scanner_test.rs` and `search_test.rs` asserting `aico/...` path emission.

---

### Phase 2: OmniSearch Multi-Repo Title Output & searchEngine Indexing

- **TASK-CORE-016-2**: OmniSearch Multi-Repo Title Output & SearchEngine Indexing
  - **Status**: Done
  - **Addresses**: REQ-003, REQ-005
  - **Blocked by**: TASK-CORE-016-1
  - **Verification Seam**: `frontend/tests/OmniSearchModal.test.tsx`, `frontend/tests/searchEngine.test.ts`
  - **What it delivers**:
    - Updates `searchEngine.ts` to index code items with `title: filePath` (`aico/crates/...`).
    - Updates `OmniSearchModal.tsx` to render multi-repo code hits in font-mono style.
    - Updates `OmniSearchModal.test.tsx` and `searchEngine.test.ts` verifying `aico/...` title rendering.

---

### Phase 3: Root-Agnostic Dynamic pathUtils Normalization & Unified View Alignment

- **TASK-CORE-016-3**: Root-Agnostic Dynamic pathUtils Normalization & Unified View Alignment
  - **Status**: Done
  - **Addresses**: REQ-004, REQ-005
  - **Blocked by**: TASK-CORE-016-1
  - **Verification Seam**: `frontend/tests/CodeViewer.test.tsx`, `frontend/tests/ShellLayoutSearch.test.tsx`, `frontend/tests/FeatureViewer.test.tsx`
  - **What it delivers**:
    - Updates `normalizeWorkspacePath` in `pathUtils.ts` to dynamically slice after `/codebase/` without hardcoded `d:/aico`.
    - Updates `mockCodeSnippets` in `CodeViewer.tsx` to key on `aico/crates/...`.
    - Updates `CodeViewer.test.tsx`, `ShellLayoutSearch.test.tsx`, and `FeatureViewer.test.tsx` to verify `aico/...` handling.

---

### Phase 4: Verification & Regression

- **TASK-CORE-016-4**: End-to-End Seam Verification & Regression Suite
  - **Status**: Done
  - **Addresses**: REQ-001, REQ-002, REQ-003, REQ-004, REQ-005
  - **Blocked by**: TASK-CORE-016-2, TASK-CORE-016-3
  - **Verification Seam**: `verify-artifacts.mjs`, `cargo test`, `npm test`
  - **What it delivers**:
    - Runs `verify-artifacts.mjs` ensuring 100% specification and plan validity.
    - Runs all Cargo unit and integration tests.
    - Runs all Vitest frontend suites.
