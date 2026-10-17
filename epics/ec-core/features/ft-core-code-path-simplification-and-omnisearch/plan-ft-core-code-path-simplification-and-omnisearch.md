---
id: plan-ft-core-code-path-simplification-and-omnisearch
type: plan
aico_skills_version: "0.1.2"
epic: "ec-core"
order: 15
title: "Code Path Simplification & OmniSearch Consistency Plan"
status: "Completed"
repositories: ["aico", "aico-specs"]
derives_from: ["spec-ft-core-code-path-simplification-and-omnisearch"]
created_at: "2026-10-17T18:57:00Z"
---

# Implementation Plan: Code Path Simplification & OmniSearch Consistency

## 1. Technical Architecture & Decisions

- **Codebase Repository-Relative Path Standard (`REQ-001`, `REQ-002`)**:
  - In `crates/aico-server/src/server.rs`, `scan_codebase_files` sets `path: relative_path.clone()` and `relative_path: relative_path.clone()`.
  - For a file at `d:/aico/codebase/aico/crates/aico-server/src/server.rs`, both `path` and `relative_path` become `crates/aico-server/src/server.rs`.
  - In `get_raw_file_handler`, the handler checks `code_path.join(&query.path)` first or `root_dir.join(&query.path)`, resolving `crates/...` cleanly.

- **OmniSearch Relative Path Title Emission (`REQ-003`, `REQ-004`)**:
  - In `crates/aico-server/src/server.rs`:
    - In `search_handler`, for code search hits:
      ```rust
      hits.push(SearchHit {
          id: file.relative_path.clone(),
          title: file.relative_path.clone(),
          scope: "code".to_string(),
          project_id: project.id.clone(),
          project_name: project.name.clone(),
          snippet,
          path: Some(file.relative_path.clone()),
      });
      ```
  - In `frontend/src/utils/searchEngine.ts`:
    - Set `title: filePath` (`crates/...`) when indexing code items.
  - In `frontend/src/components/search/OmniSearchModal.tsx`:
    - For items where `item.scope === 'code'`, render `item.title` (the relative file path) in font-mono style, keeping `item.subtitle` for snippet lines.

- **Frontend Normalization & Unified Views (`REQ-005`, `REQ-006`)**:
  - In `frontend/src/utils/pathUtils.ts`:
    - Update `normalizeWorkspacePath(file: string)`:
      ```ts
      export function normalizeWorkspacePath(file: string): string {
        if (!file) return file;
        let normalized = file.replace(/\\/g, '/');
        normalized = normalized.replace(/^(\/?[a-zA-Z]:)?\/aico\//i, '');
        normalized = normalized.replace(/^\/+/, '');
        normalized = normalized.replace(/^codebase\/[^/]+\//i, '');
        normalized = normalized.replace(/^codebase\//i, '');
        return normalized;
      }
      ```
    - Guarantees that any input (`d:/aico/...`, `codebase/aico/...`, `crates/...`) normalizes to `crates/aico-server/src/server.rs`.
  - `ShellLayout`, `FeatureViewer`, `EpicHubView`, and `CodeViewer` align automatically on `crates/...`.

---

## 2. Vertical Task Directed Acyclic Graph (DAG)

```
TASK-CORE-015-1 (Backend Code Path Simplification & SearchHit Relative Path Output)
       │
       ├──────────────────────────────┐
       ▼                              ▼
TASK-CORE-015-2 (OmniSearch Modal Code    TASK-CORE-015-3 (Universal pathUtils Normalization
Path Display & searchEngine Indexing)    & View Consistency across WebUI)
       │                              │
       └──────────────┬───────────────┘
                      ▼
TASK-CORE-015-4 (End-to-End Seam Verification & Regression Suite)
```

---

### Phase 1: Backend Code Path Simplification & SearchHit Relative Path Output

- **TASK-CORE-015-1**: Backend CodeFileInfo Path Simplification & SearchHit Relative Path Output
  - **Status**: Done
  - **Addresses**: REQ-001, REQ-002, REQ-003
  - **Blocked by**: None
  - **Verification Seam**: `crates/aico-server/tests/code_scanner_test.rs`, `crates/aico-server/tests/search_test.rs`
  - **What it delivers**:
    - Updates `scan_codebase_files` in `crates/aico-server/src/server.rs` to set `path` and `relative_path` to repository-relative format (`crates/...`).
    - Updates `search_handler` in `crates/aico-server/src/server.rs` to set `title: file.relative_path.clone()` for code search hits.
    - Updates `code_scanner_test.rs` and `search_test.rs` asserting repository-relative path emission.

---

### Phase 2: OmniSearch Modal Code Path Display & searchEngine Indexing

- **TASK-CORE-015-2**: OmniSearch Modal Code Path Title Display and Search Engine Indexing
  - **Status**: Done
  - **Addresses**: REQ-003, REQ-004
  - **Blocked by**: TASK-CORE-015-1
  - **Verification Seam**: `frontend/tests/OmniSearchModal.test.tsx`, `frontend/tests/searchEngine.test.ts`
  - **What it delivers**:
    - Updates `searchEngine.ts` to index code items with `title: filePath` (`crates/...`).
    - Updates `OmniSearchModal.tsx` to display relative file path in font-mono style for code hits.
    - Updates `OmniSearchModal.test.tsx` verifying code hits display relative path in title.

---

### Phase 3: Universal pathUtils Normalization & View Consistency

- **TASK-CORE-015-3**: Update pathUtils Normalization & Verify WebUI View Alignment
  - **Status**: Done
  - **Addresses**: REQ-005, REQ-006
  - **Blocked by**: TASK-CORE-015-1
  - **Verification Seam**: `frontend/tests/CodeViewer.test.tsx`, `frontend/tests/ShellLayoutSearch.test.tsx`, `frontend/tests/FeatureViewer.test.tsx`
  - **What it delivers**:
    - Updates `normalizeWorkspacePath` in `pathUtils.ts` to strip `codebase/<project>/` and `codebase/` prefixes.
    - Updates `CodeViewer.test.tsx`, `ShellLayoutSearch.test.tsx`, and `FeatureViewer.test.tsx` to verify repository-relative path handling.

---

### Phase 4: Verification & Regression

- **TASK-CORE-015-4**: End-to-End Seam Verification & Regression Suite
  - **Status**: Done
  - **Addresses**: REQ-001, REQ-002, REQ-003, REQ-004, REQ-005, REQ-006
  - **Blocked by**: TASK-CORE-015-2, TASK-CORE-015-3
  - **Verification Seam**: `verify-artifacts.mjs`, `cargo test`, `npm test`
  - **What it delivers**:
    - Runs `verify-artifacts.mjs` ensuring 100% specification and plan validity.
    - Runs all Cargo unit and integration tests.
    - Runs all Vitest frontend suites.
