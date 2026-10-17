---
id: spec-ft-core-multirepo-code-path-normalization
type: spec
aico_skills_version: "0.1.2"
epic: "ec-core"
order: 16
title: "Multi-Repository Code Path Normalization & Root-Agnostic Resolution"
status: "Completed"
derives_from: ["brain-ec-core", "spec-ft-core-code-path-simplification-and-omnisearch"]
created_at: "2026-10-17T19:35:00Z"
---

# Feature Specification: Multi-Repository Code Path Normalization & Root-Agnostic Resolution

## 1. Problem Statement & User Value

In AICO workspaces, codebase repositories live under the `<root>/codebase/` container directory (e.g. `codebase/aico`, `codebase/aico-standalone`, `codebase/web-service`).

Two limitations in previous path handling impact multi-repo workspaces and platform portability:
1. **Repository Folder Name Loss**: Stripping both `codebase/` and `<repo>/` collapsed paths into repository-internal paths (e.g. `crates/aico-server/src/server.rs`). In a project comprising multiple subfolders or repositories under `codebase/` (e.g. `codebase/aico` and `codebase/aico-standalone`), common files (e.g. `src/main.rs`, `Cargo.toml`) collide and lose their repository namespace. Preserving the repository root folder while omitting only the redundant `codebase/` container (e.g. `aico/crates/aico-server/src/server.rs` and `aico-standalone/src/main.rs`) guarantees unambiguous multi-repository support.
2. **Hardcoded Root Path Fragility**: Path normalizers matching hardcoded patterns (such as `d:/aico/`) break when AICO is installed in custom root directories (e.g. `C:\Users\dev\workspace`, `/opt/aico-root`, etc.). Path normalization and resolution must dynamically anchor on the container boundary `/codebase/` rather than hardcoding folder names or drive letters.

This specification addresses both concerns by:
- Standardizing code paths to `<repo>/<subpath>` (e.g. `aico/crates/aico-server/src/server.rs`, `aico-standalone/src/main.rs`).
- Making frontend path normalization root-agnostic by dynamically slicing after `/codebase/`.
- Resolving raw files in the backend dynamically against `resolve_workspace_root().join("codebase").join(&query.path)`.
- Rendering `<repo>/<subpath>` consistently across OmniSearch, Code tree view, Feature Viewer code links, and CodeViewer tabs.

---

## In-Scope

- Emitting `<repo>/<subpath>` (e.g. `aico/crates/...`) in `scan_codebase_files`.
- Dynamically resolving `<repo>/<subpath>` against `resolve_workspace_root().join("codebase")` in `/api/v1/raw`.
- Emitting `<repo>/<subpath>` for code search hits in `/api/v1/search` (`title`, `id`, `path`).
- Indexing `<repo>/<subpath>` in `searchEngine.ts` for client-side search.
- Rendering `<repo>/<subpath>` in `OmniSearchModal` in font-mono style.
- Implementing dynamic root-agnostic normalization in `pathUtils.ts` (`normalizeWorkspacePath`).
- Aligning `ShellLayout`, `FeatureViewer`, `EpicHubView`, and `CodeViewer` to `<repo>/<subpath>`.

---

## Out-of-Scope

- Modifying host filesystem layout or moving repository folders.
- Changing search ranking algorithms or tokenizers in LanceDB.
- Modifying non-code search scopes (epics, specs, ADRs).

---

## 2. User Stories

1. **US-01** (Priority: P1): As a developer working in a multi-repository project with repositories `aico` and `aico-standalone` under `codebase/`, I want code paths to display as `aico/...` and `aico-standalone/...`, so that files with identical relative names in different repositories never collide.
   - *Independent Test*: Query `/api/v1/code/files` and verify discovered files start with repository prefix (e.g. `aico/crates/aico-server/src/server.rs`).
2. **US-02** (Priority: P1): As a developer running AICO in a custom installation directory (e.g. `C:\Users\dev\workspace` or `/opt/aico`), I want path normalization and raw file streaming to work without hardcoded `d:/aico` assumptions, so that the platform is 100% portable.
   - *Independent Test*: Pass paths from arbitrary root folders containing `/codebase/` to `normalizeWorkspacePath`; verify they cleanly normalize to `<repo>/<subpath>`.
3. **US-03** (Priority: P1): As a developer searching code via OmniSearch (`Ctrl+K`), I want search results to show `aico/crates/...` or `aico-standalone/...` in the title, so that I immediately see which repository and crate a matching file belongs to.
   - *Independent Test*: Open OmniSearch, search for code; verify hits display `<repo>/<subpath>`.

---

## 3. Requirements (EARS)

### REQ-001: Multi-Repository Code Path Emission (Backend)
When `scan_codebase_files` is executed by the server, the server shall generate `CodeFileInfo.path` and `CodeFileInfo.relative_path` formatted as `<repo>/<subpath>` (e.g. `aico/crates/aico-server/src/server.rs`), preserving the repository root folder while omitting the parent `codebase/` container.

### REQ-002: Dynamic Root-Agnostic Raw File Resolution (Backend)
When a request is received at `/api/v1/raw?path=...` with a path formatted as `<repo>/<subpath>`, the server shall resolve the path dynamically against `resolve_workspace_root().join("codebase").join(&query.path)` without hardcoding host drive letters or workspace directory names, returning HTTP 200 with raw file content.

### REQ-003: Multi-Repository Code Path in OmniSearch (Backend & Search Engine)
When `search_handler` or client-side `searchEngine.ts` indexes or returns code items, the system shall set `title`, `id`, and `path` to the multi-repo path `<repo>/<subpath>` (e.g. `aico/crates/aico-server/src/server.rs`).

### REQ-004: Root-Agnostic Dynamic Normalization in pathUtils (Frontend)
When `normalizeWorkspacePath` in `pathUtils.ts` receives any file path string, the function shall dynamically slice after the `/codebase/` container boundary (or strip leading `codebase/`), producing `<repo>/<subpath>` without hardcoding `d:/aico` or any specific drive letter or folder name.

### REQ-005: Universal View & Tab Consistency (Frontend)
When code files are displayed or navigated across `ShellLayout`, `FeatureViewer`, `EpicHubView`, `CodeViewer`, and tab management, all interfaces shall consistently format, render, and link code using `<repo>/<subpath>`.

---

## 4. Designated Public Test Seams

| Requirement | Primary Test Seam | Verification Method |
|---|---|---|
| `REQ-001`, `REQ-002` | `crates/aico-server/tests/code_scanner_test.rs` | Rust integration test asserting discovered paths and raw file requests start with `aico/` |
| `REQ-003` | `crates/aico-server/tests/search_test.rs`, `frontend/tests/searchEngine.test.ts` | Rust and Vitest tests verifying code search hit title matches `aico/...` |
| `REQ-004` | `frontend/tests/CodeViewer.test.tsx` | Vitest testing `normalizeWorkspacePath` with arbitrary root paths and `aico/...` |
| `REQ-005` | `frontend/tests/ShellLayoutSearch.test.tsx`, `frontend/tests/FeatureViewer.test.tsx`, `frontend/tests/OmniSearchModal.test.tsx` | Vitest testing code rendering and navigation with `aico/...` |
