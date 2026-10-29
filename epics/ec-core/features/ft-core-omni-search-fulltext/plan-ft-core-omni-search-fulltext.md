---
id: plan-ft-core-omni-search-fulltext
type: plan
aico_skills_version: "0.1.2"
epic: "ec-core"
order: 9
title: "Workspace Tree Full-Text Fuzzy Search and OmniSearch Integration Plan"
status: "Completed"
repositories: ["aico", "aico-specs"]
derives_from: ["spec-ft-core-omni-search-fulltext"]
created_at: "2026-10-09T00:16:00Z"
---

# Implementation Plan: Workspace Tree Full-Text Fuzzy Search and OmniSearch Integration

## 1. Technical Architecture & Decisions

- **Client-Side Full-Text Fuzzy Engine (`searchEngine.ts`)**:
  - Implement a dedicated module in `frontend/src/utils/searchEngine.ts` that indexes entities across all document fields:
    - Epics: `id`, `title`, `executive_summary`, `brainstorm_content`, `deepdive_content`, `ubiquitous_language`.
    - Features: `id`, `name`, `description`, `in_scope`, `out_of_scope`, `requirements` (`id`, `statement`), `tasks` (`id`, `title`, `description`), `brainstorm_content`, `deepdive_content`.
    - Specifications: full markdown content (`raw_spec_content`, requirement statements, and in-scope text) across all epics and standalone features.
    - ADRs: `id`, `title`, `context`, `decision`, `consequences`.
    - Code Files: file path string and backend search hit snippets.
  - Compute a combined match score evaluating:
    1. Exact token and prefix matches.
    2. Approximate string matching using Levenshtein distance ($\le 2$ edits for words $> 4$ characters).
    3. Acronym / camelCase / kebab-case sub-string matches.
  - Return matched items with match details (e.g. matched field: title, task, summary, requirement) and match score.

- **Dual-Match for Specifications (`searchEngine.ts`)**:
  - When querying terms inside a specification document, the search engine returns both the Specification (`spec-...` under `scope: 'specs'`) and the parent Feature (`ft-...` under `scope: 'features'`).
  - Standalone features' specifications are explicitly aggregated into `allSpecs` and the search index.

- **Backend Codebase Scanner & Live Annotation Discovery (`aico-server`)**:
  - In `crates/aico-server/src/server.rs`, implement `scan_codebase_files()` traversing `proj.codebase_path` while ignoring `.git`, `target`, `node_modules`, and binary files.
  - Parse files for `@implements spec-...::REQ-...` annotations to extract symbol names and line numbers.
  - Expose `GET /api/v1/code/files` returning the list of discovered files, annotation counts, and symbol details.
  - Wire into `ShellLayout.tsx` so the Code tree section displays live project files with `@implements` count badges instead of a hardcoded 12-file mock list.

- **Complete File Streaming in `/api/v1/raw` & Live CodeViewer (`server.rs`, `CodeViewer.tsx`)**:
  - In `get_raw_file_handler`, if `path` is not found relative to `specs_path`, resolve against `codebase_path`.
  - Update `CodeViewer.tsx` to asynchronously fetch the complete source file from `/api/v1/raw?path=...` and display full line counts and content, retiring `mockCodeSnippets`.

- **Backend Full-Text Codebase Search (`server.rs`, `OmniSearchModal.tsx`)**:
  - In `search_handler` (`GET /api/v1/search`), when query is present, scan text files across `codebase_path` for matching terms.
  - Return code hits with line numbers, relative path, and matched context snippets.
  - Wire code hits into `OmniSearchModal.tsx` when `scopes.code` is active.

---

## 2. Task Graph (DAG)

### Phase 1: Core Fuzzy & Full-Text Search Engine

- **TASK-CORE-009-1**: Implement In-Memory Full-Text Fuzzy Search Engine
  - **Status**: Done
  - **Addresses**: REQ-001, REQ-002
  - **Blocked by**: None (Can start immediately)
  - **What it delivers**:
    - Implement `frontend/src/utils/searchEngine.ts` supporting full-text indexing across all associated document fields (executive summaries, brainstorms, deep dives, requirements, tasks, ADRs, glossary).
    - Implement fuzzy matching with typo tolerance (Levenshtein distance $\le 2$ or token similarity).
  - **Verification Seam**: `frontend/tests/searchEngine.test.ts`

### Phase 2: Sidebar Tree Fuzzy Filtering & Auto-Expansion

- **TASK-CORE-009-2**: Integrate Fuzzy Search & Section Auto-Expansion into ShellLayout
  - **Status**: Done
  - **Addresses**: REQ-003, REQ-004
  - **Blocked by**: TASK-CORE-009-1
  - **What it delivers**:
    - Update `ShellLayout.tsx` to utilize `searchEngine.ts` over available entity metadata and associated documents.
    - Auto-expand accordion sections containing matching items.
    - Filter strictly according to active scope checkboxes.
  - **Verification Seam**: `frontend/tests/ShellLayoutSearch.test.tsx`

### Phase 3: Global OmniSearch Integration & Shortcut Interception

- **TASK-CORE-009-3**: Mount Live Cross-Project OmniSearch Modal & Intercept Browser Shortcuts
  - **Status**: Done
  - **Addresses**: REQ-005, REQ-006, REQ-007
  - **Blocked by**: TASK-CORE-009-1
  - **What it delivers**:
    - Add global keydown listener in `App.tsx` intercepting `Ctrl+K` with `e.preventDefault()`.
    - Add clickable Search button with shortcut pill in header.
    - Wire live cross-project search results into `OmniSearchModal.tsx`.
    - Implement selection navigation with project switching.
  - **Verification Seam**: `frontend/tests/OmniSearchModal.test.tsx`

### Phase 4: Backend REST Search Endpoint

- **TASK-CORE-009-4**: Implement Backend Filesystem Search Endpoint in Axum Server
  - **Status**: Done
  - **Addresses**: REQ-008
  - **Blocked by**: None (Can start immediately)
  - **What it delivers**:
    - Implement `GET /api/v1/search` handler in `crates/aico-server/src/server.rs`.
    - Scan discovered epics, features, and specs on disk matching query string.
  - **Verification Seam**: `crates/aico-server/tests/search_test.rs`

### Phase 5: Verification & Blueprint Compliance

- **TASK-CORE-009-5**: Full Suite Verification and Artifact Blueprint Conformance
  - **Status**: Done
  - **Addresses**: REQ-001 through REQ-008
  - **Blocked by**: TASK-CORE-009-2, TASK-CORE-009-3, TASK-CORE-009-4
  - **What it delivers**:
    - Run Vitest suite across frontend.
    - Run Cargo test suite across crates.
    - Run `verify-artifacts.mjs` ensuring 0 errors and 0 warnings.
  - **Verification Seam**: Vitest + Cargo + `verify-artifacts.mjs`

### Phase 6: Codebase Search, Annotation Discovery & Spec Full-Text Indexing

- **TASK-CORE-009-6**: Dual-Match Spec Full-Text Indexing in Client Search Engine
  - **Status**: Done
  - **Addresses**: REQ-002, REQ-009
  - **Blocked by**: None (Can start immediately)
  - **What it delivers**:
    - In `frontend/src/utils/searchEngine.ts`, attach full markdown content (`raw_spec_content`, requirement statements, and in-scope text) to `specs` scope entities.
    - Ensure standalone features' specs are included in `allSpecs` in `ShellLayout.tsx` and indexed.
    - When querying terms inside a specification, assert both the Specification (`spec-...`) and parent Feature (`ft-...`) match.
  - **Verification Seam**: `frontend/tests/searchEngine.test.ts`

- **TASK-CORE-009-7**: Implement Codebase File Scanner & `@implements` Discovery Endpoint
  - **Status**: Done
  - **Addresses**: REQ-010
  - **Blocked by**: None (Can start immediately)
  - **What it delivers**:
    - In `crates/aico-server/src/server.rs`, implement codebase scanner walking `codebase_path` with ignore filters (`.git`, `target`, `node_modules`, binary files) and identifying files with `@implements` annotations.
    - Expose `GET /api/v1/code/files` endpoint returning discovered files, symbol names, and line numbers.
    - Wire live files and `@implements` badges into the tree view Code section in `ShellLayout.tsx`.
  - **Verification Seam**: `crates/aico-server/tests/code_scanner_test.rs`, `frontend/tests/ShellLayoutSearch.test.tsx`

- **TASK-CORE-009-8**: Full-File Streaming via `/api/v1/raw` & Live CodeViewer Component
  - **Addresses**: REQ-011
  - **Status**: Done
  - **Blocked by**: TASK-CORE-009-7
  - **What it delivers**:
    - Update `get_raw_file_handler` in `crates/aico-server/src/server.rs` to resolve relative file paths against `codebase_path` when not found in `specs_path`.
    - Update `CodeViewer.tsx` to asynchronously fetch full file content from `/api/v1/raw?path=...` and render complete line-numbered source code.
  - **Verification Seam**: `frontend/tests/CodeViewer.test.tsx`

- **TASK-CORE-009-9**: Backend Full-Text Codebase Search & OmniSearch Integration
  - **Addresses**: REQ-012
  - **Status**: Done
  - **Blocked by**: TASK-CORE-009-6, TASK-CORE-009-7
  - **What it delivers**:
    - In `crates/aico-server/src/server.rs`, enhance `search_handler` to search source file contents across `codebase_path` for query matches, returning file path, line numbers, and snippets.
    - Connect backend code hits to `OmniSearchModal.tsx` under `scope: 'code'`.
  - **Verification Seam**: `crates/aico-server/tests/search_test.rs`, `frontend/tests/OmniSearchModal.test.tsx`

- **TASK-CORE-009-10**: Full Suite Regression Verification & Blueprint Conformance
  - **Addresses**: REQ-001 through REQ-012
  - **Status**: Done
  - **Blocked by**: TASK-CORE-009-8, TASK-CORE-009-9
  - **What it delivers**:
    - Run Vitest suite across frontend.
    - Run Cargo test suite across crates.
    - Run `verify-artifacts.mjs` ensuring 0 errors and 0 warnings.
  - **Verification Seam**: Vitest + Cargo + `verify-artifacts.mjs`

### Phase 7: Sidebar Search Workspace Bar Clear Control

- **TASK-CORE-009-11**: Add Inline Clear (`X`) Button to Sidebar Workspace Search Bar
  - **Status**: Done
  - **Addresses**: `REQ-013`
  - **Blocked by**: `TASK-CORE-009-10`
  - **What it delivers**:
    - Renders an inline clear button (`X`, `data-testid="sidebar-search-clear-btn"`, `aria-label="Clear search"`) inside the `"Search workspace..."` input container in `frontend/src/components/layout/ShellLayout.tsx` when `searchQuery` is non-empty.
    - Clears `searchQuery` on button click or `Escape` keydown and restores the unfiltered accordion tree.
    - Adds component tests in `frontend/tests/ShellLayoutSearch.test.tsx` and rebuilds the frontend production bundle.
  - **Verification Seam**: `frontend/tests/ShellLayoutSearch.test.tsx`

---

## 3. Amendment Log

- **2026-10-17T15:50:00Z**: Appended Phase 6 (`TASK-CORE-009-6` through `TASK-CORE-009-10`) via `/aico-plan-amendment` to address spec full-text dual-matching (`REQ-002`, `REQ-009`), live codebase file discovery and `@implements` annotation scanning (`REQ-010`), complete file streaming in `/api/v1/raw` and `CodeViewer` (`REQ-011`), and backend full-text codebase search (`REQ-012`).
- **2026-10-29T22:28:30Z**: Appended Phase 7 (`TASK-CORE-009-11`) to add an inline clear (`X`) button and `Escape` key reset to the sidebar `"Search workspace..."` input bar (`REQ-013`).
