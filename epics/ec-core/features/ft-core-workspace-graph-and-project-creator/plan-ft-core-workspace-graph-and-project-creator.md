---
id: plan-ft-core-workspace-graph-and-project-creator
type: plan
aico_skills_version: "0.1.2"
epic: "ec-core"
order: 17
title: "Workspace UI Refinement, Project Creator & Interactive Lineage Graph Implementation Plan"
status: "Completed"
repositories: ["aico"]
derives_from: ["spec-ft-core-workspace-graph-and-project-creator"]
created_at: "2026-10-23T19:05:21.133Z"
---

# Implementation Plan: Workspace UI Refinement, Project Creator & Interactive Lineage Graph

## 1. Technical Architecture & Decisions

- **Target repositories**: `aico`.
- **Sidebar & Epic Hub Refinements**:
  - In `frontend/src/components/layout/ShellLayout.tsx`, remove the `w-2.5 h-2.5 rounded-full bg-indigo-500 animate-pulse` div from the sidebar header.
  - In `frontend/src/features/epic-hub/EpicHubView.tsx`, remove the `View Feature` button from each feature card in `Task Plan`, turn the `{feat.id}` badge into an interactive button calling `onOpenFeature(feat.id, feat.name)`, and add per-feature collapse state that defaults to collapsed (`true`) when `featPct === 100 && featTotal > 0` (or `feat.status === 'Completed' || feat.status === 'Done'`) and expanded (`false`) otherwise.
- **Project Scaffolding (`POST /api/v1/projects`)**:
  - Add `CodebaseWizard::scaffold_project_specs(workspace_root, project_name)` in `crates/aico-core/src/wizard.rs` creating `<project-name>-specs/` with `.aico/`, `adr/`, `epics/`, `features/`, `.aico-project.json`, and `glossary.md`, and initializing Git (`git init`).
  - Expose `POST /api/v1/projects` in `crates/aico-server/src/server.rs` and wire `+ New Project` UI in `frontend/src/components/navigation/ProjectSwitcher.tsx`.
- **Persistent Property Graph & Interactive Lineage Canvas**:
  - Upgrade `AicoGraphStore` in `crates/aico-storage/src/graph.rs` to store typed nodes (`id`, `label`, `node_type`, `properties`) and directed edges (`source`, `target`, `relation`), persist them to `.aico/storage/graph/lineage_graph.json`, and evaluate Cypher `CREATE`, `MERGE`, and `MATCH` statements as well as multi-hop BFS traversal.
  - Add `GET /api/v1/graph` in `crates/aico-server/src/server.rs` that constructs and synchronizes the full artifact and code lineage graph (`ADR` -> `Epic` -> `Feature` -> `Spec` / `Plan` -> `Requirement` / `Task` -> `Code`) and returns structured `nodes` and `edges`.
  - Upgrade `frontend/src/features/epic-hub/VisualLineageCanvas.tsx` and `frontend/src/App.tsx` to fetch live graph data for the active epic and render a multi-column SVG-connected DAG with interactive upstream/downstream highlighting and node navigation.

---

## 2. Task Graph (DAG)

### Phase 1: Sidebar Dot Removal & Epic Task Plan Collapsible Features

- **TASK-CORE-017-1**: Remove Sidebar Header Blinking Dot & Upgrade Epic Task Plan Feature Cards
  - **Status**: Done
  - **Addresses**: `REQ-001`, `REQ-002`, `REQ-003`, `REQ-007`, `REQ-008`
  - **Blocked by**: None (Can start immediately)
  - **What it delivers**:
    - Removes the decorative blue blinking dot next to `AICO WORKSPACE` in `ShellLayout.tsx`.
    - Removes the `View Feature` button in `EpicHubView.tsx` `Task Plan`, makes the `ft-*` tag clickable to open the feature, adds collapse/expand toggles for each feature in `Task Plan`, and auto-collapses features that are 100% implemented.
  - **Verification Seam**: `frontend/tests/EpicHubView.test.tsx`

### Phase 2: Project Creator Backend & ProjectSwitcher UI

- **TASK-CORE-017-2**: Implement Project Specs Scaffolder & `POST /api/v1/projects` Endpoint
  - **Status**: Done
  - **Addresses**: `REQ-010`, `REQ-012`
  - **Blocked by**: None (Can start immediately)
  - **What it delivers**:
    - Implements `CodebaseWizard::scaffold_project_specs` in `crates/aico-core/src/wizard.rs` creating `<project-name>-specs` (`.aico`, `adr`, `epics`, `features`, `.aico-project.json`, `glossary.md`, and `git init`).
    - Exposes `POST /api/v1/projects` in `crates/aico-server/src/server.rs` to create, register, and activate the new project.
  - **Verification Seam**: `crates/aico-server/tests/project_api_test.rs`

- **TASK-CORE-017-3**: Add Create New Project Workflow in `ProjectSwitcher.tsx`
  - **Status**: Done
  - **Addresses**: `REQ-009`
  - **Blocked by**: `TASK-CORE-017-2`
  - **What it delivers**:
    - Adds `+ New Project` button and project name input form in `ProjectSwitcher.tsx` that posts to `/api/v1/projects`, updates the project list, and activates the new project.
  - **Verification Seam**: `frontend/tests/ProjectSwitcher.test.tsx`

### Phase 3: Pure-Rust AicoGraphStore Engine & Interactive Lineage Graph

- **TASK-CORE-017-4**: Implement Persistent `AicoGraphStore` & `GET /api/v1/graph` Lineage Builder
  - **Status**: Done
  - **Addresses**: `REQ-004`, `REQ-005`
  - **Blocked by**: None (Can start immediately)
  - **What it delivers**:
    - Replaces the stub in `crates/aico-storage/src/graph.rs` with a real persistent property graph supporting Cypher node/edge creation, `MATCH` queries, and multi-hop lineage traversal in `crates/aico-mcp/src/graph_tools.rs`.
    - Adds `GET /api/v1/graph` in `crates/aico-server/src/server.rs` with case-insensitive ADR ID normalization (`ADR-0004`), `deepdive-ec-*.md` scanning, and rolled-up `ADR -> Epic`, `Spec -> Plan`, and `Plan -> Code` edges so the full hierarchy connects cleanly.
  - **Verification Seam**: `crates/aico-storage/tests/storage_hybrid_test.rs`

- **TASK-CORE-017-5**: Upgrade `VisualLineageCanvas.tsx` with 6-Column Artifact DAG, SVG Edges & Reqs/Tasks Toggle
  - **Status**: Done
  - **Addresses**: `REQ-006`, `REQ-011`
  - **Blocked by**: `TASK-CORE-017-4`
  - **What it delivers**:
    - Updates `VisualLineageCanvas.tsx` and `App.tsx` to load the real artifact graph from `/api/v1/graph?epic=<id>`, render 6 layered columns (`ADR` -> `Epic / Brainstorm / Deep-Dive` -> `Feature` -> `Spec` -> `Plan` -> `Code`) with SVG Bezier curves, collapse raw `REQ-*`/`TASK-*` nodes by default behind a `"Show Reqs & Tasks"` toggle button, highlight connected lineage paths on click, and rebuild `frontend/dist`.
  - **Verification Seam**: `frontend/tests/EpicHubView.test.tsx`

### Phase 4: Task Plan Collapse Persistence, Multi-Select Graph Filters & Full-Window Canvas

- **TASK-CORE-017-6**: Persist Epic Task Plan Feature Collapse State in Accordion Store
  - **Status**: Done
  - **Addresses**: `REQ-013`
  - **Blocked by**: None (Can start immediately)
  - **What it delivers**:
    - Updates `useAccordionStore.ts` with `sessionStorage` persistence so collapse states survive page reloads.
    - Updates `EpicHubView.tsx` to read and toggle each feature's collapse state via `useAccordionStore` using default based on 100% completion.
    - Updates `useTabStore.ts` `closeTab` to reset the accordion entity state when an epic tab is closed.
  - **Verification Seam**: `frontend/tests/EpicHubView.test.tsx`

- **TASK-CORE-017-7**: Implement Multi-Select Ordered Filters, Dynamic Column Headers & Separate Node Open Button
  - **Status**: Done
  - **Addresses**: `REQ-014`, `REQ-015`
  - **Blocked by**: None (Can start immediately)
  - **What it delivers**:
    - Orders filter pills in `VisualLineageCanvas.tsx` to match the graph's architectural columns.
    - Adds multi-select filter capability with "All" highlighting and dynamically renders column headings only for tiers with visible nodes.
    - Changes node left-click to strictly highlight provenance/blast radius, and adds a dedicated open icon button in the top-right corner of each node to navigate to the artifact.
  - **Verification Seam**: `frontend/tests/VisualLineageCanvas.test.tsx`

- **TASK-CORE-017-8**: Add Canvas Scroll-to-Zoom & Full-Window Graph Tab in Main Workspace
  - **Status**: Done
  - **Addresses**: `REQ-016`
  - **Blocked by**: `TASK-CORE-017-7`
  - **What it delivers**:
    - Adds mouse wheel zoom handling (`onWheel`) to `VisualLineageCanvas.tsx`.
    - Adds popout button to the graph card header and supports `type: 'graph'` tabs in `useTabStore.ts` and `App.tsx` for full-window edge-to-edge viewing.
  - **Verification Seam**: `frontend/tests/VisualLineageCanvas.test.tsx`

### Phase 5: Feature-Scoped Lineage Drill-down & Clean Epic Overview

- **TASK-CORE-017-9**: Implement Feature-Scoped Lineage Drill-down with Dedicated REQ/TASK Columns and Clean Epic Overview
  - **Status**: Done
  - **Addresses**: `REQ-017`, `REQ-018`
  - **Blocked by**: `TASK-CORE-017-8`
  - **What it delivers**:
    - Updates `VisualLineageCanvas.tsx` to display pure 6 architectural columns by default (`ADR` -> `Epic` -> `Feature` -> `Specification` -> `Plan` -> `Code`), removing `brainstorm`/`deepdive` from the Epic column.
    - Implements per-feature drill-down expanding only that feature's child requirements and tasks into separate `Requirement` and `Task` columns between `Specification` and `Plan`.
    - Removes global 316-task dumping and provides a reset button back to the Epic overview.
  - **Verification Seam**: `frontend/tests/VisualLineageCanvas.test.tsx`

### Phase 6: Invertible Filter Selection & Feature-Anchored Lineage Simplification

- **TASK-CORE-017-10**: Implement Invertible Multi-Select Filter Logic in `VisualLineageCanvas.tsx`
  - **Status**: Done
  - **Addresses**: `REQ-019`
  - **Blocked by**: `TASK-CORE-017-9`
  - **What it delivers**:
    - Adds tri-state selection handling (`isAllSelected`, empty view, and partial selection).
    - Clicking a type filter pill when "All" is selected unselects that specific type while retaining all others.
    - Clicking "All" when "All" is active toggles to an empty canvas view (`0` nodes).
    - Clicking any type pill from the empty view selects only that specific type.
  - **Verification Seam**: `frontend/tests/VisualLineageCanvas.test.tsx`

- **TASK-CORE-017-11**: Collapse Spec and Plan into Feature Node with Direct Lineage Edges
  - **Status**: Done
  - **Addresses**: `REQ-020`
  - **Blocked by**: `TASK-CORE-017-10`
  - **What it delivers**:
    - Updates `crates/aico-server/src/server.rs` to emit direct edges from `Feature` to `Requirement` (`specifies`), `Task` (`contains`), and `Code` (`verifies`).
    - Updates `VisualLineageCanvas.tsx` to omit intermediate `spec` and `plan` file nodes, establishing the canonical 6 tiers: `ADR` -> `Epic` -> `Feature` -> `Requirement` -> `Task` -> `Code`.
    - Default Epic overview renders 4 columns (`ADR` -> `Epic` -> `Feature` -> `Code`), expanding to 6 columns with dedicated `Requirement` and `Task` tiers when drilled down.
  - **Verification Seam**: `frontend/tests/VisualLineageCanvas.test.tsx` and `frontend/tests/EpicHubView.test.tsx`

### Phase 7: Requirement Omission & Adjacent-Tier Edge Enforcement

- **TASK-CORE-017-12**: Omit Requirement Nodes and Streamline Canonical 5-Tier Graph Pipeline
  - **Status**: Done
  - **Addresses**: `REQ-021`
  - **Blocked by**: `TASK-CORE-017-11`
  - **What it delivers**:
    - Updates `VisualLineageCanvas.tsx` to omit `requirement` nodes from `TIER_COLUMNS`, `TYPE_ORDER`, `availableTypes`, and `visibleNodes`.
    - Establishes canonical 5 tiers: `ADR` -> `Epic` -> `Feature` -> `Task` -> `Code`.
    - Feature drill-down expands only child tasks in a dedicated `Task` column between `Feature` and `Code`.
  - **Verification Seam**: `frontend/tests/VisualLineageCanvas.test.tsx`

- **TASK-CORE-017-13**: Enforce Adjacent-Tier Edge Rendering and Suppress Multi-Tier Bypass Edges
  - **Status**: Done
  - **Addresses**: `REQ-022`
  - **Blocked by**: `TASK-CORE-017-12`
  - **What it delivers**:
    - Restricts edge rendering in `VisualLineageCanvas.tsx` so directed edges are only drawn between nodes in immediately adjacent active visible columns ($j = i + 1$).
    - When `Task` is visible, renders `Feature -> Task` and `Task -> Code`, while suppressing the `Feature -> Code` bypass edge.
    - When `Task` is unselected or hidden, renders `Feature -> Code` directly.
    - Updates test suites to verify adjacent edge filtering.
  - **Verification Seam**: `frontend/tests/VisualLineageCanvas.test.tsx` and `frontend/tests/EpicHubView.test.tsx`

### Phase 8: Strict Task Header Parsing & Isolated Drill-down Traversal

- **TASK-CORE-017-14**: Strict Task Declaration Parsing in Lineage Graph Engine
  - **Status**: Done
  - **Addresses**: `REQ-023`
  - **Blocked by**: `TASK-CORE-017-13`
  - **What it delivers**:
    - Restricts task header recognition in `crates/aico-server/src/server.rs` strictly to lines starting with canonical task declaration prefixes (`- **TASK-`, `### TASK-`, `**TASK-`).
    - Prevents dependency notes (`- **Blocked by**: TASK-...`) and rewrite targets (`- **Rewrite Target**: TASK-...`) from declaring tasks or assigning cross-epic tasks to wrong epics.
    - Eliminates orphan code nodes from `GET /api/v1/graph?epic=ec-core`.
  - **Verification Seam**: `crates/aico-server/tests/server_test.rs`

- **TASK-CORE-017-15**: Isolated Downstream Traversal in VisualLineageCanvas Drilldown
  - **Status**: Done
  - **Addresses**: `REQ-024`
  - **Blocked by**: `TASK-CORE-017-14`
  - **What it delivers**:
    - Updates `drilledFeatureNodeIds` BFS in `VisualLineageCanvas.tsx` to strictly follow downstream directed edges from the drilled feature (`edge.source === current`).
    - Removes the backward traversal branch (`edge.target === current`) to eliminate cross-feature leakage through shared code verification seams.
    - Ensures drilling down into any feature (such as `ft-core-code-navigation-and-search`) renders only that feature, its child tasks, and its verified seams.
  - **Verification Seam**: `frontend/tests/VisualLineageCanvas.test.tsx`

### Phase 9: Multi-Epic Dashboard Graph Focus & Drilldown Isolation

- **TASK-CORE-017-16**: VisualLineageCanvas Epic Focus & Drilldown Isolation
  - **Status**: Done
  - **Addresses**: `REQ-025`
  - **Blocked by**: `TASK-CORE-017-15`
  - **What it delivers**:
    - Updates `frontend/src/features/epic-hub/VisualLineageCanvas.tsx` to add `drilledEpicId` state, toggle epic focus on epic node click, compute `drilledEpicNodeIds` (governing ADRs + focused epic + child features + verified seams), and filter visible nodes.
    - Adds an Epic Focus Selector (`data-testid="epic-focus-select"`) and a `✕ Reset to All Epics` button (`data-testid="reset-epic-focus-btn"`) when multiple epics are detected in `rawNodes`.
    - Updates status legend to reflect the active epic focus and feature drilldown hierarchy.
  - **Verification Seam**: `frontend/tests/VisualLineageCanvas.test.tsx`
