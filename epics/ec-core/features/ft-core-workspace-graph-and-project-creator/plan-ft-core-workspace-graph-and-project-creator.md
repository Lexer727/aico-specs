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
- **Multi-Tab Project State Isolation & Request Resolution**:
  - Implement an Axum extractor/helper `ProjectContext` in `crates/aico-server/src/server.rs` that resolves the project context from `project` / `project_id` query parameters, then `X-Aico-Project` request header, with fallback to `WorkspaceRegistry.get_active()`.
  - Wire `ProjectContext` into all workspace REST endpoints (`/api/v1/epics`, `/api/v1/features`, `/api/v1/adrs`, `/api/v1/glossary`, `/api/v1/code/files`, `/api/v1/git/status`, `/api/v1/projects/manifest`).
  - Update `useTabStore` and `frontend/src/App.tsx` to maintain `activeProjectId` strictly in window-isolated `sessionStorage`, dispatch `X-Aico-Project: activeProjectId` on all workspace requests, and ignore global `active_project_changed` WebSocket broadcasts on desktop windows.
  - Parse `?project=<project_id>` query parameter on application boot in `App.tsx` and `useTabStore` to enable deep linking and detached window launches.

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

- **TASK-CORE-017-16**: VisualLineageCanvas Downstream-Filtered Lineage Isolation
  - **Status**: Done
  - **Addresses**: `REQ-025`
  - **Blocked by**: `TASK-CORE-017-15`
  - **What it delivers**:
    - Updates `frontend/src/features/epic-hub/VisualLineageCanvas.tsx` so that when an artifact node (Epic or Feature) at tier $C$ is selected, all nodes in upstream tiers and in the same tier (tier <= C) remain visible, while downstream tiers (tier > C) display strictly only nodes connected downstream from the selected node.
    - Isolates edge rendering to active connections linked to the selected node (upstream provenance and downstream blast-radius).
    - Enables canvas background click ("clicking nothing") and node toggle click to unselect and restore the full graph overview.
    - Updates unit tests in `frontend/tests/VisualLineageCanvas.test.tsx` to verify tier <= C retention, tier > C downstream filtering, edge isolation, and canvas background unselect.
  - **Verification Seam**: `frontend/tests/VisualLineageCanvas.test.tsx`

### Phase 10: Canvas Pan Drag vs Click Distinction & Cursor-Centric Zoom

- **TASK-CORE-017-17**: VisualLineageCanvas Pan Drag vs Click Distinction & Cursor-Centric Zoom
  - **Status**: Done
  - **Addresses**: `REQ-016`, `REQ-025`
  - **Blocked by**: `TASK-CORE-017-16`
  - **What it delivers**:
    - Updates `VisualLineageCanvas.tsx` with drag displacement threshold tracking ($\Delta > 4\text{px}$) to cleanly separate pan-dragging from stationary clicks, preserving node selection and downstream filtering throughout pan operations.
    - Implements focal-point zoom around cursor position on wheel scroll, calculating simultaneous scale and pan offsets ($pan' = x_m - (x_m - pan) \cdot (z_{next} / z_{current})$) to keep the world point under the cursor stationary.
    - Updates toolbar zoom controls to scale around viewport center.
    - Disables CSS transform transitions during active panning for immediate 1:1 pointer responsiveness.
    - Adds automated regression tests in `frontend/tests/VisualLineageCanvas.test.tsx` verifying drag pan selection preservation, stationary click deselection, and cursor-centered zoom offsets.
  - **Verification Seam**: `frontend/tests/VisualLineageCanvas.test.tsx`

### Phase 11: Unified Cascading Lineage Selection & Toolbar Streamlining

- **TASK-CORE-017-18**: VisualLineageCanvas Unified Cascading Lineage Selection & Toolbar Streamlining
  - **Status**: Done
  - **Addresses**: `REQ-025`
  - **Blocked by**: `TASK-CORE-017-17`
  - **What it delivers**:
    - Replaces disparate drilldown states in `VisualLineageCanvas.tsx` with a unified `selectedPath` (`Map<number, string>`) mapping column tier index to selected node.
    - Highlights all nodes along the active cascading selection path with `data-selected="true"`.
    - Progressively filters downstream tiers to nodes connected downstream from each selected tier.
    - Resets downstream selections when an upstream node is modified.
    - Toggles off tier and clears downstream selections on re-clicking an active node.
    - Removes `epic-focus-select` and `feature-drilldown-select` from the canvas toolbar.
    - Updates `frontend/tests/VisualLineageCanvas.test.tsx` to verify cascading multi-tier selection (`ADR` -> `Epic` -> `Feature` -> `Task` -> `Code`), upstream resets, node toggle-off, and toolbar streamlining.
  - **Verification Seam**: `frontend/tests/VisualLineageCanvas.test.tsx`

### Phase 12: Multi-Project State Isolation, Display Name Preservation & `.aico-project.json` Configurator Modal

- **TASK-CORE-017-19**: Preserve Manifest Display Name, Add Manifest Read/Update API & Support Optional Repo URLs
  - **Status**: Done
  - **Addresses**: `REQ-027`, `REQ-028`, `REQ-029`
  - **Blocked by**: `TASK-CORE-017-18`
  - **What it delivers**:
    - Updates `ProjectManifest` in `crates/aico-core/src/manifest.rs` with `display_name: Option<String>` and `url: Option<String>`, and updates `RootScanner::scan` in `crates/aico-core/src/scanner.rs` so `DiscoveredProject.name` preserves the human-readable `name` (`"AICO Specs"`) while `project_name` remains the canonical slug (`"aico"`).
    - Extends `POST /api/v1/projects` in `crates/aico-server/src/server.rs` to accept full `.aico-project.json` fields (`name` display name, optional `url`, and `repositories` list with fixed `../codebase/<id>` paths).
    - Adds `GET /api/v1/projects/manifest` and `PUT /api/v1/projects/manifest` in `crates/aico-server/src/server.rs` to read and update the active project's `.aico-project.json`, refresh `WorkspaceRegistry`, and broadcast `projects_updated`.
    - Normalizes empty/omitted repository `url` values to `None` in `crates/aico-server/src/git.rs`.
  - **Verification Seam**: `crates/aico-core/tests/scanner_test.rs` and `crates/aico-server/tests/project_api_test.rs`

- **TASK-CORE-017-20**: Fix Empty-Project `aico-specs` State Leakage & Live Dashboard Refresh on Project Switch
  - **Status**: Done
  - **Addresses**: `REQ-026`, `REQ-027`
  - **Blocked by**: `TASK-CORE-017-19`
  - **What it delivers**:
    - Updates `fetchEpics`, `fetchFeatures`, `fetchAdrs`, and `fetchCodeFiles` in `frontend/src/App.tsx` to assign empty arrays (`[]`) when a project has 0 items, and passes `discoveredEpics`, `discoveredFeatures`, `discoveredAdrs`, and `codeFiles` directly to `ShellLayout` without coercing empty arrays to `undefined`.
    - Updates `ProjectSwitcher.tsx` to `await` `POST /api/v1/projects/active` before firing `onSelectProject`, and keys `<DashboardView key={`${activeProjectId}-${workspaceRefreshKey}`} />` in `App.tsx` so switching projects immediately updates Epics, Specs, Glossary terms, ADRs, Storage stats, Lineage Graph, and Git Repositories without requiring `F5`.
    - Updates `DashboardView.tsx` so empty glossary/ADR lists from the active project render `0` items instead of falling back to hardcoded `aico-specs` defaults.
  - **Verification Seam**: `frontend/tests/ShellLayout.test.tsx`

- **TASK-CORE-017-21**: Move New Project Button Outside Dropdown & Build OmniSearch-Style `ProjectConfigModal`
  - **Status**: Done
  - **Addresses**: `REQ-027`, `REQ-028`
  - **Blocked by**: `TASK-CORE-017-19`
  - **What it delivers**:
    - Moves the New Project trigger (`data-testid="btn-new-project-trigger"`) out of the `ProjectSwitcher` dropdown menu into a standalone button next to the dropdown.
    - Creates `frontend/src/components/navigation/ProjectConfigModal.tsx` (`data-testid="project-config-modal"`) styled like `OmniSearchModal` for creating and editing `.aico-project.json`, supporting `project_name` with live `<project_name>-specs` preview, UI display `name`, optional specs `url`, and dynamic `repositories` list (`id`, `name`, `type`, optional `url`, and read-only path `../codebase/<id>`).
  - **Verification Seam**: `frontend/tests/ProjectSwitcher.test.tsx`

- **TASK-CORE-017-22**: Add Optional Remote URL State & Edit `.aico-project.json` Action to `GitRepositoriesCard`
  - **Status**: Done
  - **Addresses**: `REQ-028`, `REQ-029`
  - **Blocked by**: `TASK-CORE-017-20`, `TASK-CORE-017-21`
  - **What it delivers**:
    - Updates `frontend/src/features/dashboard/GitRepositoriesCard.tsx` to render a `"No remote URL set"` badge (`data-testid="git-no-url-<repo.id>"`) when a specs or codebase/skills repository has no remote URL configured, and disables `Clone` / `Pull` / `Push` buttons until a remote URL is set.
    - Adds an Edit Project Configuration button (`data-testid="edit-project-manifest-btn"` in the card header and `data-testid="git-edit-repo-<repo.id>"` on each repository card) that opens `ProjectConfigModal` in edit mode (`PUT /api/v1/projects/manifest`) and refreshes the Git status and project switcher upon save.
  - **Verification Seam**: `frontend/src/features/dashboard/__tests__/GitRepositoriesCard.test.tsx`

### Phase 13: Multi-Tab Project Isolation & Decoupled State Synchronization

- **TASK-CORE-017-23**: Implement Axum `ProjectContext` Extractor & Wire Project-Scoped REST Handlers
  - **Status**: Done
  - **Addresses**: `REQ-030`
  - **Blocked by**: `TASK-CORE-017-22`
  - **What it delivers**:
    - Creates an Axum extractor / helper `ProjectContext` in `crates/aico-server/src/server.rs` that resolves the project from query params (`?project=` or `?project_id=`), then `X-Aico-Project` header, with fallback to `registry.get_active()`.
    - Updates `list_epics_handler`, `get_epic_handler`, `list_features_handler`, `get_feature_handler`, `list_adrs_handler`, `get_adr_handler`, `get_glossary_handler`, `list_code_files_handler`, `get_code_file_handler`, `get_git_status_handler`, and `get_project_manifest_handler` to use the resolved `ProjectContext`.
  - **Verification Seam**: `crates/aico-server/tests/project_api_test.rs`

- **TASK-CORE-017-24**: Add `X-Aico-Project` Header to Frontend Workspace API Requests & Decouple `active_project_changed` WebSocket
  - **Status**: Done
  - **Addresses**: `REQ-031`, `REQ-032`
  - **Blocked by**: `TASK-CORE-017-23`
  - **What it delivers**:
    - Updates `fetchProjects`, `fetchEpics`, `fetchFeatures`, `fetchAdrs`, `fetchCodeFiles`, and `fetchGlossaryTerms` in `frontend/src/App.tsx`, as well as `GitRepositoriesCard.tsx` and `ProjectConfigModal.tsx`, to include `X-Aico-Project: activeProjectId` in HTTP request headers.
    - Removes desktop window forced switching on `active_project_changed` in `App.tsx` (allowing desktop tabs to retain independent active projects), while preserving targeted project sync for mobile companions with matching `targetWindowId`.
  - **Verification Seam**: `frontend/tests/ProjectSwitcher.test.tsx` and `frontend/tests/ShellLayout.test.tsx`

- **TASK-CORE-017-25**: Support URL `?project=<id>` Initialization & Fix Detached Window Launcher
  - **Status**: Done
  - **Addresses**: `REQ-033`
  - **Blocked by**: `TASK-CORE-017-24`
  - **What it delivers**:
    - Updates `App.tsx` and `useTabStore.ts` initialization logic to inspect `new URLSearchParams(window.location.search).get('project')`. If provided, it overrides `initialProjectId` and persists it to `sessionStorage`.
    - Ensures `ProjectSwitcher.tsx` "Open in new window" button (`data-testid="detach-window-${project.id}"`) opens with clean URL parameters and initializes the new window with the detached project.
  - **Verification Seam**: `frontend/tests/ProjectSwitcher.test.tsx`

- **TASK-CORE-017-26**: End-to-End Multi-Tab Project Switching Integration Verification
  - **Status**: Done
  - **Addresses**: `REQ-030`, `REQ-031`, `REQ-032`, `REQ-033`
  - **Blocked by**: `TASK-CORE-017-23`, `TASK-CORE-017-24`, `TASK-CORE-017-25`
  - **What it delivers**:
    - Comprehensive unit and integration test coverage verifying that concurrent requests with different `X-Aico-Project` headers return isolated project data without server cross-talk.
    - Frontend unit tests simulating two independent window sessions verifying project switching in Window 1 does not alter Window 2.
  - **Verification Seam**: `crates/aico-server/tests/project_api_test.rs` and `frontend/tests/ProjectSwitcher.test.tsx`
