---
id: spec-ft-core-workspace-path-normalization
type: spec
aico_skills_version: "0.1.2"
epic: "ec-core"
order: 14
title: "Workspace Path Normalization & Architecture Topology View"
status: "Completed"
derives_from: ["brain-ec-core", "spec-ft-core-code-navigation-and-search"]
created_at: "2026-10-17T18:25:00Z"
---

# Feature Specification: Workspace Path Normalization & Architecture Topology View

## 1. Problem Statement & User Value

In the AICO local-first platform, code files can be inspected and opened from multiple navigation entry points: the sidebar Code tree view, inline `@implements` badges in `CodeViewer`, implementation cards and task links in `FeatureViewer`, and aggregated lists in `EpicHubView`.

Currently, these navigation pathways pass inconsistent path formats:
1. **Sidebar Code Tree**: Opens files using repository-relative paths (e.g. `crates/aico-server/src/server.rs`).
2. **Feature & Epic Views**: Open files using full host filesystem paths (e.g. `d:/aico/codebase/aico/crates/aico-server/src/server.rs`).
3. **Tab Manager Redundancy**: Because `App.tsx` directly uses the incoming file string as the tab `id`, opening the same file from the sidebar and from a feature creates two separate duplicate tabs.
4. **Undocumented Workspace Root**: The platform workspace root `d:/aico`—which hosts all multi-project specifications (`*-specs/`) and all codebase repositories (`codebase/*/`)—is not documented or visible in the `SettingsModal` UI.

This feature eliminates path divergence by:
- Normalizing all code file paths across the backend and frontend relative to the workspace root `d:/aico` (e.g. `codebase/aico/crates/aico-server/src/server.rs`).
- Updating the `/api/v1/raw` endpoint to resolve workspace-relative paths against the workspace root while preserving backward compatibility.
- Normalizing tab IDs in `App.tsx` to prevent duplicate open tabs for the same file.
- Adding a dedicated "Workspace Architecture & Topology" card in `SettingsModal` clearly documenting `d:/aico` as the workspace root.

---

## In-Scope

- Generating canonical workspace-relative paths (`codebase/<project>/...`) in `scan_codebase_files`.
- Multi-prefix resolution in `/api/v1/raw` supporting workspace-relative, repo-relative, specs-relative, and absolute paths.
- Emitting canonical workspace-relative paths from `ShellLayout`, `FeatureViewer`, and `EpicHubView`.
- Input path normalization in `App.tsx` (`handleOpenCode`) to unify tab IDs and deduplicate open tabs.
- "Workspace Architecture & Topology" card in `SettingsModal` detailing the workspace root `d:/aico`, `*-specs/`, and `codebase/*/`.
- Integration and unit tests across backend scanner and frontend components.

---

## Out-of-Scope

- Modifying the underlying disk directory structure on host machine.
- Moving specs or codebase folders to arbitrary external drives.
- Editing source code files directly in the browser.

---

## 2. User Stories

1. **US-01** (Priority: P1): As a developer opening a code file from the sidebar Code tree view or from a feature card, I want both navigation actions to open the file with the same workspace-relative path (`codebase/aico/...`), so that my active editor tab is unified and never duplicated.
   - *Independent Test*: Click a file in the sidebar Code tree; then navigate to a feature and click the same code implementation; verify that only a single tab is open and active.
2. **US-02** (Priority: P1): As a developer inspecting a code tab header, I want to see the canonical workspace path (`codebase/aico/...`), so that I immediately understand where the file resides in the overall workspace topology.
   - *Independent Test*: Open any code file; verify the header in `CodeViewer` displays `codebase/aico/...`.
3. **US-03** (Priority: P2): As an engineer checking system settings in `SettingsModal`, I want to see a clear description of the workspace root and multi-project directory layout, so that I understand how specifications and codebases are structured under `d:/aico`.
   - *Independent Test*: Open `SettingsModal`; verify the "Workspace Architecture & Topology" card is displayed with `d:/aico`, `d:/aico/*-specs/`, and `d:/aico/codebase/*/`.

---

## 3. Requirements (EARS)

### REQ-001: Ubiquitous Workspace Root Resolution (Backend)
When `scan_codebase_files` is executed by the server, the server shall generate `CodeFileInfo.path` normalized relative to the workspace root directory (`d:/aico`, e.g. `codebase/aico/crates/...`), while preserving `relative_path` relative to the codebase repository root (`crates/...`).

### REQ-002: Multi-Prefix Raw File Streaming (Backend)
When a request is received at `/api/v1/raw?path=...`, the server shall attempt to resolve the path sequentially against the workspace root directory (`d:/aico`), against the active project specifications root (`*-specs`), against the project codebase root, and as an absolute path, returning HTTP 200 with raw file content upon resolution.

### REQ-003: Sidebar Tree Path Alignment (Frontend)
When rendering and selecting items in the sidebar Code tree view, `ShellLayout` shall display and emit the canonical workspace-relative path (`c.path`, e.g. `codebase/aico/...`) so that selected files match the unified workspace root convention.

### REQ-004: Feature & Epic Code Link Alignment (Frontend)
When a user clicks an implementation file link or task-linked code file in `FeatureViewer` or `EpicHubView`, the interface shall emit the canonical workspace-relative path (`f.path`, e.g. `codebase/aico/...`) to `onSelectCode`.

### REQ-005: Tab ID Deduplication & Path Normalization (Frontend)
When `handleOpenCode` in `App.tsx` receives a file path string, the application shall normalize the path by stripping host root prefixes (such as `d:/aico/`, `D:/aico/`, leading forward or backward slashes) and standardizing path separators to forward slashes before assigning tab `id` and `path`, preventing duplicate tabs for the same file.

### REQ-006: Workspace Root & Topology Settings Display (Frontend)
While the Settings modal is open, `SettingsModal` shall render a "Workspace Architecture & Topology" card displaying `d:/aico` as the Workspace Root, documenting that it contains all project specifications (`d:/aico/*-specs/`) and all codebase repositories (`d:/aico/codebase/*/`).

---

## 4. Designated Public Test Seams

| Requirement | Primary Test Seam | Verification Method |
|---|---|---|
| `REQ-001`, `REQ-002` | `crates/aico-server/tests/code_scanner_test.rs` | Rust integration test with reqwest querying `/api/v1/code/files` and `/api/v1/raw` |
| `REQ-003` | `frontend/tests/ShellLayoutSearch.test.tsx` | Vitest testing sidebar tree rendering, selection, and click callbacks |
| `REQ-004` | `frontend/tests/FeatureViewer.test.tsx`, `frontend/tests/EpicHubView.test.tsx` | Vitest testing code link clicks in FeatureViewer and EpicHubView |
| `REQ-005` | `frontend/tests/CodeViewer.test.tsx` | Vitest testing tab path rendering and normalized path fetching |
| `REQ-006` | `frontend/tests/SettingsModal.test.tsx` | Vitest testing presence of Workspace Architecture & Topology card and paths |
