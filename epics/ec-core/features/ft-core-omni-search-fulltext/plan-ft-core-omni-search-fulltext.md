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
    - ADRs: `id`, `title`, `context`, `decision`, `consequences`.
    - Code Files: file path string.
  - Compute a combined match score evaluating:
    1. Exact token and prefix matches.
    2. Approximate string matching using Levenshtein distance ($\le 2$ edits for words $> 4$ characters).
    3. Acronym / camelCase / kebab-case sub-string matches.
  - Return matched items with match details (e.g. matched field: title, task, summary, requirement) and match score.

- **Sidebar Tree Auto-Expansion & Highlighting (`ShellLayout.tsx`)**:
  - Accept full entity data or search engine results in `ShellLayout.tsx`.
  - When `searchQuery` is non-empty:
    - Evaluate active items against the fuzzy search engine.
    - Automatically expand all accordion sections (`openSections`) that contain matching entities so hits are immediately visible.
    - Render hit badges or matching indicators (e.g., small highlight or hit badge indicating matching task/summary).
    - Respect the 5 active scope toggles (`Epics`, `Features`, `Specs`, `Code`, `ADRs`).

- **Global OmniSearch Modal & Cross-Project Search (`App.tsx`, `OmniSearchModal.tsx`)**:
  - Attach a global `keydown` event listener in `App.tsx` on `window`:
    - On `(e.ctrlKey || e.metaKey) && (e.key === 'k' || e.key === 'K')`:
      Call `e.preventDefault()` to stop Google Chrome from focusing the address bar, and toggle `isOmniSearchOpen`.
  - Add a clickable Search button/trigger with `Ctrl+K` hint in the top application header and the sidebar workspace header.
  - Feed live `fullEpics`, `fullFeatures`, and `fullAdrs` across all registered projects into `OmniSearchModal`, displaying a project badge on each result.
  - On selecting a result item, if the item belongs to another project, invoke `setActiveProject(item.projectId)` before opening the corresponding tab and closing the modal.

- **Backend REST Search Endpoint (`aico-server`)**:
  - Add `GET /api/v1/search?q=<query>&project=<id>` in `crates/aico-server/src/server.rs`.
  - Scan specs and codebase directories across registered projects (or filtered to specific project) for matching files and text snippets, returning structured JSON results.

---

## 2. Task Graph (DAG)

### Phase 1: Core Fuzzy & Full-Text Search Engine

- **TASK-CORE-009-1**: Implement In-Memory Full-Text Fuzzy Search Engine
  - **Status**: Done
  - **Addresses**: REQ-001, REQ-002
  - **Blocked by**: None (Can start immediately)
  - **What it delivers**:
    - Implement `frontend/src/utils/searchEngine.ts` supporting full-text indexing across all associated document fields (executive summaries, brainstorms, deep dives, requirements, tasks, ADRs, glossary).
    - Implement fuzzy matching with typo tolerance (Levenshtein distance $\le 2$ or token similarity) so `ft-cor-host-runtime` matches `ft-core-host-runtime`.
    - Unit seam tests in `frontend/tests/searchEngine.test.ts`.
  - **Verification Seam**: `frontend/tests/searchEngine.test.ts`

### Phase 2: Sidebar Tree Fuzzy Filtering & Auto-Expansion

- **TASK-CORE-009-2**: Integrate Fuzzy Search & Section Auto-Expansion into ShellLayout
  - **Status**: Done
  - **Addresses**: REQ-003, REQ-004
  - **Blocked by**: TASK-CORE-009-1
  - **What it delivers**:
    - Update `ShellLayout.tsx` to utilize `searchEngine.ts` over available entity metadata and associated documents.
    - When search query is entered, auto-expand accordion sections containing matching items.
    - Display match indicators on matching nodes in the tree.
    - Filter strictly according to active scope checkboxes.
  - **Verification Seam**: `frontend/tests/ShellLayoutSearch.test.tsx`

### Phase 3: Global OmniSearch Integration & Shortcut Interception

- **TASK-CORE-009-3**: Mount Live Cross-Project OmniSearch Modal & Intercept Browser Shortcuts
  - **Status**: Done
  - **Addresses**: REQ-005, REQ-006, REQ-007
  - **Blocked by**: TASK-CORE-009-1
  - **What it delivers**:
    - Add global keydown listener on `window` in `App.tsx` intercepting `Ctrl+K` with `e.preventDefault()`.
    - Add a clickable Search button with `Ctrl+K` shortcut pill in the top header and sidebar header.
    - Wire live cross-project search results into `OmniSearchModal.tsx` showing project tags.
    - Implement selection navigation with automatic active project switching (`setActiveProject`) when choosing an entity from another project.
  - **Verification Seam**: `frontend/tests/OmniSearchModal.test.tsx`

### Phase 4: Backend REST Search Endpoint

- **TASK-CORE-009-4**: Implement Backend Filesystem Search Endpoint in Axum Server
  - **Status**: Done
  - **Addresses**: REQ-008
  - **Blocked by**: None (Can start immediately)
  - **What it delivers**:
    - Implement `GET /api/v1/search` handler in `crates/aico-server/src/server.rs` querying across registered projects.
    - Scan discovered epics, features, and specs on disk matching query string.
    - Integration seam test in `crates/aico-server/tests/search_test.rs`.
  - **Verification Seam**: `crates/aico-server/tests/search_test.rs`

### Phase 5: Verification & Blueprint Compliance

- **TASK-CORE-009-5**: Full Suite Verification and Artifact Blueprint Conformance
  - **Status**: Done
  - **Addresses**: REQ-001 through REQ-008
  - **Blocked by**: TASK-CORE-009-2, TASK-CORE-009-3, TASK-CORE-009-4
  - **What it delivers**:
    - Run Vitest suite across frontend.
    - Run Cargo test suite across crates.
    - Run `verify-artifacts.mjs` ensuring 0 errors and 0 warnings across all 51 artifacts.
  - **Verification Seam**: Vitest + Cargo + `verify-artifacts.mjs`
