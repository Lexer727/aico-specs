---
id: spec-ft-core-code-path-simplification-and-omnisearch
type: spec
aico_skills_version: "0.1.2"
epic: "ec-core"
order: 15
title: "Code Path Simplification & OmniSearch Consistency"
status: "Completed"
derives_from: ["brain-ec-core", "spec-ft-core-workspace-path-normalization"]
created_at: "2026-10-17T18:57:00Z"
---

# Feature Specification: Code Path Simplification & OmniSearch Consistency

## 1. Problem Statement & User Value

In the AICO WebUI and backend services, code file paths can be navigated and inspected from multiple viewpoints: the sidebar Code tree view, Feature Viewer implementation links, task cards, Epic Hub, and the global OmniSearch modal (`Ctrl+K`).

Two ergonomic issues currently impact code discovery and readability:
1. **Redundant `codebase/<project>/` Prefix**: Because all codebase repositories reside under `d:/aico/codebase/<project>/` and the active project is already bound in context, prefixing `codebase/<project>/` to every code file path creates unnecessary visual noise and overly verbose labels across tabs, cards, and headers. Standardizing on clean repository-relative paths (e.g. `crates/aico-server/src/server.rs`) improves visual clarity.
2. **Ambiguous Basename in OmniSearch**: In the global `OmniSearchModal`, code search hits currently display only the file basename (`server.rs`) in `item.title`. When searching common files (e.g. `mod.rs`, `server.rs`, `index.ts`), developers cannot discern the crate, package, or directory location from the search result title.

This specification addresses both issues by:
- Simplifying code paths across backend scanner, API endpoints, and frontend views to be repository-relative (e.g. `crates/aico-server/src/server.rs`).
- Updating OmniSearch in both backend (`search_handler`) and client-side indexer (`searchEngine.ts`) to emit and render the clean repository-relative path in the title line for code hits.
- Updating `normalizeWorkspacePath` in `pathUtils.ts` to strip `codebase/<project>/` and `codebase/` prefixes so tab IDs and paths are unified and deduplicated.

---

## In-Scope

- Emitting repository-relative paths (`crates/...`) in `scan_codebase_files`.
- Streaming raw files via repository-relative paths in `/api/v1/raw` while maintaining fallback for workspace and absolute paths.
- Emitting repository-relative paths for code hits in `/api/v1/search` (`title` and `path`).
- Indexing repository-relative paths in `searchEngine.ts` for code items (`title: filePath`).
- Rendering repository-relative paths in `OmniSearchModal` for all code search hits.
- Normalizing input paths in `pathUtils.ts` (`normalizeWorkspacePath`) to repository-relative format (`crates/...`).
- Verifying consistency across `ShellLayout`, `FeatureViewer`, `EpicHubView`, `CodeViewer`, and `OmniSearchModal`.

---

## Out-of-Scope

- Altering file storage directory structure on host filesystem.
- Changing search ranking algorithms or tokenizers in LanceDB.
- Modifying non-code search scopes (epics, specs, ADRs).

---

## 2. User Stories

1. **US-01** (Priority: P1): As a developer searching code via OmniSearch (`Ctrl+K`), I want code search results to display the relative file path (e.g. `crates/aico-server/src/server.rs`) in the title line, so that I can immediately identify the crate and module location of matching files.
   - *Independent Test*: Open OmniSearch, search for a symbol present in multiple files; verify that each code result displays its relative path rather than just the file basename.
2. **US-02** (Priority: P1): As a developer inspecting code files across the sidebar tree, feature implementations, and tabs, I want all code paths to omit the redundant `codebase/<project>/` prefix and display as clean repository-relative paths (e.g. `crates/aico-server/src/server.rs`), so that paths are concise and readable.
   - *Independent Test*: Inspect the sidebar Code tree view, Feature Viewer code links, and Code Viewer tabs; verify that paths display as `crates/...`.
3. **US-03** (Priority: P1): As a developer opening a code file from OmniSearch, the sidebar tree, or a feature card, I want all three pathways to resolve to the identical tab ID (`crates/...`), so that no duplicate editor tabs are created.
   - *Independent Test*: Open a file from OmniSearch, then from the sidebar tree; verify that only a single unified tab exists.

---

## 3. Requirements (EARS)

### REQ-001: Codebase-Relative Path Emission (Backend)
When `scan_codebase_files` is executed by the server, the server shall generate `CodeFileInfo.path` and `CodeFileInfo.relative_path` as repository-relative paths relative to the active project's codebase root (e.g. `crates/aico-server/src/server.rs`), leaving `codebase/` and project container prefix out.

### REQ-002: Multi-Prefix Raw File Streaming Compatibility (Backend)
When a request is received at `/api/v1/raw?path=...`, the server shall resolve the path sequentially against the active project codebase root, against the workspace root (`d:/aico`), against the active project specifications root, and as an absolute path, returning HTTP 200 with raw file content upon resolution.

### REQ-003: Consistent Code Path in OmniSearch (Backend & Search Engine)
When `search_handler` or client-side `searchEngine.ts` indexes or returns code items, the system shall set `title`, `id`, and `path` to the repository-relative path (e.g. `crates/aico-server/src/server.rs`) rather than only the file basename (`server.rs`).

### REQ-004: OmniSearch Modal Path Rendering (Frontend)
When rendering search results in `OmniSearchModal`, the interface shall display the repository-relative path (`crates/aico-server/src/server.rs`) in the title line for all code search hits, preserving matched line snippets in the subtitle.

### REQ-005: Universal Path Normalization in pathUtils (Frontend)
When `normalizeWorkspacePath` in `pathUtils.ts` receives a file path string, the function shall strip host drive letters, `d:/aico/`, `codebase/<project>/`, `codebase/`, and leading slashes, returning the clean repository-relative path (e.g. `crates/...`) for unified tab IDs and tab deduplication.

### REQ-006: Unified View Consistency (Frontend)
When code files are rendered or clicked across `ShellLayout`, `FeatureViewer`, `EpicHubView`, and `CodeViewer`, all interfaces shall consistently display and emit repository-relative paths (`crates/...`).

---

## 4. Designated Public Test Seams

| Requirement | Primary Test Seam | Verification Method |
|---|---|---|
| `REQ-001`, `REQ-002` | `crates/aico-server/tests/code_scanner_test.rs` | Rust integration test querying `/api/v1/code/files` and `/api/v1/raw` with repository-relative paths |
| `REQ-003` | `crates/aico-server/tests/search_test.rs`, `frontend/tests/searchEngine.test.ts` | Rust and Vitest tests verifying code search hit title matches repository-relative path |
| `REQ-004` | `frontend/tests/OmniSearchModal.test.tsx` | Vitest testing code hit title rendering with relative path |
| `REQ-005` | `frontend/tests/CodeViewer.test.tsx` | Vitest testing `normalizeWorkspacePath` stripping `codebase/<project>/` |
| `REQ-006` | `frontend/tests/ShellLayoutSearch.test.tsx`, `frontend/tests/FeatureViewer.test.tsx` | Vitest testing sidebar tree and feature view code link emission |
